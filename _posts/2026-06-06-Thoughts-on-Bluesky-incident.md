---
layout: post
title:  "Thoughts on Bluesky's April-2026 incident"
date: 2026-06-06
categories: ["model-checking", "incidents"]
---

Jim Calabro published a great post-mortem on Bluesky's April-2026 incident. I learned from reading this good incident report. 

The root cause was a RPC handler with unbounded concurrency. According to the report:

> This particular RPC (`GetPostRecord`) takes a batch of post URIs, and looks them all in memcached, then scylla upon cache miss. What I had missed is that we deployed a new internal service last week that sent less than three `GetPostRecord` requests per second, but it did sometimes send batches of 15-20 thousand URIs at a time. 

The report also says:

> Every RPC handler in the data plan does bounded concurrency (i.e. errgroup.SetLimit). However, this endpoint did not! It was the only endpoint in the entire system that was missing it. 

> This means that we'd launch 15-20 thousand goroutines for the request, slam the daylights out of memcached by dialing a tone of connections, then close and return them to OS since our max idle conn pool size was 1000. They would build up in the TCP_TIME_WAIT state, and exhaust all available ports. 

This all later cascaded into a series of failures; RPC handler written in Go had a lot of log writes which caused blocking system calls, and that led the Go runtime to spawn many more OS threads, that in turn burdeneed the garbage collector and since they had aggressive memory limits their system OOM'ed often (foot note what oom is). The saturation and the cascades kept the system unstable for a couple of days. 

In hindsight it's easy to fix the past sitting in the present. And that's not the purpose of this post. Neither to suggest what could have prevented it. I want to look at it from educational standpoint and ask myself, what can I learn from this to avoid similar mistakes while developing such systems in the future.

One possible learning path from this system is to first model the system and try to reproduce the problem. It's not easy to learn from failure you cannot construct. Reproducing a failure makes it observable and you can learn a lot from an observable failure as you internalise what you must avoid in that context. 

Writing a model in a model checker can also force you to reason about correctness of the system. You can check the correctness of your system by specifying invariants that must hold in all reachable states of your system. To do that, you write a model of your system and specify invariants and then let a model checker validate that your inavriants hold in all reachable states. As you develop complex systems, doing this can greatly increase team's confidence in the correctness of the system. 

It's a fair counter point that a model and system implementation can diverge, so why model? If you write the model of your system, that you've to implement, in which invariants hold, you've already done enough thinking to minimise the implemnetation errors later. Modelling the system, coming up with invariants, and then writing a model such that invariants hold in all interleavings and states require clear thinking. Moreover, as AI is getting better at code, we can write a model and then check with AI that the implementation adheres to the model before it makes it to production. This way, a model can be embedded in the development process even if it is written after the implementation or after an incident. It helps you correct your wrong assumptions for correct future implementation behavior. A model is not just a stylistic artifact but it can help create correct design and an implementation. 

From the incident report, we can extract bounded concurrency as a key correctnes property of the system in question. Port exhaustion was another problem that emerged in the incident and so was excessive logging. Let's model the system so it can reproduce unboudned concurrency and port exhaustion and then address those; excessive logging is something which can be added to it later. The model would also show that the modelled system could not recover on its own. 


We try to wrtie correctness properties of the system first and then we write a model and try to ensure using the model checker that the correctness properties pass. We've used Spin as a model checker (which uses Promela as specification language in which models are written); I find Spin particularly relevant for teams that use Go language because it shares some share similar features Spin (e.g. channels, etc), although teams could benefit from other model checkers like TLA+, P, FizzBee as well.


Let's see how the model models boudned concurrency, port exhaustion, and recovery from failure. First, let's look at high level design of the model. 

### High level design of the model 

The model has two basic components, a 'Service', and a 'URI handler'. The service produces work and service consumes it. 

The model also has three correctness propreties that must hold in all reachable states:

- Concurrency remains bounded. 
- Used ports remain within a limit so as not to trigger port exhaustion. 
- System recovers from stress. 

'Sevice' produces work and sends it to the 'URI handler' on a channel and the handler consumes it. For the system's state, the number of work items in flight, system load, number of ports used (those that are either idle, active, or stuck in 'TIME_WAIT') are tracked in variables. 

### How the Service works 

The 'Service' process a 'batch' and once the work items in flight tend to zero it processes the next badge. The batch size and work limits are intentionally kept small to avoid state space explosion. Since we're trying to reproduce the issues, we don't check whether work in flight is less than the work limit and this helps us reproduce the unbounded concurrency issue. The correctness property that checks bounded concurrency is violated when work in flight is more than the work limit and thus the issue is reproduced. (The model code linked below has many comments and among other things it points out where the problem occurs and how to fix it.) There is some more detail about it below. 

For each work item, the 'Service' tries one of three connection paths; reuse an idle connection, opens a new one if ports are available, or fails the dial if no port is available. When it fails the dial the model triggers the port exhaustion. Again, since we've specified this as a correctness property, it catches the port exhasution bug as soon as the model is in that state. 

#### How the URI handler works 

The URI handler receives work on a channel (WHAT?). After receiving it, it marks the work complete. It then models the fate of the connection. If the idle pool has room, the connection becomes reusable. If idle room is full, the connection closes and goes in a TIME_WAIT state. 

The URI handler also models an external shock event. The state of the shock reflects in the model system by setting a variable. 

The handler also models the recovery bug in such a manner that load is not shedded and the load remains (CHECK). 

### How the model models bounded concurrency

We have set a small work limit and the 'System' submits more work than the limit. URI handler does not care about a work limit and keeps processing work even if it is over the work limit. We have a system correctness property which ensures that cocurrency remains bounded and work remains with a limit. As soon as we submit more work and system has to handle more work than the limit, this correctness property is violated. And when we run the model and ask the model checker to check this property, we see the violation. We explain this more below, including a visualisation showing precisely how we're able to reproduce the problem using our model. 

### How the model models port exhasution 

The model considers those ports in use that are any of these statuses: active, idle, or in TIME_WAIT state. The model also limits ports and when ports used are more than that limit, the model triggers port exhaustion (modelling it by setting a port_exhasution boolean variable basically). When the 'Service' starts, it tries to find an idle connection. If there is no idle connection and ports in use are still less than the ports limit, it opens a new connection, and mark it as an active port. When URI handler completes work, it moves the port from active to idle if the idle pool has room; otherwise it closes the connection and put it in TIME_WAIT state. Port exhaustion occurs when new owork needs a fresh connection, but no idle connection exists and all modeled ports are unavailable.

### How the model simulates load 

Model simulates load crossing a given threshold in two ways. First, by modelling an external shock in the URI handler. Second, when ports are exhausted. There is a correctness property that looks for it and when you run the model to validate this property, it would fail because the model does not recover form it. 

### How the model checks for correctness 

In the model, we check for correctnes using LTL (Linear Temporal Logic) properties. LTL correctness properties say what must be true as a program runs over time. For example, a lock is never held by two processes or every reqquest eventually gets a reply. For one, we could say `'[] p'` which translates to `'always p holds'` or `'<> p'` which translates to `'eventually p holds'` ([] denotes always, <> denotes eventually, etc). SPIN checks all possible executions of your model and tries to find a counterexample that breaks the property. 

In our model we define three LTL properties:

- `ltl p1 { [] (work_inflight <= work_limit) }`: Always: work in flight is at or below the work limit. That is, work in flight must never exceed the work limit. That is the bounded concurrency check. 
- `ltl p2 { [] (loaded -> <> !loaded) }`: Always: if the model becomes loaded, it eventually becomes not loaded. Whenever the model becomes loaded, it must eventually become not loaded. 
- `ltl p3 { [] (!port_exhausted) }`: Always: ports are not exhausted. Taht is, ports must never be exhauseted. 

Let's see how the model reproduces the bouned concurrency bug, where work in flight exceeds the work limit. First, let's ask Spin to take our model, turn it into a model checking program and compile it:

```
$ $spin -a model.pml
$ $cc -O2 -o pan pan.c
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

At the bottom we've state of varibles and when the assertion was violated work_inflight was 4 which is more than the work limit 3, hence the violation.We could see the steps that led to this invariant's violation in the image below:

  <figure>
    <img src="{{ '/images/2026-08-24-bluesky-incident/p1_sequence_short.svg' | relative_url }}"
         alt="Sequence diagram showing work_inflight increasing to 4 and violating the P1 invariant">
    <figcaption>
      The critical interleaving: item 2 remains in flight while Service produces items 3, 4, and 5.
    </figcaption>
  </figure>

We could similarly run and check violations of other LTL properties. When I try to check the 'port exhasution' correctness property, I get an error and this trail:
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

We can see that ports used are 6, that is, active_ports plus idle_ports plus time_wait_ports equals six - which is at the port limit of 6.

If I would like to see whether the load shredding correctness property holds, when I run it I see it does not and I see this in trail that once the system is under load it does not recover from it. 

The commands to run the model with correctnes properties and to view the trail are given in comments in the model's code. In the repository, you will also find a model called model_fixed.pml [1, 2] in which the bugs are fixed and when you run correctness properties in it they hold. 

So why is the exercise useful? I think it's good for learning. I learned more about TCP so hopefully I will avoid some future mistkaes. More importantly, if a team models such incidents, they can embed this learning in the design stage of their development. Such models, when run with model checkers, could also be useful for verification purposes and help ship sytems that work. Not to say they always work and never break; sometimes they may fail but you learn from your mistakes and do more verification and such verification can help make your systems more robust, more resilient. 

If your team is writing code manually, you could check the implementation against the model to ensure that your implementation has the correctness properties as inavriants in your code. And if your team is using AI, you could use the model as a validation artifact to ensure your implementation not only implements those invariants but it is also faithful to the model. If you have a model which is correct in all reachable states and interleavings and if you validate your implementation against against this model then at least, as has been the case in my experience, it will catch many issues than it would not without the model. 


# References 

1. 