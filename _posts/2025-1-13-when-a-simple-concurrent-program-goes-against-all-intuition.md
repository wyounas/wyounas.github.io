---
layout: post
title:  "When the Simplest Concurrent Program Goes Against All Intuition"
date: 2025-1-13
categories: concurrency
---

I came across a fascinating and surprising aspect of a seemingly simple concurrent program when run on a model checker. Consider this:


<img loading="lazy" src="{{ site.baseurl }}/images/2025-1-13-concurrency-failing-intuition/one.png" />

If we run P and Q concurrently, with ‘n’ initialized to zero, what would the value of ‘n’ be when the two processes finish executing their statements?

I thought the final value of ‘n’ would be between 10 and 20. What do you think? Take a guess.

I came across this in Ben-Ari’s book on the SPIN model checker [1]. He said he was shocked to discover an extreme interleaving that set the final value of 'n' to 2. I was shocked as well—the outcome completely defied my intuition.

To see this in action, we could write a simple program in SPIN and claim that there is a computation where the value is 2. We can obtain this computation automatically by adding the assertion ```assertion (n > 2)``` at the end of the program and running a verification. SPIN searches the state space, looking for counterexamples.

Here is the program followed by the trail that shows the extreme interleaving:

<img loading="lazy" src="{{ site.baseurl }}/images/2025-1-13-concurrency-failing-intuition/two.png"   />

When I ran this, the error appeared:

<img loading="lazy" src="{{ site.baseurl }}/images/2025-1-13-concurrency-failing-intuition/three.png"   />

You can check out the complete interleaving trail in 
<a href="{{site.baseurl}}/files/interleavings.txt" target="_blank">this file</a>. I have never seen an interleaving this extreme.

After seeing this, I wondered: Is it possible to observe a computation in practice where 'n' is set to 2? I think it’s highly unlikely to create such a computation in practice. A Go expert shared this following code with me, which limits execution to one thread and explicitly reschedules operations. When you run this Go program, the value of 'n' is sometimes 11, sometimes 10, but never less than 10.

```go
package main

import "runtime"

func main() {
	runtime.GOMAXPROCS(1)

	n := 0
	done := make(chan bool)
	for range 2 {
		go func() {
			for range 10 {
				runtime.Gosched()
				t := n + 1
				runtime.Gosched()
				n = t
			}
			done <- true
		}()
	}
	<-done
	<-done
	println(n)
}

```

Anyway, I found it not only counterintuitive but fascinating, even though we may not encounter it in practice. So, I thought I should share it.

I wonder if it’s possible to create this computation in any other way, or a computation with the value of 'n' lower than 10? 

This one surprised me. What’s the simplest concurrent program that has surprised you the most?

### References:

1. Principles of the Spin Model Checker by Mordechai Ben-Ari.

<hr />