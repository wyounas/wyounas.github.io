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

Converting the other way

Now we've to consider the inverse problem. Given an integer 'n', find a decimal fraction 

0.d1d2...dk

that approximates to 2^-16n so closely that P1 will reproduce 'n' exactly. As an example, if input 8132, we ought to see 0.124 as output.

It's desirable, Knuth notes, to find 'k' as small as possible; to seek a shortest decimal fraction taht will reproduce given value of 'n'. 
We also want to choose fraction closeest to n/2^16. For example, both 0.0001 and 0.0002 yield n=1  but we choose 0.0002 as it's closeest to 1. Here is the program P2 as given in the paper:

P2: j := 0; s := 10 * n + 5; t := 10;
    repeat if t > 65536 then s := s + 32678 - (t div 2);
        j := j + 1; d[j] := s div 65536;
        s := 10 * (s mod 65536); t := 10 * t;
    until s <= t;
    k := j;

And Knuth observed: 

" Why does this work? Everybody knows Dijkstra's famous dictum that testing can reveal the prescense of errors but not their absence. However, a program like this, with oinly finitely many inputs, is a counterexample! Suppose we test it for all 65536 values of 'n', and suppose the resulting fractions .d1 ... dk all reproduce the original value when converted back. Then we need only verify that none of the shorter fractions or neighboring fractions of equal length are better; this testing will prove the program correct.

But this testing is still not a good way to guarantee correctness, because it gives us no insight into generalizations. Therefore we seek a proof that is comprehensible and educational.

Even more, we seek a proof that reflects the ideas need to create the program, rather than a proof that was concocted ex post facto. The program didn't emerge by itself from a vacuum, nor did I simply try tall possible short programs until I found one that worked."

That above is an important one. 