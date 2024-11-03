---
layout: post
title:  "Visualizing State Space of Concurrent Programs"
date: 2024-10-06
categories: cs, model-checking
math: true
---

Concurrent programming is hard.

It can be challenging to mentally enumerate all the possible states that concurrent code might pass through. Visualizing how concurrent programs operate not only helps manage their complexity but also helps ensure their correctness. Recently, I’ve been exploring model checking and found these visualizations quite intriguing. So, I decided to write about them—to deepen my understanding and share what I’ve learned.

A few questions come up immediately: What is a state? And what is a state space? Let’s tackle the first question first, and then we’ll return to the second one.

A state of a program is defined by the values of its variables and the location counters (which indicate what comes next in program execution). Let’s walk through an example using a C-like language. We’ll illustrate how states are transitioned during a sequential (not concurrent) run of the following program. (We’ll explore a concurrent program right after this example.)

 <img loading="lazy" src="{{ site.baseurl }}/images/statespace/one.png" />

 The above program has three instructions, and for explanation purposes, I’ve added labels as location counters on the far left (i.e., _1_, _2_, _3_, and _end_). The program contains a single variable, **n**, declared as a byte.

The state of the program can be represented as a tuple consisting of the variable n and the location counter: **(n, location counter)**. For example, when the program starts at the first line, its state is **(0, 1)**, meaning **n** is 0 at location counter 1 (assuming the default value is 0 in this context ).

We can visualize this state as follows: (the orange arrow indicates the location counter, and at the bottom, the current value of n is displayed for clarity):

 <img loading="lazy" src="{{ site.baseurl }}/images/statespace/three.png" />

 The next state would be **(0, 2)**, indicating that **n** is 0 and the location counter has incremented to 2 after executing the statement at location counter 1. So, we have this:

 <img loading="lazy" src="{{ site.baseurl }}/images/statespace/two.png" />

 The next state would be **(1, 3)**. Here, the value of **n** has changed to 1 because, in the previous state, the location counter was 2, and executing the statement at that location incremented **n** to 1. At this point, the situation looks like this:

  <img loading="lazy" src="{{ site.baseurl }}/images/statespace/five.png" />

  Finally, the location counter increments once more to reach the end and execution finishes. So the value of **n** at the end is 1.

  
  <img loading="lazy" src="{{ site.baseurl }}/images/statespace/six.png" />

  A given sequence of states, starting from an initial state and continuing as statements execute, is called a computation. For example, in the program above, the following is a computation:

<img loading="lazy" src="{{ site.baseurl }}/images/statespace/seven.png" />

The state space of a program is simply the set of all possible states that can possibly occur during a computation. Ben-Ari offers a clear definition of state space in [2]: "The state space of a program is a directed graph; each reachable state is a node, and there is an edge from state s1 to state s2 if a transition enabled in s1 moves the computation to s2."  In model checking, the state space is generated to verify correctness specifications. 

Now, let’s explore what the state space looks like for a concurrent program. We’ll take two procedures, run them concurrently, and observe how their state space is generated. We have a global variable called **n** and two procedures, P and Q, each containing the following statements:



<img loading="lazy" src="{{ site.baseurl }}/images/statespace/eight.png" />

These are two simple procedures, quite similar in their operation, each with its respective location counters.

Since we’re discussing concurrent code, let’s take a moment to discuss interleaving.

In concurrent programs, interleaving refers to the unpredictable order in which program statements from different processes are executed. Statements are chosen nondeterministically—meaning the sequence of execution isn’t fixed and can vary.

During interleaving, a statement from P could execute first, followed by a statement from Q, and so forth. Imagine two cooks, A and B, preparing a meal in the same pot. They can’t stir simultaneously; they take turns. Cook A stirs, leaves the kitchen, then Cook B enters to stir, leaves, and Cook A returns for another turn. Alternatively, Cook A might stir twice before B even starts. The sequence of their turns is unpredictable, and the outcome of the meal depends on how their actions interleave.

This nondeterministic execution order is what makes concurrent programming tricky, as it can lead to unexpected results if not handled carefully.

So, how can we represent a state for the program mentioned above?

We can represent the state as a triple, consisting of the value of the variable n and the location counters of P and Q: **(n, location counter of P, location counter of Q)**.

For example, the initial state could be **(0, 1, 2)**, where the value of **n** is 0, and the location counters are at the first statements of both P and Q. At this point, it would look like this (with the orange arrows indicating the location counters, and the value of **n** shown at the bottom):


<img loading="lazy" src="{{ site.baseurl }}/images/statespace/eleven.png" />


As part of one possible interleaving, we can transition to the next possible state by moving to the next location counter in P, resulting in **(1, end, 2)**. Now, **n** becomes 1, which is the result of the evaluated expression when the statement at location counter 1 was executed. So, what we have is this:

<img loading="lazy" src="{{ site.baseurl }}/images/statespace/ten.png" />

Another possible next state from the initial state could be **(2, 1, end)**, and it might look like this:

<img loading="lazy" src="{{ site.baseurl }}/images/statespace/nine.png" />

This time, it's a bit different, as we incremented the location counter of Q, which reached its end, while P remained at location counter 1, and the value of **n** is 2.

So, the transition from the initial state **(0, 1, 2)** to these states could be represented like this (left arrow points to the next state after an increment in the location counter of P whereas right arrow points to the next state after an increment in the location counter of Q) :

<img loading="lazy" src="{{ site.baseurl }}/images/statespace/twelve.png" />


Next, we’ll write the program in PROMELA (a C-like language that SPIN uses) and then visualize its state space of the program.


<img loading="lazy" src="{{ site.baseurl }}/images/statespace/thirteen.png" />


The code is mostly C-like. The keyword proctype is used to declare a process, while the keyword active activates a process, ensuring that processes P and Q run concurrently. 

Again, we represent the state as a triple: **(n, location counter of P, location counter of Q)**. In the image above, the line numbers on the far left will serve as location counters.

Below, you can see the state space of the program:

<img loading="lazy" src="{{ site.baseurl }}/images/statespace/fourteen.png" />

The left arrows indicate that to get the next state we incremented the location counter in P, while the right arrows indicate that to get the next state we incremented the location counter in Q. What this state space implies is at the end of the program, depending on the interleaving, the value **n** could be either 1 or 2. 

Looking at the initial state at the top of the state space. The first element is the value of **n**, displayed as 0.  The second element in the triple is the location counter at line 5 in P. The last element shows the location counter at line 9 in Q. We can visualize this initial state as:

<img loading="lazy" src="{{ site.baseurl }}/images/statespace/fifteen_1.png" />


From this initial state, there are two possible next states—one to the left and one to the right. We’ll follow the right-side path.

Starting from the initial state **(0, 5, 9)**, if we increment the location counter in Q, moving it to line 10, the statement at line 9 executes, setting **n** to 2. This brings us to the next state, **(2, 5, 10)**.

<img loading="lazy" src="{{ site.baseurl }}/images/statespace/fifteen.png" />

From this state, we transition to the next state by incrementing the location counter in P, resulting in **(1, 6, 10)**. At this point, the value of n becomes 1 as the statement at location counter 5 executes, marking the end of this computation (and also the end of the right-hand side path since there are no more location counters to follow).

<img loading="lazy" src="{{ site.baseurl }}/images/statespace/seventeen.png" />


What happens if we modify P and Q by adding more statements? Does the state space expand? Or does it shrink? Let’s find out.

Let’s say we add a couple more lines to both P and Q—introducing a local variable and incrementing the global variable total by this local variable. These changes are not significant, but let’s see how they affect the updated code and state space (to generate the visualization I used jSpin [3], which I believe can prove to be a good tool for learning and teaching concurrency) :

<img loading="lazy" src="{{ site.baseurl }}/images/statespace/many.png" />

The state space has expanded dramatically compared to the earlier example.

Now, imagine a concurrent program with dozens of processes, each executing thousands of lines of code with numerous variables. The state space would grow exponentially. (Even small programs, if not carefully managed, can generate massive state spaces, but we’ll explore that another day.)

Out of curiosity, to further examine the impact of thread count on state space size, I modified the above program to run five concurrent processes instead of two. As a result, the state space grew so large that, in the final visualization, each state appeared as a tiny dot, rendering the entire image unreadable.


<img loading="lazy" src="{{ site.baseurl }}/images/statespace/large.png" />


This illustrates how complex it is to debug and test concurrent programs. You may have to tackle countless states and scenarios.

Now, you might be wondering: Yes, the state space looks cool, but what is it actually useful for? 

Well, to increase our confidence in a program’s correctness, we can define certain correctness properties that must hold during execution. Model checkers like SPIN can ensure that these properties are maintained across the entire state space. For example, we can define a property such as "X should always be true," meaning X must not be false in any state.
This is what makes model checkers so powerful—they allow you to verify the correctness of your programs across a wide range of scenarios that traditional testing methods, like unit tests or integration tests, might overlook.

*Thanks to Gerard Hozlmann for reading the draft of this. Please [email](mailto:waqas.younas@gmail.com) or [tweet](https://x.com/wyounas) with questions, ideas, or corrections.*

### References:

1. https://spinroot.com. And the overview paper The Model Checker Spin,   IEEE Trans. on Software Engineering Vol. 23, No. 5, May 1997, pp. 279-295.
2. Mathematical Logic for Computer Science, 3rd edition, by Mordechai Ben-Ari.
3. For state space visualization, I used https://github.com/motib/jspin. 	

