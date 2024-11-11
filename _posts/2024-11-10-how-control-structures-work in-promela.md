---
layout: post
title:  "How Control Structures Work in PROMELA"
date: 2024-11-10
categories: spin, model-checking, formal-methods
---

The PROMELA language is used to write models in the SPIN model checker. PROMELA’s syntax is similar to C, but its control statements are inspired by a formalism called “guarded commands,” invented by E.W. Dijkstra, which is particularly well-suited for handling nondeterminism.

Let’s start by examining the if statement in PROMELA. It begins with the reserved word “if” and ends with “fi”. Within it, there are one or more alternatives, each starting with a double colon (::), followed by a guard and an arrow (->). Consider this example:

<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/one.png" alt="control structure"  />

In the program above, we have an “if” statement with three alternatives, where the alternative on line 6 executes. Each alternative starts with a double colon, followed by a guard (e.g. n > 0) and an arrow (->). Line 6 executes because its guard evaluates to true. 
Now, let’s explore how nondeterminism works. Consider:

<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/two.png" alt="control structure" width="500" />


In this case, when two or more guards evaluate to true, the statements associated with either guard could be executed. If you run the program multiple times, sometimes "Greater than two" is printed, and other times "Greater than three". To run the program, save the code in a file named file.pml and run it with the command “spin file.pml”. The choice is made nondeterministically.

Interestingly, this nondeterminism can be used to generate random numbers in PROMELA. In the program below, the value of "n" is determined randomly. Running the program several times will show that a different value is chosen for "n" each time.

<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/three.png" alt="control structure" width="500" />

Now another thing to consider. What happens when all alternatives are false? When all alternatives are false, the program will block. This means it cannot proceed until at least one of the guards becomes true. Consider:

<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/four.png" alt="control structure" width="500" />

The program above prints "Begin" and then blocks. In a more complex setup with multiple concurrent processes modifying the value of a global variable, a blocked process with an if-statement like the one above may eventually become unblocked. As a result, one of its guards may evaluate to true, allowing the corresponding lines to execute.

And now let’s see how looping structures work. 

One of the looping structures in PROMELA is the do-statement, which is similar to an if-statement. Let’s look at this program:

<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/five.png" alt="control structure" width="500" />

The loop continues to run until the value of 'n' reaches 10, at which point it breaks out. 
A for-statement was also introduced in version 6 of SPIN, allowing us to write a similar program using a for-statement as follows:


<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/six.png" alt="control structure" width="500" />

*Please [email](mailto:waqas.younas@gmail.com) or [tweet](https://x.com/wyounas) with questions, ideas, or corrections.*


### Note:  
To run a PROMELA program, save the code in a file with a .pml extension and run the program from the command line using `spin file.pml`.
