---
layout: post
title:  "A simple program by Knuth whose proof isn't"
date: 2025-2-24
categories: formal-methods
math: true
---

I accidentlly stumbled upon this book "Beauty is our business, A birthday Salute to Edsger W. Dijkstra" edited by W.H.J Feijen et al. As to what this book is about, here is an excerpt from preface: "More than anything else, this book is a tribute to Edsger W. Dijkstra, on the 
occassion of his sixtieth birthday, by just a few of those fortunate enough to be influenced by him and his work and to be called his friend, or relation, his matster, colleague, or pupil." 

And in the preface in another paragraph down below the previous one, it says, "In the end, on 
Dijkstra's fifty-ninth birthday, we asked fifty-odd friends and colleages of his to contribute to this birthday salute. We requested 
mainly technical contributions, and we asked that they be relatively short. Fifty-odd authors responded, and we are proud to place their contributions before Edsger Dijistra." 

As for how how did they choose this title, the preface shares a quote by Dijkstra that inspired the title. That Dijkstra's quote is 
worth sharing:"...when we recognize the battle against chaos, mess, and unmastered complexity as one of computing science's major 
callings, we must that 'Beauty is our business.'"

So after buying the book, I was skimming through table of contents and I came across an essay by Donald E. Knuth and I thought that would
be fun. Knuth's essay is titled "A simple program  whose proof isn't."

In this contribution, Knuth shares two programs. The first program converts decimal fraction to fixed-point binary and he shares that
he finds it easy to proof this is correct. The second program converts the other way, given an integer 'n', it finds a decimal fraction and it's the correctness of this program which isn't straightforward. Let's talk about these one by one. 

## Convert decimal fraction to fixed-point binary

Knuth had to write these routiens he was working on Tex (a typesetting program). Tex works with integer multiples of 2^-16. And he had
to write a routine to convert 0.d1d2...dk to nearest possible binary fraction. 

It's fascinating how Knuth developed this program. 

We need to find the nearest integer multiple of \(2^{-16}\). We want to round the quantity:  
$$
2^{16} \sum_{j=1}^k \frac{d_j}{10^j}
$$

to the nearest integer \(n\). If two integers are equally near, we let \(n\) be the integer:
$$
n = \left\lfloor 2^{16} \sum_{j=1}^k \frac{d_j}{10^j} + \frac{1}{2} \right\rfloor
$$

Let's say we have the input \(0.12408\), then:
$$
n = 2^{16} \cdot \left( \frac{1}{10^1} + \frac{2}{10^2} + \frac{4}{10^3} + \frac{0}{10^4} + \frac{8}{10^5} \right) + \frac{1}{2}
$$

This evaluates to:
$$
n = 8132
$$

Since input values \(d_j\) are small nonnegative integers, Knuth wanted to keep the intermediate results reasonably small.

Also, since \(k\) can be arbitrarily large, Knuth had concerns that these could become too large for our computers' hardware. He demonstrated that **17 digits are not only sufficient — they are sometimes necessary** to determine the correct value of \(n\). We can safely ignore \(d_j\) for \(j > 17\), but **do the math yourself** to verify. Knuth’s attention to detail here is brilliant. 

Here is this program called P1, as outlined in the paper:

```
P1: l := min(k, 17); m := 0;

repeat m := (131072 * d[l] + m) div 10;
       l := l - 1;
until l = 0;

n := (m + 1) div 2
```

Following shows a dry run of program P1 on input `0.12408`. The digits of the input 0.12408 are processed in reverse order from d[5] to d[1]: 

| Iteration | Current 'l' | d[l] | Updated 'm' | Updated 'l' |
|---|---|---|---|---|---|
| 1 | 5 | 8 | 104857 | 4 |
| 2 | 4 | 0 | 10485 | 3 |
| 3 | 3 | 4 | 53477 | 2 |
| 4 | 2 | 2 | 31562 | 1 |
| 5 | 1 | 1 | 16263 | 0 |

---

Final Step when l = 0:
$$
n = \frac{m + 1}{2} = \frac{16263 + 1}{2} = \frac{16264}{2} = 8132
$$

Thus, converting \(0.12408\) via **P1** produces:
$$
\mathbf{n = 8132}
$$

Here is a Python version of this:


As an example, we could input 0.124 and it would convert it into 8132. 

Knuth shares that the proof is easy for this one. "We have m = m1 at the beginnning of the repeat loop, assuming that k <= 17; and 
we have shown that it is legitimate to replace k by 17 if k is larger."

## Converting the other way

Now we've to consider the inverse problem. Given an integer 'n', find a decimal fraction 

0.d1d2...dk

that approximates to 2^-16n so closely that P1 will reproduce 'n' exactly. As an example, if input 8132, we ought to see 0.124 as output.

It's desirable, Knuth notes, to find 'k' as small as possible; to seek a shortest decimal fraction taht will reproduce given value of 'n'. 
We also want to choose fraction closeest to n/2^16. For example, both 0.0001 and 0.0002 yield n=1  but we choose 0.0002 as it's closeest to 1. Here is the program P2 as given in the paper:

```
P2: j := 0; s := 10 * n + 5; t := 10;
    repeat if t > 65536 then s := s + 32678 - (t div 2);
        j := j + 1; d[j] := s div 65536;
        s := 10 * (s mod 65536); t := 10 * t;
    until s <= t;
    k := j;
```

And Knuth observed: 

" Why does this work? Everybody knows Dijkstra's famous dictum that testing can reveal the prescense of errors but not their absence. However, a program like this, with oinly finitely many inputs, is a counterexample! Suppose we test it for all 65536 values of 'n', and suppose the resulting fractions .d1 ... dk all reproduce the original value when converted back. Then we need only verify that none of the shorter fractions or neighboring fractions of equal length are better; this testing will prove the program correct.

But this testing is still not a good way to guarantee correctness, because it gives us no insight into generalizations. Therefore we seek a proof that is comprehensible and educational.

Even more, we seek a proof that reflects the ideas need to create the program, rather than a proof that was concocted ex post facto. The program didn't emerge by itself from a vacuum, nor did I simply try tall possible short programs until I found one that worked."

That above is an important one. 

## Germs of Proof Rework

The way Knuth establishes the invariant is beautiful.

First, let me explain how this proof works, and then I will walk through it carefully.

### Step 1: Setting up the digit strings

Knuth notes: The set of digit strings 

$$d_{j+1} d_{j+2} \dots d_k$$ 

that produce a given result \(n\) when preceded by 

$$0.d_1 d_2 \dots d_j$$ 

can be characterized as a set of decimal fractions:

$$
0.d_{j+1} \dots d_k
$$

This set lies within some **half-open interval**:

$$
[\alpha \dots \beta)
$$


Initially, we have \(j = 0\) and this interval is:

$$
\left[2^{-16}\left(n - \frac{1}{2}\right), \, 2^{-16}\left(n + \frac{1}{2}\right)\right)
$$

### Step 2: Evolving the interval after each digit choice

If we have already chosen some digits, what are the valid choices for the remaining digits?

Knuth notes that permissible values of 

$$d_{j+1}$$ 

are decimal digits \(d\) such that:

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

- We select digit \(d\)

- We increment \(j\), and after choosing \(d\), the new interval shrinks to:

$$
\left[10\alpha - d, \, 10\beta - d\right)
$$

This narrows down the possibility of subsequent digits. 

Now comes the other beautiful aspect of his proof. Above uses real numbers, but Knuth wants to stick to 
integers so he introduces variables 's' and 't'. 

## Representing the Interval with Integer Variables 's' and 't'

Knuth introduces two integer variables to represent the interval boundaries more elegantly using integers instead of real numbers:

- `s` represents the upper bound.
- `t` represents the decimal scale.

He represents `α` and `β` using `s` and `t` as follows:

$$
10\alpha = 2^{-16}(s + t)
$$

$$
10\beta = 2^{-16}t
$$


The initial values of `s` and `t` are:

$$
s = 10n + 5, \quad t = 10
$$

### Invariant

The invariant condition for each new digit `d` is:

$$
0 \leq d \leq 9,\quad s > 2d,\quad s - t \leq 2d + 2^{16}
$$

Whenever we increment `j` and select a new digit `d`, the interval evolves, and we update `s` and `t` as:

$$
s = 10(s - 2^{-16}d), \quad t = 10t
$$

### Example

Let's work through an example to see how this works for:
$$
n = 8192
$$


#### First digit

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

Solving for 'd':

$$
d < \frac{81325}{65536} \approx 1.241
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

Solving for \(d\):

$$
d+1 > \frac{81315}{65536} \approx 1.2408
$$


Thus:

$$
d \geq 1
$$

The only value that works is:

$$\mathbf{d = 1}$$

<div style="background-color: #fff8c4; padding: 5px; display: inline-block;">
Using the invariant relationship, we find out first digit which is 1. </div> <br/>


#### Second digit

Now we've to update 's' and 't' to find out what next digit holds the invariant. 

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

Solving for \(d\):

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

Solving for \(d\):

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

It's time to update 's' and 't' again. 

#### Third digit

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

Solving for \(d\):

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

Solving for \(d\):

$$
d+1 > \frac{267180}{65536} \approx 4.078
$$

This gives:

$$
d > 3.078 \implies d \geq 4
$$

The only value that works is:

$$
d = 4
$$

<div style="background-color: #fff8c4; padding: 5px; display: inline-block;">So the next digit is 4 </div><br/>
Isn't it beautiful how do we kep getting our next digit through the lens of this invariant?

#### Fourth digit

Alright, it's time update 's' and 't' again. 

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

Solving for \(d\):

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

Solving for \(d\):

$$
d+1 > \frac{50360}{65536} \approx 0.768
$$

This gives:

$$
d > -0.231
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

#### Fifth digit

Let's update 's' and 't' one last time to get our final digit. 

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

Solving for \(d\):

$$
d < \frac{603600}{65536} \approx 9.204
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

Factor out \(65536\):

$$
503600 < 65536 \cdot (d+1)
$$

Solving for \(d\):

$$
d+1 > \frac{503600}{65536} \approx 7.685
$$

This gives:

$$
d > 6.685 \implies d \geq 7
$$

First inequality allows:

$$
d = 0, 1, \dots, 9
$$

Second inequality allows:

$$
d \geq 7
$$

The overlap gives:

$$
d = 7, 8, 9
$$

To find the **largest valid \(d\)** within range \(0 \leq d \leq 9\), this gives:

$$
d = 8
$$

<div style="background-color: #fff8c4; padding: 5px; display: inline-block;">
We can continue further, but for now complete the chain of calculations to get our answer 
which is 0.12408. </div><br/>

We can summarize above steps in a table below.

This table summarizes the key values of \(s\), \(t\), and \(d\) at each step.


## Final Summary Table

<table border="1" style="border-collapse: collapse; width: 100%; font-family: Arial, sans-serif; font-size: 16px;">
<thead>
    <tr>
        <th style="text-align: center; padding: 8px;">Step</th>
        <th style="text-align: center; padding: 8px;">s</th>
        <th style="text-align: center; padding: 8px;">t</th>
        <th style="text-align: center; padding: 8px;">Inequality 1 (upper bound)</th>
        <th style="text-align: center; padding: 8px;">Inequality 2 (lower bound)</th>
        <th style="text-align: center; padding: 8px;">Computed d</th>
    </tr>
</thead>
<tbody>
    <tr style="margin-bottom: 5px;">
        <td style="padding: 8px;">Initial</td>
        <td style="padding: 8px;">81325</td>
        <td style="padding: 8px;">10</td>
        <td style="padding: 8px;">d &lt; 1.241</td>
        <td style="padding: 8px;">d ≥ 1</td>
        <td style="padding: 8px;">1</td>
    </tr>
    <tr style="margin-bottom: 5px;">
        <td style="padding: 8px;">After 1st Update</td>
        <td style="padding: 8px;">157890</td>
        <td style="padding: 8px;">100</td>
        <td style="padding: 8px;">d &lt; 2.409</td>
        <td style="padding: 8px;">d ≥ 2</td>
        <td style="padding: 8px;">2</td>
    </tr>
    <tr style="margin-bottom: 5px;">
        <td style="padding: 8px;">After 2nd Update</td>
        <td style="padding: 8px;">268180</td>
        <td style="padding: 8px;">1000</td>
        <td style="padding: 8px;">d &lt; 4.092</td>
        <td style="padding: 8px;">d ≥ 4</td>
        <td style="padding: 8px;">4</td>
    </tr>
    <tr style="margin-bottom: 5px;">
        <td style="padding: 8px;">After 3rd Update</td>
        <td style="padding: 8px;">60360</td>
        <td style="padding: 8px;">10000</td>
        <td style="padding: 8px;">d &lt; 0.921</td>
        <td style="padding: 8px;">d ≥ 0</td>
        <td style="padding: 8px;">0</td>
    </tr>
    <tr style="margin-bottom: 5px;">
        <td style="padding: 8px;">After 4th Update</td>
        <td style="padding: 8px;">603600</td>
        <td style="padding: 8px;">100000</td>
        <td style="padding: 8px;">d &lt; 9.204</td>
        <td style="padding: 8px;">d ≥ 7</td>
        <td style="padding: 8px;">8</td>
    </tr>
</tbody>
</table>






The idea of a keeping an invariant to check ensure that the next possible digit in the output guards your output is fascinating. 


----





