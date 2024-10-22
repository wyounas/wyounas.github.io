---
layout: post
title:  "Hello world in SPIN model checker"
date: 2024-09-25
categories: cs, model-checking
math: true
---


Spin is a model checker, a software tool for verifying computer software. To verify your software, you first write a model that describes the behaviour of your system, you then specify the correctness requirements of your system, and finally you run the model checker to check if the correctness requirements hold for the model. And if they don’t hold, then a model checker presents a counterexample. 

I stumbled upon the SPIN model checker by chance. A while ago, I finished Ben-Ari’s fantastic book on Mathematical Logic and I was browsing the list of books he has co-authored and I came across “Principles of the Spin Model Checker”. This caught my attention because in the past I scratched the surface of TLA+ and I like the idea of verifying the correctness of computer software, particularly if it’s complex. 

Instructions for installing SPIN are given <a href="https://spinroot.com/spin/Man/README.html">here</a>. PROMELA language, which uses C-like syntax, is used to write models in SPIN. 

A program in PROMELA is composed of a set of processes. You can create a process using the keyword “active proctype.” A process may or may not have parameters. You can write statements between the curly braces ({ and }) and comments between /* and */. 

You can use different data types in PROMELA, following are the details:

- **bit**, bool: Their size is 1 bit and values could be 0, 1, false, true. 
- **byte**: Size is 8 bits and values could range from 0 .. 255. 
- **short**: 16 bits and values range from -32768 .. 32767.
- **int**: 32 bits, with values ranging from $ -2^{31} $ to $ 2^{31} - 1 $.
- **unsigned**: Up to 32 bits, with values ranging from $ 0 $ to $ 2^n - 1 $.

You can also use printf statements just as you do in C language. Now let’s write a simple program that increments a global variable. 

<pre class="code">

byte total;

active proctype routine(){

    int incr = 5;
    printf("Started running \n")

    total = total + incr;

    printf("End of program. Total: %d \n", total)

}

</pre> 
Save the above in a file and save the file with *pml* file extension e.g., *hello.pml* and run it on the command prompt:

<pre class="code">
$ spin hello.pml 
Started running. 
End of program. Total 5
1 process created.
</pre>

Voila. You see the output. 

That’s all there is to a hello world program in the PROMELA. 
