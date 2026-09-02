---
layout: post
title:  "Thoughts and learnings from Bluesky's April-2026 incident"
date: 2026-06-06
categories: ["model-checking", "incidents"]
---

Jim Calabro published a great post-mortem on Bluesky's April-2026 incident. I'm actually grateful for companies that operate at scale and publish their post-mortems; I always learn things from these reports that I didn't know before. 

The root cause was an RPC handler with unbounded concurrency. According to the report:

> This particular RPC (`GetPostRecord`) takes a batch of post URIs, and looks them all up in memcached, then scylla upon cache miss. What I had missed is that we deployed a new internal service last week that sent less than three `GetPostRecord` requests per second, but it did sometimes send batches of 15-20 thousand URIs at a time. 

The report also says:

> Every RPC handler in the data plane does bounded concurrency (i.e. errgroup.SetLimit). However, this endpoint did not! It was the only endpoint in the entire system that was missing it. 
> This means that we'd launch 15-20 thousand goroutines for the request, slam the daylights out of memcached by dialing a ton of connections, then close and return them to the OS since our max idle conn pool size was 1000. They would build up in the TCP_TIME_WAIT state, and exhaust all available ports. 


This later cascaded into several failures. The Go RPC handler logged many memcached errors, and those log writes used blocking system calls. The Go runtime responded by spawning many more OS threads, which increased pressure on the garbage collector. Because the service also had aggressive memory limits, it was OOM-killed repeatedly.

In hindsight it's easy to fix the past sitting in the present. And that's not the purpose of this post. Nor is my purpose to say, with hindsight, exactly what would have prevented it. I want to look at it from educational standpoint and ask myself, what can I learn from this to avoid similar mistakes while developing such systems in the future.

One possible learning path from such failures is to first model the system and try to reproduce the problem. It's not easy to learn from failure you cannot construct. Reproducing a failure makes it observable and you can learn a lot from an observable failure as you internalize what you must avoid in that context. In this post, we write a small Spin model to explore and reason about this class of failures (Spin is a model checker). 

A model can help us test the correct properties and invariants we expect the system to satisfy. Writing the model also forces us to reason about correctness of our system. By specifying correctness properties of a model, model checker checks the model and produces a counterexample when a property fails.  


From the incident report, we can extract bounded concurrency as a key correctness property of the system in question. Port exhaustion was another problem that emerged in the incident and so was excessive logging. Let's model the system so it can reproduce unbounded concurrency and port exhaustion and then address those; excessive logging is something which can be added to it later. The model would also show that the modeled system could not recover on its own. 


Let's model the system as given in the Bluesky's incident report. We'll write correctness properties of the system first and then we'll write a model and using the model checker validate that the correctness properties hold. This is not a full simulation of Bluesky's production system; it is a small teaching model of the failure pattern. We'll use Spin as a model checker; I find SPIN especially approachable for teams that write Go, because Promela has familiar ideas such as processes and channels. Although teams could benefit from other model checkers like TLA+, P, FizzBee as well.


Let's see how the model models bounded concurrency, port exhaustion, and recovery from failure. First, let's look at high level design of the model. 

### High level design of the model 

The model has two active processes: `Service` produces work, and `URI_Handler` consumes it.

The model also has three correctness properties that must hold in all reachable states:

- Concurrency remains bounded. 
- A memcached dial never fails because no idle connection is available and all modeled ports are already unavailable.
- System recovers from stress. 

`Service` produces work and sends it to the `URI_Handler` on a channel (channels are used to transfer messages between processes in Spin) and the handler consumes it. For the system's state, the number of work items in flight, system load are tracked in variables. The model tracks active ports, idle ports, and TIME_WAIT ports separately, and derives total port use from those counters.  

### How the Service works 

`Service` starts a new batch only after all in-flight work from the previous batch has completed. The batch size and work limits are intentionally kept small to avoid state space explosion. Since we're trying to reproduce the issues, we don't check whether work in flight is less than the work limit and this helps us reproduce the unbounded concurrency issue. The correctness property that checks bounded concurrency is violated when work in flight is more than the work limit and thus the issue is reproduced. (The model code linked below has many comments and among other things it points out areas where the problem occurs and how to fix it.) There is some more detail about it below. 

For each work item, `Service` reuses an idle connection, opens a new one if a port is available, or records a failed dial when no port is available.

#### How the URI_Handler works 

`URI_Handler` receives work from `work_ch` channel. Receiving an item is not the same as completing it; completion happens when the handler marks the work done and releases or closes the connection. It then models the fate of the connection. If the idle pool has room, the connection becomes reusable. If idle room is full, the connection closes and goes in a TIME_WAIT state. 

The `URI_Handler` also models an external shock event. The state of the shock reflects in the model system by setting a variable. 

The buggy model can enter `loaded` and then stay there forever, because it only sheds load below the overload threshold.

### How the model models bounded concurrency

We have set a small work limit and the `Service` submits more work than the limit. `Service` can launch more work because its start-work guard does not enforce work limits. We have a system correctness property which ensures that concurrency remains bounded and work remains with a limit. As soon as we submit more work and system has to handle more work than the limit, this correctness property is violated. And when we run the model and ask the model checker to check this property, we see the violation. We explain this more below, including a visualization showing precisely how we're able to reproduce the problem using our model. 

### How the model models port exhaustion 

The model considers those ports in use that are any of these statuses: active, idle, or in TIME_WAIT state. The model does not let total ports exceed the limit. It records port exhaustion when new work needs a connection, no idle connection exists, and every modeled port is unavailable. When the `Service` starts, it tries to find an idle connection. If there is no idle connection and ports in use are still less than the ports limit, it opens a new connection, and mark it as an active port. When `URI_Handler` completes work, it moves the port from active to idle if the idle pool has room; otherwise it closes the connection and put it in TIME_WAIT state. Port exhaustion occurs when new work needs a fresh connection, but no idle connection exists and all modeled ports are unavailable.

### How the model simulates load 

The model can become loaded through an external shock or port exhaustion. A correctness property (labeled as `p2` below) checks that any loaded state eventually becomes healthy.

### How the model checks for correctness 

In the model, we check for correctness using LTL (Linear Temporal Logic) properties. LTL correctness properties say what must be true as a program runs over time. For example, a lock is never held by two processes or every request eventually gets a reply. For one, we could say `'[] p'` which translates to `'always p holds'` or `'<> p'` which translates to `'eventually p holds'` ([] denotes always, <> denotes eventually, etc). SPIN searches the reachable executions of this finite model and reports a counterexample when a property fails.. 

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

To check recovery, run `p2`. SPIN reports an acceptance cycle: once the buggy model reaches `loaded`, it can remain loaded forever. 

The commands to run the model with correctness properties and to view the trail are given in comments in the model's code. In the repository, you will also find a model called `model_fixed.pml` [1, 2] that fixes the modeled failures by adding the concurrency guard and a recovery step; with those changes, all three checked properties pass.

So why is the exercise useful? I think it's good for learning. I learned more about TCP so hopefully I will avoid some future mistakes. More importantly, if a team models such incidents, they can embed this learning in the design stage of their development. Such models, when run with model checkers, could also be useful for verification purposes and help ship systems that work. Not to say they always work and never break; sometimes they may fail but you learn from your mistakes and do more verification and such verification can help make your systems more robust, more resilient. 

If your team is writing code manually, you could check the implementation against the model to ensure that your implementation has the correctness properties as invariants in your code. And if your team is using AI, you could use the model as a validation artifact to ensure that the implementation not only implements those invariants but is also faithful to the model. If the selected correctness properties hold for this finite model, it does not mean the implementation is bug-free. But if you validate your implementation against this model, then at least, as has been the case in my experience, it can catch many issues that would be easier to miss without the model.


# References 

1. 
