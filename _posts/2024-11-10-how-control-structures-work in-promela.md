---
layout: post
title:  "How Control Structures Work in PROMELA (SPIN)"
date: 2024-11-10
categories: spin, model-checking, formal-methods
---

The PROMELA language is used to write models in the SPIN model checker. PROMELA’s syntax is similar to C, but its control statements are inspired by a formalism called "guarded commands," invented by E.W. Dijkstra, which is particularly well-suited for handling nondeterminism.

Let’s start by examining the `if` statement in PROMELA. It begins with the reserved word "if" and ends with "fi." Within it, there are one or more execution sequences, each starting with a double colon (::), followed by a guard and an arrow (->). Consider this example:

<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/one.png" alt="control structure"  />

In the program above, we have an "if" statement with three execution sequences (at lines 6, 8, and 10). Each statement after the double colon and before the arrow is a guard; for example, at line 6 the guard is "n > 0," at line 8 it is "n < 0," and at line 10 the guard is "n == 0." An execution sequence runs when its guard evaluates to true. Here, the guard at line 6 evaluates to true, so its execution sequence executes.

If more than one guard evaluates to true, one of the sequences is selected nondeterministically. If no guard evaluates to true, the process remains blocked until at least one guard becomes selectable.

Now, let’s first explore how nondeterminism works. Consider:


<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/two.png" alt="control structure" width="500" />


In this case, when two or more guards evaluate to true, the statements associated with either guard could be executed. Running the program multiple times will sometimes print "Greater than two" and other times "Greater than three." To try this out, save the code in a file named file.pml and run it with the command "spin file.pml". The choice is made nondeterministically.

Interestingly, this nondeterminism can be used to generate random numbers in PROMELA. In the program below, the value of "n" is determined randomly. Running the program multiple times will show that a different value is chosen for "n" each time.

<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/three.png" alt="control structure" width="500" />

Now let’s see what happens when all guards evaluate to false. When all guards evaluate to false, the process will block. This means it cannot proceed until at least one of the guards becomes true. Consider:

<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/four.png" alt="control structure" width="500" />

The program above prints "Begin" and then blocks. In a more complex setup with multiple concurrent processes modifying the value of a global variable, a blocked process with an if-statement like the one above may eventually become unblocked if one of the guards evaluates to true.

Now, let’s take a look at looping structures in PROMELA. One of these is the do-statement, which functions similarly to an if-statement. Consider this program:

<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/five.png" alt="control structure" width="500" />

The loop continues to run until the value of 'n' reaches 10, at which point it breaks out. The guard selection rules work the same way as they do in an if-statement. If multiple guards in a loop evaluate to true, SPIN will select one nondeterministically. If all guards evaluate to false, the execution process will block.

The keyword "else" can be used as a guard in a selection or repetition structure, and it defines a condition that is true only if all other guards evaluate to false within the same structure. Let’s look at how it can be used:


<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/else.png" alt="control structure" width="500" />

A for-statement was also introduced in version 6 of SPIN, allowing us to write a similar program using a for-statement as follows:

<img loading="lazy" src="{{ site.baseurl }}/images/2024-11-10-control-structures-promela/six.png" alt="control structure" width="500" />


*Please [email](mailto:waqas.younas@gmail.com) or [tweet](https://x.com/wyounas) with questions, ideas, or corrections.*


### Note:  
To run a PROMELA program, save the code in a file with a .pml extension and run the program from the command line using `spin file.pml`.
