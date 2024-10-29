---
layout: post
title:  "The Beautiful Simplicity of the Gentzen System: A Deductive Framework"
date: 2024-10-29
categories: cs
math: true
---

Gentzen system, created by German mathematician Gerhard Gentzen, is a deductive system which
can be used to prove propositional formulas. I recently learned about it while I was reading Ben-Ari's fantastic book on mathematical logic [1] and I like its simplicity.  

Why should we care about the Gentzen system? Let's say you're a programmer, why should you care? Fair question. 

I recently started learning more about mathematical logic, and I’ve realized that, just like writing can clarify your thoughts, formal mathematical can bring coherence to your thinking. If your arguments aren’t logically sound, you won’t be able to prove what you set out to prove—mathematical reasoning helps you construct those logically sound arguments.

I’ve been programming for a while now, mostly self-taught, and I’ve observed that while learning formalism isn’t necessary 
for most programming jobs, it definitely helps you think more carefully about the correctness 
of your code. It helps you reason through problems with greater precision.

Now let's dive into the Gentzen system, starting with a scenario.

Imagine you're a detective trying to solve a robbery. You need to prove that a certain person robbed the house. 
You assemble truths to establish the robbery. For example, you find someone entering the house on a video camera 
on the day of the robbery, and you later also find a written receipt of the same person selling items that belong 
to the house's owner. Now, by presenting an irrefutable truth and supporting evidence you can establish the claim of robbery.

Deductive systems work the same way. You establish a truth using statements assumed to be true, called axioms and you 
also use some inference rules to support your claim. The Gentzen system is remarkably simple. It offers just one axiom and a few rules of 
inference, yet with these, you can prove complex propositional fomrulas. There’s a certain beauty in seeing a 
simple system—built on a single axiom and a handful of rules—solving complex tasks with unshakable logical coherence.

**Gentzen’s Genius: A Single Axiom and a Few Inference Rules**

For me, what makes the Gentzen System is its minimalism. Some logical systems come with a long list of axioms 
and complicated rules to prove new things. Gentzen system’s approach is surprisingly simple: one axiom and just
 a handful of inference rules. And then you can use it to prove complex propositional formulas.

Let's see how Gentzen system can be used to prove a propositional formula 
(Please note that I am teaching myself these concepts by learning from this wonderful book [1]. 
Any errors are my own, and I’m happy to correct anything you find incorrect. I'm writing this to reinforce my own understanding, 
following Confucius' advice: "I hear and I forget. I see and I remember. I do and I understand.").

A little context about propositional logic and formulas: A propositional formula, like $ p \lor q $ (which is 
basically 'p `or` q', a [logical disjunction](https://en.wikipedia.org/wiki/Logical_disjunction)), uses atomic propositions. 
The atomic propositions in $ p \lor q $ are $ p $ and $ q $. The atomic propositions can be assigned values $ \text{true} $ or $ \text{false} $. 
If $ p $ is $ \text{true} $, then $ \neg p $ (not $ p $) is $ \text{false} $. A complementary pair is a pair containing both $ p $ and $ \neg p $.

And now the axiom and a few words about inference rules. 

**The Axiom**: Gentzen system starts with only one axiom: a complementary pair, e.g., $ p $ and $ \neg p $. That's the only axiom in the Gentzen system. 

**The Inference Rules**: The system also provides inference rules. These rules, alongside the one axiom, allow us to prove new things. 
There are two types of inference rules: $ \alpha $ (alpha) and $ \beta $ (beta). Here are some details about each of them:

- $ \alpha $ inference rules: Think of these rules as ways to shape two logical formulas into a propositional formula. 
Let's say you have two propositional formulas, $ A1 $ and $ A2 $, then you can combine them into a single $ \alpha $ formula. 
There are a few $ \alpha $ rules, but for simplicity, I'm not including them all here—they are all in Ben-Ari's book. 
Let’s look at two $ \alpha $ inference rules as we will use these both in a proof below. 

    - If you have two formulas, $ A1 $ and $ A2 $, then you can write them as $ A1 \lor A2 $. 
    - If you have $ \neg A1 $ and $ A2 $, then you can write them as $ A1 \rightarrow A2 $ (A1 implies A2).


- $ \beta $ inference rules: There are also a few $ \beta $ rules. Like $ \alpha $ rules, you can use them to simplify 
two $ \beta $ formulas. Here is one inference rule we'll use in a proof below. 
    - If you have $ \neg B1 $ and $ \neg B2 $, then you can write them as $ \neg (B1 \lor B2) $.

Now let's try to do a proof using the above axiom and inference rules.

We want to prove $ (p \lor q) \rightarrow (q \lor p) $ using the Gentzen system.

Basically, in such proofs, you proceed step by step, using the outcome of each step to derive the next one, eventually leading to the proof.

Prove: $ (p \lor q) \rightarrow (q \lor p) $

Here is the proof (sorry for the extra text due to the added explanation):

1. We start by examining the propositional formula we want to prove. Which atomic propositions does it invovle? Well, $ p $ and $ q $. 
For p, we apply the axiom, giving us both $ p $ and $ \neg p $. We then combine this with $ q $, so the outcome of the first step is: $ p, \neg p, q $.


2. Next, we apply the axiom again, this time for $ q $. This gives us both $ q $ and $ \neg q $. So, after the second step, we have: $ \neg q, q, p $.

3. Now, we can apply a $ \beta $ inference rule to the results of the first two steps. We have $ \neg p $ from the first step and $ \neg q $ 
from the second step. According to the $ \beta $ inference rule, if we have $ \neg B1 $ and $ \neg B2 $, we can combine them into $ \neg (B1 \lor B2) $. 
So, we treat $ \neg p $ as B1 and $ \neg q $ as B2. Applying the rule gives us $ \neg (p \lor q) $. The result of this third step is $ \neg (p \lor q) $, with p and q carrying over from the first two steps. Now, to explain the carryover: when we use $ \neg p $ and $ \neg q $ to 
form $ \neg (p \lor q) $, the remaining elements, $ p $ and $ q $, from both steps are combined with $ \neg (p \lor q) $. Since we are working with sets, 
we only keep unique elements, so the result is $ \neg (p \lor q), p, q $.


4. In this step, we apply an $ \alpha $ rule. According to this rule, if we have A1 and A2, we can combine them into $ A1 \lor A2 $. 
From step 3, we have $ q $ and $ p $. Applying the alpha rule to these two propositions gives us $ q \lor p $. Importantly,  
 $ \neg (p \lor q) $ is unchanged, so it carries over. The result of this step is: $ \neg (p \lor q), (q \lor p) $.

5. Now, let’s look at what the result of step 4. Can we apply a rule to get the formula we set out to prove? Yes! We can use another $ \alpha $ rule, 
which says if we have $ \neg A1 $ and $ A2 $, we can form $ A1 \rightarrow A2 $ (A1 implies A2). In step 4, we have $ \neg (p \lor q) $ 
as $ \neg A1 $ and $ (q \lor p) $ as $ A2 $. Using this rule, we arrive at $ (p \lor q) \rightarrow (q \lor p) $, which is exactly what we set out to prove. Voila!

Here is a brief summary of the above steps. In the first two steps of the proof below, we start with axioms. Then, in the third step, we apply the $ \beta $ inference rule on the axioms (the outcome of the first and second step). Then, on the result of the third step, we apply an $ \alpha $ inference rule. Finally, on the outcome of the fourth step, we apply another $ \alpha $ rule, and we are be able to prove what we set out to prove.

At first, I didn't understand this by just reading it—it seemed too clever (as the author also hinted that it seemed clever). I had to solve it a couple of times using pen and paper.

This can also be presented in the tree form. In the screenshot below, the top nodes are axioms, the internal nodes are inference rules, and the node at the bottom is the formula that needed to be proved:

<img loading="lazy" src="{{ site.baseurl }}/images/gentzen_tree.png" alt="gentzen tree" width="500" />

There is an inexplicable beauty in the idea that you start with only one established truth and a few inference rules, and using them, you can prove complex propositional formulas.

I believe the principles of deductive systems can be applied to computer programming. Many of the skilled programmers 
I’ve worked with approach problems by understanding things at their most fundamental level—starting with first principles. 
They firmly grasp the core "axioms" of the system, and then reason their way up. They rarely violate the system’s properties, 
and when anomalies arise, they can spot them easily.

This approach is essential when dealing with complex systems. Without a solid foundational understanding, 
you develop a shaky intuition that won’t hold up when facing tough challenges, especially under time pressure. I think we 
can learn from deductive systems like Gentzen to build this same foundational understanding when working with complex codebases.

I think this idea connects to other fields, like technical writing, business writing, or even essays. Great writers build their case 
on solid "axioms" and strong inferences. As a writer, if the picture you paint for your readers is made up of points that are hard 
to refute, it’s difficult for the readers not to be convinced. Good writing leaves little room for logical flaws.  

Creating a system with just one axiom and a few inference rules that can prove complex propositional formulas is a sign of elegance. This kind of elegance inspires us to use such systems 
as a framework for learning and mastering complexity in other fields.

### References:

1. Mathematical Logic for Computer Science by Mordechai Ben-Ari, 3rd edition.
