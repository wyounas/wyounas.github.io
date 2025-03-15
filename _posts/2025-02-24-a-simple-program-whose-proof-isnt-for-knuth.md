---
layout: post
title:  "A simple program by Knuth whose proof isn't"
date: 2025-2-24
categories: formal-methods
math: true
---

I accidentlly stumbled upon this book "Beauty is our business, A birthday Salute to Edsger W. Dijkstra" edited by W.H.J Feijen et al. This book
is a tribute by his friends and colleagues who were influenced by Dijkstra's work. 

In the preface in another paragraph down below the previous one, it says, "In the end, on 
Dijkstra's fifty-ninth birthday, we asked fifty-odd friends and colleages of his to contribute to this birthday salute. We requested 
mainly technical contributions, and we asked that they be relatively short. Fifty-odd authors responded, and we are proud to place their contributions before Edsger Dijistra." 

The preface also shares a quote by Dijkstra that inspired the book's title. That Dijkstra's quote is 
worth sharing: "...when we recognize the battle against chaos, mess, and unmastered complexity as one of computing science's major 
callings, we must that 'Beauty is our business.'"

So after buying the book, I was skimming through table of contents and I came across an contribution by Donald E. Knuth titled "A simple program  whose proof isn't." I thought
it would be fun and it sure was.

In this contribution, Knuth shares two programs which he had to write while he was working on TeX (a typesetting program). 

The first program converts decimal fraction to fixed-point binary and he shares that
he finds it easy to proof this is correct (he calls this program P1 in the paper). 

The second program converts the other way, given an 
integer _n_, it finds a decimal fraction and it's the correctness of this program which isn't straightforward (he calls this program P2). 

Let's talk about these one by one. 

## Convert decimal fraction to fixed-point binary

TeX works with integer multiples of 2^-16 so Knuth had
to write a routine to convert 0.d1d2...dk to nearest possible binary fraction. 

It's fascinating how Knuth developed this program. Here is some detail as given in the paper. 

Since ee need to find the nearest integer multiple of 2^-16, so Knuth says we can round the quantity: 

$$
2^{16} \sum_{j=1}^k \frac{d_j}{10^j}
$$

to the nearest integer _n_. If two integers are equally near, we let _n_ be the integer:

$$
n = \left\lfloor 2^{16} \sum_{j=1}^k \frac{d_j}{10^j} + \frac{1}{2} \right\rfloor
$$

To see how above may work, let's say we have the input `0.12408`, then:

$$
n = 2^{16} \cdot \left( \frac{1}{10^1} + \frac{2}{10^2} + \frac{4}{10^3} + \frac{0}{10^4} + \frac{8}{10^5} \right) + \frac{1}{2}
$$

This will give us:

$$
n = 8132
$$

Knuth shares two important considerations at this point. 

First, since input values _dj_ are small nonnegative integers, Knuth wanted to keep the intermediate results reasonably small.

Second, since _k_ can be arbitrarily large, Knuth had concerns that these could become too large for our computers' hardware. He 
demonstrated that 17 digits are not only sufficient — they are sometimes necessary to determine the correct value of _n_. 
We can safely ignore _dj_ for _j > 17_. This to me highlights that Knuth’s attention to detail here is brilliant. 

Here is this program called P1, as outlined in the paper:

```
P1: l := min(k, 17); m := 0;

repeat m := (131072 * d[l] + m) div 10;
       l := l - 1;
until l = 0;

n := (m + 1) div 2
```

Following shows a dry run of program P1 on input `0.12408`. The digits of the input `0.12408` are processed in reverse order 
from d[5] to d[1]: 

| Iteration | Current 'l' | d[l] | Updated 'm' | Updated 'l' |
|---|---|---|---|---|---|
| 1 | 5 | 8 | 104857 | 4 |
| 2 | 4 | 0 | 10485 | 3 |
| 3 | 3 | 4 | 53477 | 2 |
| 4 | 2 | 2 | 31562 | 1 |
| 5 | 1 | 1 | 16263 | 0 |

---

So the loop exits when _l_ is zero but there is one more thing still left for P1 to do. To complete the dry run we take the 
value of _m_ from the last row, when _l_ is zero, from the above table and use it to 
get _n_:

$$
n = \frac{m + 1}{2} = \frac{16263 + 1}{2} = \frac{16264}{2} = 8132
$$

Thus, converting `0.12408` via P1 produces:

$$
n = 8132
$$


Knuth shares that the proof is easy for this one. "We have m = m1 at the beginnning of the repeat loop, assuming that k <= 17; and 
we have shown that it is legitimate to replace k by 17 if k is larger."

## Converting the other way

Now we've to consider the inverse problem. Given an integer _n_, find a decimal fraction 

0.d1d2...dk

that approximates to 

$$2^-16.n$$ 

so closely that P1 will reproduce _n_ exactly. As an example, if input 8132, we ought to see `0.12408` as output.

It's desirable, Knuth notes, to find _k_ as small as possible; to seek a shortest decimal fraction that will reproduce given value of _n_. 

We also want to choose fraction closeest to n/2^16. For example, both 0.0001 and 0.0002 yield n = 1  but we choose 0.0002 as it's closeest to 1. 

Here is the program P2 as given in the paper:

```
P2: j := 0; s := 10 * n + 5; t := 10;
    repeat if t > 65536 then s := s + 32678 - (t div 2);
        j := j + 1; d[j] := s div 65536;
        s := 10 * (s mod 65536); t := 10 * t;
    until s <= t;
    k := j;
```

Following shows a dry run of P2 on the input n = 8132. Value of _s_ and _t_ in the table show their updated values as they would be at the end of each iteration. 

| Iteration | d[j] | s | t |
|---|---|---|---|---|
| 1 | 1 | 157890 | 100 |
| 2 | 2 | 268180 | 1000 |
| 3 | 4 | 60360 | 10000 |
| 4 | 0 | 603600 | 100000 |
| 5 | 8 | 620800 | 1000000 |


So if you start with a decimal fraction like `0.12408`, the algorithm basically pulls out the digits one at 
a time — first the 1, then the 2, then the 4, and then 0 and finally 8. 

The algorithm works by repeatedly shifting the number left, like sliding digits out of a fraction one by one. 
Each time it shifts, the next digit pops out, and the algorithm records it. This way, the fraction gradually 
reveals itself as a sequence of digits, until there’s no more meaningful information left to extract. 


Knuth observes: 

"Why does this work? Everybody knows Dijkstra's famous dictum that testing can reveal the prescense of errors but not their absence. However, a program like this, with oinly finitely many inputs, is a counterexample! Suppose we test it for all 65536 values of 'n', and suppose the resulting fractions .d1 ... dk all reproduce the original value when converted back. Then we need only verify that none of the shorter fractions or neighboring fractions of equal length are better; this testing will prove the program correct.

But this testing is still not a good way to guarantee correctness, because it gives us no insight into generalizations. Therefore we seek a proof that is comprehensible and educational.

Even more, we seek a proof that reflects the ideas need to create the program, rather than a proof that was concocted ex post facto. The program didn't emerge by itself from a vacuum, nor did I simply try tall possible short programs until I found one that worked."

Above is an important point. We could write a unit test to test for all 65536 values but that doesn't guarentee that none of the 
shorter fractions or nerighboring fractions of equal length are better. 

Knuth thus sought a proof first before creating the program. He wanted the program to emerge from that proof and not to makea proof emerge after 
writing the program.

This shows that P2 emerged after careful thought, analysis, and work. 
The values of _s_ and _t_ didn't emerge out of thin air, but they emerged after careful deliberation and 
mathematical analysis. We will see below how. Also, what's fascinating is how Knuth establishes invariants to ensure
correct values are being calculated for the result. More on this below. 

## Germs of Proof Rework

### Step 1: Setting up the digit strings

Knuth notes: The set of digit strings 

$$d_{j+1} d_{j+2} \dots d_k$$ 

that produce a given result _n_ when preceded by 

$$0.d_1 d_2 \dots d_j$$ 

can be characterized as a set of decimal fractions:

$$
0.d_{j+1} \dots d_k
$$

This set lies within some half-open interval:

$$
[\alpha \dots \beta)
$$


Initially, we have j = 0 and this interval is:

$$
\left[2^{-16}\left(n - \frac{1}{2}\right), \, 2^{-16}\left(n + \frac{1}{2}\right)\right)
$$


Although Knuth is dealing with real numbers at this point but just for curiuosity's sake I wanted to see what 
this interval would be for an integer, so if n = 8132, then the interval is:

$$
\bigl[\,2^{-16}\,(8132 - \tfrac12),\;2^{-16}\,(8132 + \tfrac12)\bigr)
$$

$$
= \bigl[\,0.12407,\; 0.12409\,\bigr)
$$


Although it maybe wrong to view this interval as an integer when the analysis in for real numbers but what's fascinating is 
how tight the interval is i.e. our desired output is close to these values; leaving little room for error. 

### Step 2: Evolving the interval after each digit choice

If we have already chosen some digits, what are the valid choices for the remaining digits?

Knuth notes that permissible values of 

$$d_{j+1}$$ 

are decimal digits _d_ such that:

$$
\left[\frac{d}{10}, \, \frac{d+1}{10}\right)
$$

overlaps with:

$$
[\alpha .. \beta)
$$


### Step 3: Interval evolution after each new digit

After we choose each digit, our interval evolves:

- Initially, it is:

$$
[\alpha .. \beta)
$$

- We select digit _d_

- We increment _j_, and after choosing _d_, the new interval evolves to:

$$
\left[10\alpha - d, \, 10\beta - d\right)
$$

This narrows down the possibility of subsequent digits. 

Now comes the other beautiful aspect of his proof. Above uses real numbers, but Knuth wants to stick to 
integers so he introduces variables _s_ and _t_. How this transition is made to integers and how do _s_ and 
_t_ emerge from this analysis has some beauty to it. Beauty is indeed Knuth's business. 

<details>

<summary>I wanted to see how this interval updates in case of integer n = 8132, please click expand to view the calculation.</summary>

<br/>
Initially, we start with:

$$
[\alpha, \beta] = [0.12407, 0.12409)
$$

As the interval evolves: <br/>

1. First update:  
   $$ [0.2407, 0.2409) $$

2. Second update:  
   $$ [0.407, 0.409) $$

3. Third update:
   $$ [0.07, 0.09) $$

4. Fourth update:
   $$ [0.07, 0.9) $$

5. Final update: 
   $$ [0, 0) $$

<br/>
This process illustrates how the interval shifts left step by step, progressively narrowing towards zero.

</details> 


## Representing the Interval with Integer Variables <span style="text-transform: lowercase;">_s_ and _t_</span>


Knuth introduces two integer variables to represent the interval boundaries more elegantly using integers instead of real numbers:

- _s_ represents the upper bound.
- _t_ represents the decimal scale.

He represents `α` and `β` using _s_ and _t_ as follows:

$$
10\alpha = 2^{-16}(s + t)
$$

$$
10\beta = 2^{-16}t
$$


The initial values of _s_ and _t_ are:

$$
s = 10n + 5, \quad t = 10
$$

### Invariant

The invariant condition for each new digit _d_ is:

$$
0 \leq d \leq 9,\quad s > 2d,\quad s - t \leq 2d + 2^{16}
$$


This invariant relationship is important as we will see more below. This, as I understand it, is a core reason that helps us ensure
the correctnes of results output by P2. 

Whenever we increment _j_ and select a new digit _d_, the interval evolves, and we update _s_ and _t_ as:

$$
s = 10(s - 2^{-16}d)
$$

$$
t = 10t
$$

### Example

Let's work through an example to see how this works for:

$$
n = 8192
$$

The summarized calculations are given below in the table. Detailed calculations for each step, outlined in the table, are given right after the table. 

This table summarizes the key values of _s_, _t_, and _permissible d_ at each step.


| Step               | s | t | Inequality 1  | Inequality 2 | Permissible d |
|--------------------|------------------|------------------|----------------------------|----------------------------|------------|
| Initial            | 81325            | 10               | d < 1.240                   | d > 0.24                       | 1          |
| After 1st update   | 157890           | 100              | d < 2.409                   | d > 1.40                       | 2          |
| After 2nd update   | 268180           | 1000             | d < 4.092                   | d > 3.07                       | 4          |
| After 3rd update   | 60360            | 10000            | d < 0.921                   | d > 0                          | 0          |
| After 4th update   | 603600           | 100000           | d < 9.210                   | d > 6.68                       | 7, 8, 9          |



 
<details>

<summary>Click to expand the detailed calculation</summary>


<h3> First digit </h3><br/>

Given that the initial values are:
$$
s = 81325, 
t = 10
$$

We've two inequalities to satisfy:

1. $$ s > d \cdot 2^{16} $$ 
2. $$ (s - t) < 2^{16} \cdot d + 2^{16} $$

Since:

$$2^{16} = 65536$$ 

Let's evaluate for 's = 81325' and 't = 10' and evaluate the first inequality:

$$
81325 > 65536 \cdot d
$$

Solving for _d_:

$$
d < \frac{81325}{65536} \approx 1.240
$$



This allows:

$$
d = 0 \text{ or } d = 1
$$

Now evaluate the second inequality:

$$
81325 - 10 < 65536 \cdot d + 65536
$$

This simplifies to:

$$
81315 < 65536 \cdot (d+1)
$$

Solving for _d_:

$$
d+1 > \frac{81315}{65536} \approx 1.2407
$$


Thus:

$$
d \geq 1
$$

The only value that works is:

$$\mathbf{d = 1}$$

<div style="background-color: #fff8c4; padding: 5px; display: inline-block;">
Using the invariant relationship, we find out first digit which is 1. </div> <br/>


<h3>  Second digit </h3> <br/>

Now we've to update _s_ and _t_ to find out what next digit holds the invariant. 

Now:

$$
s = 10 \cdot \left( 81325 - 2^{16} \cdot 1 \right) \\
t = 100
$$

This evaluates to:

$$
s = 10 \cdot (81325 - 65536) = 10 \cdot 15789 = 157890
$$

$$
s - t = 157890 - 100 = 157790
$$

Let's now evaluate the first inequality: 

$$
157890 > 65536 \cdot d
$$

Solving for _d_:

$$
d < \frac{157890}{65536} \approx 2.409
$$

This allows:

$$
d = 0, 1, 2
$$

Now let's evaluate the second inequality:

$$
157790 < 65536 \cdot (d+1)
$$

Solving for _d_:

$$
d+1 > \frac{157790}{65536} \approx 2.407
$$

This gives:

$$
d > 1.407 \implies d \geq 2
$$

The only value that works is:

$$
d = 2
$$

<div style="background-color: #fff8c4; padding: 5px; display: inline-block;">
This gives the next possible value 2. 
</div> <br/>
I find it beautiful how this invariant holds and how keep getting our next digits. 

It's time to update _s_ and _t_ again. 

<h3> Third digit </h3> <br/>

Now:

$$
s = 10 \cdot \left(157890 - 2^{16} \cdot 2 \right) \\
t = 1000
$$

This evaluates to:

$$
s = 10 \cdot (157890 - 2 \cdot 65536) = 10 \cdot (157890 - 131072) = 10 \cdot 26818 = 268180
$$

$$
s - t = 268180 - 1000 = 267180
$$

Let's evaluate the first inequality:

$$
268180 > 65536 \cdot d
$$

Solving for _d_:

$$
d < \frac{268180}{65536} \approx 4.092
$$

This allows:

$$
d = 0, 1, 2, 3, 4
$$

Let's evaluate the second inequality:

$$
267180 < 65536 \cdot (d+1)
$$

Solving for _d_:

$$
d+1 > \frac{267180}{65536} \approx 4.076
$$

This gives:

$$
d > 3.076 \implies d \geq 4
$$

The only value that works is:

$$
d = 4
$$

<div style="background-color: #fff8c4; padding: 5px; display: inline-block;">So the next digit is 4 </div><br/>
Isn't it beautiful how do we kep getting our next digit through the lens of this invariant?

<h3> Fourth digit </h3> <br/>

Alright, it's time update _s_ and _t_ again. 

Now:

$$
s = 10 \cdot \left(268180 - 2^{16} \cdot 4 \right) \\
t = 10000
$$

This evaluates to:

$$
s = 10 \cdot (268180 - 4 \cdot 65536) = 10 \cdot (268180 - 262144) = 10 \cdot 6036 = 60360
$$

$$
s - t = 60360 - 10000 = 50360
$$

Evalauting the first inequality we get:

$$
60360 > 65536 \cdot d
$$

Solving for _d_:

$$
d < \frac{60360}{65536} \approx 0.921
$$

This allows:

$$
d = 0
$$

Now the second inequality:

$$
50360 < 65536 \cdot (d+1)
$$

Solving for _d_:

$$
d+1 > \frac{50360}{65536} \approx 0.768
$$

This gives:

$$
d > -0.23
$$

This allows:

$$
d \geq 0
$$

The only value that works is:

$$
d = 0
$$

<div style="background-color: #fff8c4; padding: 5px; display: inline-block;">
This gives us our next digit which is 0. </div> <br/>

<h3> Fifth digit </h3> <br/>

Let's update _s_ and _t_ one last time to get our final digit. 

$$
s = 10 \cdot \left(60360 - 2^{16} \cdot 0 \right) \\
t = 100000
$$

Since:

$$
2^{16} \cdot 0 = 0
$$

This gives:

$$
s = 10 \cdot (60360 - 0) = 10 \cdot 60360 = 603600
$$

$$
s - t = 603600 - 100000 = 503600
$$

The first inequality is:

$$
s > d \cdot 2^{16}
$$

This becomes:

$$
603600 > 65536 \cdot d
$$

Solving for _d_:

$$
d < \frac{603600}{65536} \approx 9.21
$$

This allows:

$$
d = 0, 1, 2, \dots, 9
$$

The second inequality is:

$$
s - t < 2^{16} \cdot d + 2^{16}
$$

This becomes:

$$
503600 < 65536 \cdot d + 65536
$$

Solving for _d_:

$$
d+1 > \frac{503600}{65536} \approx 7.684
$$

This gives:
_t__d__d_
$$
d > 6.684 \implies d \geq 7
$$

First inequality allows:

$$
d = 0, 1, \dots, 9
$$

Second inequality allows:

$$
d \geq 7
$$

So the integer values of _d_ satisfying both inequalities are

$$
d = 7, 8, 9
$$

<div style="background-color: #fff8c4; padding: 5px; display: inline-block;">
We can continue further, but for now complete the chain of calculations to get our answer 
which is 0.12408. </div><br/>

</details>


The values of permisslbe _d_ in the above table are determined by the invariant that we just setup above. 

This idea of a keeping an invariant to guard the permissibility of the next possible digit in the output 
is remarkable. At each step, for every permissible value(s) of _d_, it's beautiful how tight the interval is, how close permissible 
digits are to the actual digit required in the result at that step, reducing the 
likelihood of an incorrect value. 

Next, Knuth tackles how to optimally choose the fifth decimal digit in P2. We'll see just in a moment why the fifth digit and why this adjustment matters. 
To optimally choose this fifth decimal digit, Knuth does an adjustment of _s_ in this conditional in P2:

```if (t > 65536) then (s := s + 32768 - (t div 2));```

In P2, after the first four iterations, if the input _n_ is 8132, we have:

(d[1] = 1, d[2] = 2, d[3] = 4, d[4] = 0

And the values of _s_ and _t_ are:

s = 603600, t = 100000

If we take these values of _s_ and _t_ and do not do any adjustment (using the above conditional), then d[5] would be 9. 
But is 9 the best choice for digit 5? So we know that: 

$$
\frac{8132}{2^{16}} = 0.12408447265625
$$


We can see that `0.12408` is closer to the true value than `0.12409` here.

So 8 is a better choice for d[5] than 9 for correctness reasons and this is why the choice of fifth digit matters. 
It's fascinating how Knuth goes for optimal rounding using the above conditional for choice for d[5] than 9 for correctness reasons. 

Overall two beautiful insights emerge after ensuring P2's correctness:

- How an invariant relationship checks for the permissibility of each decimal digit in the outcome. If the outcome is
`0.d1d2d3d4d5` then the invariant ensures first that _d1_ is permissible, then it checks for _d2_, then for _d3_, again for _d4_, 
and lastly for _d5_. 

- How the proof elegantly and optimally rounds the fifth digit. 

Knuth didn't leave any crucial detail unaddressed. 

Each part of the program is critically analzed and evaluated for correctness. As a programmer, 
what inspires me here is to strive for such attention to detail. Also, to strive for determining invariants whenever I 
can for every crucial part of my algorithm. 

Yes, unit testing is crucial and so is 
analysis aided by formal-methods and model-checkers but so is carefully determining invariants (at least for the critical code). 
I say this after spending a couple of days last week chasing a bug which was not caught by unit tests and most likely wouldn't have been 
caught by high-level abstractions of model-checkers and the team agreed that a good invariant could have prevented it. 

Just for curiosity's sake, I implemented both P1 and P2 in Python and then wrote a small unit test to test for most values. 

All code is in this Github repository. 

----





