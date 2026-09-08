---
layout: post
title:  "Thoughts and learnings from Bluesky's April-2026 incident"
date: 2026-06-06
categories: ["model-checking", "incidents"]
---

Jim Calabro published a great post-mortem on Bluesky's April-2026 incident. I'm actually grateful for companies that operate at scale and publish their post-mortems; I always learn things from these reports. 

The root cause was an RPC handler with unbounded concurrency. According to the report:

> This particular RPC (`GetPostRecord`) takes a batch of post URIs, and looks them all up in memcached, then scylla upon cache miss. What I had missed is that we deployed a new internal service last week that sent less than three `GetPostRecord` requests per second, but it did sometimes send batches of 15-20 thousand URIs at a time. 

The report also says:

> Every RPC handler in the data plane does bounded concurrency (i.e. errgroup.SetLimit). However, this endpoint did not! It was the only endpoint in the entire system that was missing it. 
> This means that we'd launch 15-20 thousand goroutines for the request, slam the daylights out of memcached by dialing a ton of connections, then close and return them to the OS since our max idle conn pool size was 1000. They would build up in the TCP_TIME_WAIT state, and exhaust all available ports. 


This caused a chain of failures. The Go RPC handler logged many memcached errors, and those log writes used blocking system calls. The Go runtime then spawned many OS threads, which increased pressure on the garbage collector. The service was OOM-killed repeatedly because it also had aggressive memory limits.

In hindsight it's easy to fix the past sitting in the present. And that's not the purpose of this post. The purpose is to learn from this to avoid similar mistakes.

One possible way to learn from such failures is to first model the system and try to reproduce the problem. It's not easy to learn from failure you cannot construct. Reproducing a failure makes it observable and you can learn a lot from an observable failure as you develop a sense of what you must avoid. In this post, we write a small Spin model to explore and reason about this class of failures (Spin is a model checker). 

Why write and validate the model with a model checker? Writing the model also forces us to reason about correctness of our system. A model can help us test the correct properties we expect the system to satisfy. The model checker validates the model against correctnes properties and produces a counterexample when a property fails.  


From the incident report, we can model bounded concurrency as a key correctness property of the system. Port exhaustion was another problem that emerged in the incident and so was excessive logging. Let's model the system so it can reproduce unbounded concurrency and port exhaustion and then address those; excessive logging is something which we can add later. The model would also show that the modeled system could not recover on its own. 


This is not a full simulation of Bluesky's production system; it is a small model, for learning purposes, to model the failures. As for using Spin, I find Spin approachable for teams that write Go, because Promela has familiar ideas such as processes and channels. Although teams could benefit from other model checkers like TLA+, P, FizzBee as well, I like them all.


First, let's look at high level design of the model. 

### High level design of the model 

The model has two active processes: `Service` and `URI_Handler`. `Service` produces work, and `URI_Handler` consumes it.

The model also has three correctness properties that must hold in all reachable states:

- Concurrency remains bounded. 
- A memcached dial never fails because no idle connection is available and all modeled ports are already unavailable. (todo)
- System recovers from stress. 

`Service` produces work and sends it to the `URI_Handler` on a channel (channels are used to transfer messages between processes in Spin) and the handler consumes it. The number of work items in flight and system load are tracked in variables. The model tracks active ports, idle ports, and TIME_WAIT ports separately, and derives total port use from those counters.  

### How the Service works 

`Service` starts a new batch only after all in-flight work from the previous batch has completed. The batch size and work limits are intentionally kept small to avoid state space explosion. Since we're reproducing the failures, to reproduce the unbounded concurrency issue, we don't check whether work in-flight is less than the given work limit. 

The `Service` models its work by processing three batches of requests, in a loop. In each round, it could use one of three paths:
  - If idle ports are available, it uses an idle port and produces a work item.
  - If there is no idle port but there is an available port, it uses it and produces a work item. 
  - If there is neither an idle port, nor any port available, it marks ports as exhausted. 

Here is a rough code sketch, with mostly pseudocode mixed with some Spin/Promela concepts. One important Promela construct used in the model in a loop is a double colon, `::`. 

The`::` are option sequences in a do loop in Promela. The first statement in option sequence is called a guard. An option sequence is executed if its guard is true. If more than one guard is true at the same time, then one is selcted non-deterministically:


  ```text
  Service() {
    do
    :: all rounds are done ->
         break

    :: current batch is done and no work is in flight ->
         start the next round

    :: current batch still has work ->
         if
         :: idle connection exists ->
              reuse it and start work

         :: no idle connection exists and a port is free ->
              open a new connection and start work

         :: no idle connection exists and no port is free ->
              port_exhausted = true
         fi

         assert(ports_used <= port_limit)
    od
  }
```

The `Service` process processes two rounds of request batches. Each batch has a few work items. These numbers are intentionally small; the real system had much larger batches, but small numbers make the counterexample easier to read. 

For each work item, `Service` reuses an idle connection, opens a new one if a port is available, or records a failed dial when no port is available (by setting `port_exhausted = true`).

The correctness property that checks bounded concurrency is violated when work in flight is more than the work limit and thus the issue is reproduced. (The model code linked below has many comments and among other things it points out the lines where the problem occur; there is also another mode linked below which fixes these issues.) What violates this property is the fact that we don't check whether work in flight is less than the given work limit when we decide the current batch still has work and we've to process it. There is some more detail about it below. 



#### How the URI_Handler works 

`URI_Handler` receives work from a channel (called `work_ch`). To simulate work completion, completion happens when the handler marks the work done and releases or closes the connection. After that, the handler decides the fate of the connection. If the idle pool has room, the connection becomes reusable. If idle room is full, the connection closes and goes in a TIME_WAIT state. 

The `URI_Handler` also models an external shock event. The state of the shock reflects in the model by maarking a variable. 

Here is the rough shape of the `URI_Handler`:

```
URI_Handler() {
    do
    :: work item is received from work_ch ->
         finish that work item,
         record that it is no longer in flight,
         simulate releasing the active memcached connection used by it;

         if
         :: idle pool has room ->
              keep that memcached connection open for reuse;

         :: idle pool is full ->
              close that memcached connection into TIME_WAIT;
         fi

         assert(ports_used <= port_limit)

    :: shock has not happened yet ->
         shocked = true;
         load = thresh;

    :: some port is in TIME_WAIT ->
         let one TIME_WAIT port expire;

    :: load is greater than zero and service is not loaded ->
         shed some load;
    od
  }
```

  The last branch is the bug in the model. It only sheds load when the service is below the loaded threshold. Once the model becomes `loaded`, this branch cannot run, so the service can stay loaded forever. We have a liveness property, called `p2`, that will catch this (more on this in a bit).

  The benefit of `::` and non-determinism is that Spin tries the different things that could happen next in the model: `Service` starting more work, `URI_Handler` finishing work, TIME_WAIT ports expiring, or load being shed. This helps us find bugs, such as `Service` starting more work before the handler has freed enough resources. 

### How the model models bounded concurrency

We have set a small work limit and the `Service` submits more work than the limit. `Service` can launch more work because its start-work guard does not enforce work limits. We have a system correctness property which ensures that concurrency remains bounded and work remains with a limit. As soon as we submit more work and system has to handle more work than the limit, this correctness property is violated. And when we run the model and ask the model checker to check this property, we see the violation. We explain this more below, including a visualization showing precisely how we're able to reproduce the problem using our model. 

### How the model models port exhaustion 

The model considers those ports in use that are any of these statuses: active, idle, or in TIME_WAIT state. The model does not let total ports exceed the limit. It records port exhaustion when new work needs a connection, no idle connection exists, and every modeled port is unavailable. When the `Service` starts, it tries to find an idle connection. If there is no idle connection and ports in use are still less than the ports limit, it opens a new connection, and mark it as an active port. When `URI_Handler` completes work, it moves the port from active to idle if the idle pool has room; otherwise it closes the connection and put it in TIME_WAIT state. Port exhaustion occurs when new work needs a fresh connection, but no idle connection exists and all modeled ports are unavailable.

### How the model simulates load 

The model can become loaded through an external shock or port exhaustion. A correctness property (labeled as `p2` below) checks that any loaded state eventually becomes healthy.

### How the model checks for correctness 

In the model, we check for correctness using LTL (Linear Temporal Logic) properties. LTL correctness properties say what must be true as a program runs over time. For example, a lock is never held by two processes or every request eventually gets a reply. For one, we could say `'[] p'` which translates to `'always p holds'` or `'<> p'` which translates to `'eventually p holds'` ([] denotes always, <> denotes eventually, etc). Spin searches the reachable executions of this finite model and reports a counterexample when a property fails. 

In our model we define three LTL properties:

- `ltl p1 { [] (work_inflight <= work_limit) }`: Always: work in flight is at or below the work limit. That is, work in flight must never exceed the work limit. That is the bounded concurrency check. 
- `ltl p2 { [] (loaded -> <> !loaded) }`: Always: if the model becomes loaded, it eventually becomes not loaded. Whenever the model becomes loaded, it must eventually become not loaded. 
- `ltl p3 { [] (!port_exhausted) }`: Always: ports are not exhausted. That is, ports must never be exhausted. 

Let's see how the model reproduces the bounded concurrency bug, where work in flight exceeds the work limit. First, let's ask Spin to take our model, turn it into a model checking program and compile it:

```
$ spin -a model.pml
$ cc -O2 -o pan pan.c
```
We then try to run it so it finds a violation of LTL property `p1` (which checks for bounded concurrency):
```
$ ./pan -a -N p1
```

And it finds a violation, following is the truncated output:
```
spin: trail ends after 91 steps
#processes: 2
                queue 1 (work_ch): [3][4][5]
                work_inflight = 4
                load = 5
                shocked = 1
                active_ports = 4
                idle_ports = 0
                time_wait_ports = 0
                port_exhausted = 0
```

 As we see above, in the final state of the counterexample, `work_inflight` is 4 while the limit is 3, so `p1` is false. We could see the steps that led to this invariant's violation in the image below:

  <figure>
    <img src="{{ '/images/2026-08-24-bluesky-incident/p1_sequence_short.svg' | relative_url }}"
         alt="Sequence diagram showing work_inflight increasing to 4 and violating the P1 invariant">
    <figcaption>
      The critical interleaving: item 2 remains in flight while Service produces items 3, 4, and 5.
    </figcaption>
  </figure>

We could similarly run and check violations of other LTL properties. Checking `p3` also produces a counterexample: the replay ends with `port_exhausted` set to true.
```
spin: trail ends after 87 steps
#processes: 2
                queue 1 (work_ch): [1][2][3][4]
                work_inflight = 4
                load = 5
                shocked = 1
                active_ports = 4
                idle_ports = 0
                time_wait_ports = 2
                port_exhausted = 1
```

At that point all 6 modeled ports are unavailable: 4 active, 0 idle, and 2 in TIME_WAIT. Since no idle connection exists, the next dial fails.

To check recovery, run `p2`. Spin reports an acceptance cycle: once the buggy model reaches `loaded`, it can remain loaded forever. 

The commands to run the model with correctness properties and to view the trail are given in comments in the model's code. In the repository, you will also find a model called `model_fixed.pml` [1, 2] that fixes the modeled failures by adding the concurrency guard and a recovery step; with those changes, all three checked properties pass.

So why is the exercise useful? I learned from it. For one, I learned more about TCP so hopefully I will avoid some future mistakes. More importantly, if a team models such incidents, they can embed this learning in the design stage of their development. Such models, when run with model checkers, could also be useful for verification purposes and help ship systems that work. Not to say they always work and never break; sometimes they may fail but you learn from your mistakes and do more verification and such verification can help make your systems more robust, more resilient. 

If your team is writing code manually, you could check the implementation against the model to ensure that your implementation has the correctness properties as invariants in your code. And if your team is using AI, you could use the model as a validation artifact to ensure that the implementation not only implements those invariants but is also faithful to the model. If the selected correctness properties hold for this finite model, it does not mean the implementation is bug-free. But if you validate your implementation against this model, then at least, as has been the case in my experience, it can catch many issues that would be easier to miss without the model.


# References 

1. 
