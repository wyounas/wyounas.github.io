---
layout: post
title:  "A simple program by Knuth whose proof isn't"
date: 2025-2-24
categories: formal-methods
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

Convert decimal fraction to fixed-point binary

Knuth had to write these routiens he was working on Tex (a typesetting program). Tex works with integer multiples of 2^-16. And he had
to write a routine to convert 0.d1d2...dk to nearest possible binary fraction. Since input values dj are small nonnegative integers, it
was desirable to keep intermediate results small. Knuth had this fascinating insight that since j < 17 have no effect on the answer, Tex 
need only maintain an array capable of olding up to 17 digits, and all after 17 could be discarded. Here is this program called P1, as 
outlined in the paper:

P1: l := min(k, 17), m := 0;
    repeat m := (131072 * d[l] + m) div 10;
        l := l - 1;
    until l = 0;
    n := (m + 1) div 2;


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