---
layout: post
title:  "Thoughts on Bluesky's April-2026 incident"
date: 2026-06-06
categories: ["model-checking", "incidents"]
---

Jim Calabro published a great post-mortem on Bluesky's April-2026 incident. I learned from this good incident report. 

The root cause was a RPC handler with unbounded concurrency. The report says:

> Every RPC handler in the data plan does bounded concurrency (i.e. errgroup.SetLimit). However, this endpoint did not! It was the only endpoint in the entire system that was missing it. 

This caused saturation as system ran out of resources. And it later cascaded into a series of failures; RPC handler written in Go had a lot of log writes which caused blocking system calls, and that led the Go runtime to spawn many more OS threads, which burdeneed the garbage collector and since they had aggressive memory limits their system OOM'ed often (foot note what oom is). The saturation plus the cascades turned metastable and kept the system unstable for a couple of days. 

In hindsight things look easy and it's easy to fix the past sitting in the present. But I want to look at it from educational standpoint and ask myself, what can I learn from this? What can I learn to better reason while developing such systems in the future? So my goal is not to suggest what could have prevented this incident but to try to learn from it to reason better about such systems and issues in future. 

I have worked on systems that caused saturation and cascaded failures. These failures humble the engineers and you almost always tend to ask how could I prevent it from happening again? One practical way which has proven effective, for teams I have been part of, when reasoning about designs and correctness is to build a model of the system being designed using a model checker. We try to wrtie correctness properties of the system first and then we write a model and try to ensure using the model checker that the correctness properties pass and not fail in all reachable system states. We've used Spin as a model checker (which uses Promela as specification language in which models are written); I find it particularly relevant when teams use Go because Spin/Promela share similar features (e.g. buffered channels, unbuffered channels, etc), although teams could benefit from other model checkers like TLA+ as well.

One can argue that during the implementation, after having created such models, bugs come up implementation so what's the point of writing a model? The reason why creating a model is helpful because it helps in structuring your thinking. You could show this structure to your colleagues and get feedback to improve your ideas. You could reflect on this structure with regards to implementation and could come up with a roadmap and potential issues you might come across. Also, and when you give structure to your thinking you could still leave some gaps but could also uncover many gaps as well; you uncover more than you leave in my experience. A newly constructed house or a long importnat bridge may have some shortcomings but those shortcomings should not be the reason to stop making maps, because for the purpose they serve their benefits outweigh other costs.

Reading the Bluesky's incident report, it's the unbounded concurrency in one of the new RPC handlers that triggered the incident. The RPC handler would launch 15-20 thousand goroutines for each request, and each would then do a chain of events then would make the system stable e.g. by exhuasting all availble ports, spawning more OS threads than the usual, leading to extra burden on the garbage collector, eventually leading to out of memory (OOM) errors, what made it worse was to not being able to recover from OOMs because existing connections were stuck even after restarts. The system got into a metastable state and could not recover from it. 

The incident report also mentioned that every RPC handler had bounded concurrency configured except this new RPC handler. Couple this with the fact that the bounded concurrency is the root cause, it emerges as a core correctness property of RPC handlers, a property which we must enforce in all reachable system states. 

The second correctness property that emerges is the ability of a system to be able to recover from an overloaded state. The system should be able to always recover from an overloaded state to an eventual stable state. 

So we can write a model which can abstract away the Bluesky scenario and ensure that the following two correctness properties hold in every reachable state of the model:

a. The system should guard against unboudned concurrency. The available amount of work should be less than a threshold. 
b. Always, if a system goes into an overloaded state, it should eventual get to a stable state. 


In the model, first, we can write a "Service" which produces work and also write a "URI-Handler" that processes that work. 

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