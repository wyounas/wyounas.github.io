---
layout: post
title:  "Thoughts on Bluesky's April-2026 incident"
date: 2026-06-06
categories: ["model-checking", "incidents"]
---

Jim Calabro published a great post-mortem on Bluesky's April-2026 incident. I learned from this good incident report. 

The root cause was a RPC handler with unbounded concurrency. The RPC handler, according to the report:

> This particular RPC (`GetPostRecord`) takes a batch of post URIs, and looks them all in memcached, then scylla upon cache miss. What I had missed is that we deployed a new internal service last week that sent less than three `GetPostRecord` requests per second, but it did sometimes send batches of 15-20 thousand URIs at a time. 

The report also says:

> Every RPC handler in the data plan does bounded concurrency (i.e. errgroup.SetLimit). However, this endpoint did not! It was the only endpoint in the entire system that was missing it. 

> This means that we'd launch 15-20 thousand goroutines for the request, slam the daylights out of memcached by dialing a tone of connections, then close and return them to OS since our max idle conn pool size was 1000. They would build up in the TCP_TIME_WAIT state, and exhaust all available ports. 

This all later cascaded into a series of failures; RPC handler written in Go had a lot of log writes which caused blocking system calls, and that led the Go runtime to spawn many more OS threads, that in turn burdeneed the garbage collector and since they had aggressive memory limits their system OOM'ed often (foot note what oom is). The saturation and the cascades kept the system unstable for a couple of days. 

In hindsight it's easy to fix the past sitting in the present. And that's not the purpose of this post. Neither to suggest what could have prevented it. I want to look at it from educational standpoint and ask myself, what can I learn from this to better reason while developing such systems in the future.

One possible learning path from this system is to first model the system and try to reproduce the problem. It's not easy to learn from failure you cannot construct. Reproducing a failure makes it observable and you can learn a lot from an observable failure as you internalise what you must avoid in that context. 

Writing a model in a model checker can also force you to reason about correctness properties of the system. A model can also check the correctness of the system in all possible system states and this can increase the confidence of the team. It's a fair counter point that a model and system implementation can diverge. As AI is getting better at code, we can write a model and then check with AI that the implementation adheres to the model before it makes it to production. This way, a model can be embedded in the development process even if it is written after the implementation or after an incident. It helps you correct your wrong assumptions for correct future implementation behavior. A model is not an artifact that possess information about danger but also it helps design and create a system capable of responding to that danger. 

From the incident report, we can extract bounded concurrency as a key correctnes property of the system in question. Port exhaustion was another problem that emerged in the incident and so was excessive logging. Let's model the system in which can reproduce unboudned concurrency and port exhaustion, excessive logging is something which can be added to it later. The model would also show that the modelled system could not recover on its own. 


We try to wrtie correctness properties of the system first and then we write a model and try to ensure using the model checker that the correctness properties pass. We've used Spin as a model checker (which uses Promela as specification language in which models are written); I find it particularly relevant when teams use Go because Spin/Promela share similar features (e.g. channels, etc), although teams could benefit from other model checkers like TLA+ as well.


Let's see how the model models boudned concurrency, port exhaustion, and recovery from failure. First, let's look at high level design of the model. 

### High level design of the model 

The model has two basic components, a 'Service', and a 'URI handler'. The service produces work and service consumes it. 

The model also has three correctness propreties that must hold in all reachable states:

- Concurrency remains bounded. 
- Used ports remain within a limit so as not to trigger port exhaustion. 
- System recovers from stress. 

'Sevice' produces work and sends it to the 'URI handler' on a channel and the handler consuems it. The number of work items in flight, system load, number of ports used (those that are either idle, active, or stuck in 'TIME_WAIT') are tracked in variables. 

#### How the Service works 

The 'Service' process a 'batch' and once the work items in flight tend to zero it processes the next badge. The batch size and work limits are intentionally kept small to avoid state space explosion. Since we're trying to reproduce the issues, we don't check whether work in flight is less than the work limit and this helps us reproduce the unbounded concurrency issue. The correctness property that checks bounded concurrency is violated when work in flight is more than the work limit and thus the issue is reproduced.  

For each work item, the 'Service' tries one of three connection paths; reuse an idle connection, opens a new one if ports are available, or fails the dial if no port is available (). When it fails the dial the model triggers the port exhaustion. Again, since we've specified this as a correctness property, it catches the port exhasution bug as soon as the model is in that state. 

#### How the URI handler works 

The URI handler receives work (on a channel). After receiving it, it marks the work complete. It then models the fate of the connection. If the idle pool has room, the connection becomes reusable. If idle room is full, the connection closes and goes in a TIME_WAIT state. 

The URI handler also models an external shock event. The state of the shock reflects in the model system by setting a variable. 

The handler also models the recovery bug in such a manner that load is not shedded and the load remains (CHECK). 

### How the model models bounded concurrency

We have set a small work limit and the System submits more work than the limit. URI handler does not care about a work limit and keeps processing work even if it is over the work limit. We have a system correctness property defined which ensures that cocurrency remains bounded and work remains with a limit. As soon as we submit more work and system has to handle more work than the limit, this correctness property is violated. And when we run the model and ask the model checker to check this property, we see the violation. 

### How the model models port exhasution 



Let's assume that the work of the "Service" is just to produce work so that's simple. 

The "URI-Handler" receives work and consume it. It also once receives a "shock" and in that case its loaded is increased up to its threshold. 

The model also specifies two correctness properties as:

- [] { work_inglifht <= work_limit }
- [] {loaded -> <> !loaded}

The first ensures that work inflight should always be less than a given limit. Second ensures that always, if the system is loaded then eventually it sheds it load. The symbol `[]` denotes always, `->` an implication, `<>` means eventually, and `!` represents a not in Spin/Promela. 

Here is the code of this simplified model. 

```promela 
```





 address the objection of model vs implmemnentation

While reading the report, and in particular the above quoted excerpt, my observation is that there is a underlying system correctness property (at least for RPC handlers) and that is, we ensure that concurrency remains bounded. 