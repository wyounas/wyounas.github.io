---
layout: post
title:  "How to Help Santa Claus Concurrently"
date: 2025-12-23
categories:  [puzzles, concurrency]
---

I came across the Santa Claus concurrency puzzle [^1], which is stated as follows:

> Santa Claus sleeps at the North Pole until awakened by either all of the nine reindeer or by a group of three out of ten elves: He performs one of two indivisible actions:
> 
> - If awakened by the group of reindeer, Santa harnesses them to a sleigh, delivers toys, and finally unharnesses the reindeer, who then go on vacation.
> - If awakened by a group of elves, Santa shows them into his office, consults with them on toy R&D, and finally shows them out so they can return to work constructing toys.
> 
> A waiting group of reindeer must be served by Santa before a waiting group of elves. Since Santa’s time is extremely valuable, marshalling the reindeer or elves into a group must not be done by Santa.”

The puzzle captures synchronization challenges that arise whenever multiple processes must coordinate. Rather than just solving it, I wanted to validate the solution’s correctness using a model checker. To do this, I use SPIN and Promela, SPIN’s specification language. 

A natural question is why use a model checker instead of simply writing the solution in Python or Go. The difference is coverage. A model checker explores all possible interleavings of concurrent execution, not just the ones exercised by tests or experiments. A program can appear correct in many runs and still fail under an untested schedule; a model checker either proves correctness for all executions or produces a concrete counterexample. Once we have such a model, we can then translate it into a conventional programming language with much greater confidence.

<img loading="lazy" src="{{ site.baseurl }}/images/2025-12-23-how-to-help-santa-concurrently/first.png" alt="Santa claus puzzle" />

(If the images appear small, please open them in a new browser tab.)

Let’s look at one possible sequence of events.


<img loading="lazy" src="{{ site.baseurl }}/images/2025-12-23-how-to-help-santa-concurrently/second.png" alt="Santa claus puzzle" />

In the sequence above, when nine reindeer arrive, Santa delivers toys with them and then releases them. Later, when three elves arrive, Santa consults with them and releases them as well.

But this sequence is not entirely faithful to the puzzle’s specification. In particular, it violates the first of these following three key constraints:

- Santa must not marshal the groups himself.

- If both reindeer and elves are waiting, Santa must serve the reindeer first (Christmas delivery is critical).

- When Santa serves a group, the entire group must participate together: exactly nine reindeer for delivery, or exactly three elves for consultation. Santa cannot deliver toys with seven reindeer or consult with two elves, for example.

In the sequence above, Santa was effectively marshalling the group himself, which he should not. To prevent this, we use a room-based approach, inspired by Ben-Ari's [^1]. A room collects arrivals and signals Santa only when a complete group is ready. Santa never counts arrivals himself; he merely responds to these “group is ready” signals.

We use one room for the reindeer and another for the elves. Let’s now look at the modified sequence of events.

<img loading="lazy" src="{{ site.baseurl }}/images/2025-12-23-how-to-help-santa-concurrently/third.png" alt="Santa claus sequence" />

When a group of nine reindeer arrives at the reindeer room, the room signals Santa. While this is happening, two elves also arrive. Santa delivers toys and releases the reindeer. When the third elf later arrives, the elf room signals Santa; Santa consults with the elves and then releases them.
To model this behavior in SPIN, each component is represented as a separate process. SPIN explores all possible interleavings of these processes and checks that specified correctness properties hold in every reachable state.

We model the system using the following processes:

_Reindeer_: We run nine reindeer processes concurrently. Each reindeer arrives at the reindeer room and waits there until it is released.

_Elves_: We run ten elf processes concurrently. Each elf arrives at the elf room and waits there until it is released.

_Reindeer room_: We run a single instance of this process, which simulates the reindeer room. It collects arriving reindeer, signals Santa when a full group of nine has formed, and releases the group after Santa completes delivery.

_Elf room_: Similarly, we run a single instance of this process to simulate the elf room. It collects arriving elves, signals Santa when a group of three has formed, and releases the group after Santa completes consultation.

_Santa_: A single Santa process waits for “group ready” signals from the rooms. It serves reindeer with priority over elves and notifies the appropriate room when its work is complete.

To communicate between processes, we use channels. In SPIN, a channel is a data type that supports two operations: send and receive. Channels transfer messages of a specified type from a sender to a receiver.

SPIN supports two kinds of channels. In this model, we use rendezvous channels. In a rendezvous channel, send and receive synchronize: the sender blocks until the receiver is ready (and vice versa), and the transfer occurs as one atomic handshake.

We use the following rendezvous channels, each carrying messages of type _bit_:

```promela
chan r_arrive  = [0] of { bit };
chan e_arrive  = [0] of { bit };
chan r_release = [0] of { bit };
chan e_release = [0] of { bit };
chan r_done    = [0] of { bit };
chan e_done    = [0] of { bit };

```
We append an _r_ to channel names that transfer data for reindeer, and an _e_ to those for elves. Below are the roles of the main channels:

- _r_arrive / e_arrive_: These rendezvous channels represent how reindeer and elves enter their respective rooms. For example, a reindeer enters the reindeer room by communicating with the reindeer room process over _r_arrive_. When a reindeer or an elf attempts to enter, it blocks until the room is ready to accept it. If the room is full, it simply stops accepting arrivals, and new arrivals wait.


- _r_release / e_release_: These channels represent how a room releases a group. For example, after Santa finishes delivering toys, the reindeer room releases the reindeer by sending on _r_release_; each reindeer process synchronizes with the room on this channel to leave. Because _r_release_ is a rendezvous channel, the room cannot race ahead, it must wait for each reindeer to actually leave before proceeding.


- _r_done / e_done_: These channels represent how Santa signals “I have completed the task.” The room waits for this signal before releasing anyone. For example, after Santa finishes delivering toys, he signals the reindeer room by sending on _r_done_. Without this signal, the room would have no way to know when Santa is done.

Now let’s look at the reindeer and elf processes, which are quite similar. We’ll start with the reindeer process:

```promela
active [9] proctype Reindeer()
{
    do
    :: r_arrive ! 1;     // enter the room
       r_release ? 1;    // wait to be released
    od
}
```

Because of the _active [9]_ keyword, SPIN starts nine reindeer processes running concurrently in the initial system state. The do construct introduces a repetition structure whose alternatives are written using double colons (::). The first statement following each option is called its guard. An option is executable if its guard is executable.

If more than one option is executable, SPIN selects one nondeterministically. If none of the options are executable, the loop blocks.

A send operation consists of a channel variable followed by an exclamation mark (!) and a sequence of expressions whose number and types must match the channel’s message type. A receive operation, by contrast, consists of a channel variable followed by a question mark (?) and a sequence of variables into which the message is received.

For example, we can declare a rendezvous channel named _request_ that carries messages of type _byte_ as follows:
```
chan request = [0] of { byte };
```
We can send data on this channel as:

```promela
request ! 1
```
and receive data as:

```promela
byte client;
request ? client
```

Returning to the reindeer process, the statement _r_arrive ! 1_ indicates that a reindeer is attempting to enter the reindeer room by sending on the _r_arrive_ channel. Because _r_arrive_ is a rendezvous channel, the reindeer blocks until the room is ready to accept it. After entering the room, the reindeer waits to be released by blocking on the _r_release_ channel.

The code for the elf process is similar:

```promela
active [10] proctype Elf()
{
    do 
    :: e_arrive ! 1;
       e_release ? 1;
    od
}
```

The room is where marshalling and synchronization take place. Let’s now look at the reindeer room.

```
active proctype RoomReindeer()
{
    byte waiting = 0;
    byte i = 0;

    do
    /* Collect reindeer until the group is complete (entrance open iff waiting < 9) */
    :: (waiting < NUM_REINDEER) ->
        atomic {
            r_arrive ? 1;
            waiting++;
            reindeer_waiting = waiting;
            if
            :: (waiting == NUM_REINDEER) -> r_request = 1
            :: else -> skip
            fi
        }

    /* Group complete: wait for Santa, then release all 9 and reset */
    :: (waiting == NUM_REINDEER) ->
        atomic { r_done ? 1; i = 0; }

        do
        :: (i < NUM_REINDEER) ->
            atomic { r_release ! 1; i++; }
        :: else -> break
        od;

        atomic {
            waiting = 0;
            reindeer_waiting = 0;
        }
    od
}
```

The repetition structure has two alternatives. When _waiting_ is less than _NUM_REINDEER_, the room accepts arriving reindeer and increments the counter. If the arrival completes the group, the room sets _r_request = 1_ to notify Santa. 

When _waiting_ equals _NUM_REINDEER_, the room stops accepting arrivals and waits for Santa to signal completion on _r_done_. After receiving this signal, the room releases the reindeer via _r_release_ and resets its state to accept new arrivals.

The logic for the elf room follows the same pattern and is omitted here for brevity.

The Santa process models two kinds of work: toy delivery with the reindeer and consultation with the elves. Its main loop contains two guarded alternatives.

In the first alternative, guarded by _r_request_, Santa performs the toy delivery action, clears _r_request_, and signals completion to the reindeer room by sending on _r_done_. The corresponding code is shown below.


```
:: r_request ->
        atomic {
            delivering = true;
            r_request = 0;   /* claim the request */
        }

        /* Deliver toys (abstracted). */
        delivering = false;

        /* Notify the Room that delivery is complete; it can release the reindeer. */
        r_done ! 1;
```

The second alternative is guarded by _!r_request && e_request_. This guard enforces the priority rule: Santa consults with the elves only if no reindeer group is waiting and a group of three elves has formed. Santa then performs the consultation and signals completion to the elf room by sending on _e_done_.

```promela
:: (!r_request && e_request) ->
        atomic {
            consulting = true;
            e_request = 0;   /* claim the request */
        }

        /* Consult (abstracted). */
        consulting = false;

        /* Notify the Room that consultation is complete; it can release the elves. */
        e_done ! 1;
```

How do we validate that the model is correct? 

A model checker explores the state space of the model, that is, all the different states the system can possibly be in as it runs. Starting from the initial state, it systematically explores every reachable state by considering all possible ways the processes might interleave. Given a set of correctness properties, the model checker then checks whether each property holds in all states. If a property does not hold, SPIN produces a concrete counterexample trace showing exactly how the failure occurs.

We can specify two kinds of correctness properties: safety properties and liveness properties. Let’s look at them one by one.

Safety properties assert that nothing bad ever happens. We use three safety properties in this model.
First, we ensure that Santa always delivers toys with exactly nine reindeer, that is, a full group of nine reindeer must be present in the room. We specify this as:

```
ltl safety_delivery { [] (delivering -> reindeer_waiting == 9) }
```

In SPIN, [] is the always operator. This above property states that, at all times, if Santa is delivering toys, then the variable _reindeer_waiting_ must equal 9.

Second, we ensure that Santa always consults with exactly three elves. We specify this as:

```
ltl safety_consult { [] (consulting -> elf_waiting == 3) }
```

Finally, we specify a mutual-exclusion property to ensure that Santa never delivers toys and consults at the same time:

```
ltl mutex_santa { [] !(delivering && consulting) }
```

You might ask why is delivering or consulting true only briefly? Delivery and consultation are abstracted to single steps; the boolean flags exist to state and check properties.

Now let’s turn to the liveness property. Liveness properties state that something good eventually happens. The liveness property we specify is: always, if a request is pending, then eventually Santa will either be delivering toys or consulting with elves. This rules out executions where requests remain pending forever without Santa ever doing work.

We specify this property as:

```
ltl live_progress { [] ((r_request || e_request) -> <> (delivering || consulting)) }
```

One important role of a liveness property is to avoid vacuous correctness. For example, a safety property such as `[] (delivering -> reindeer_waiting == 9)` can hold even if delivering is never true. In contrast, the liveness property ensures that the system eventually makes progress, that Santa eventually performs work when there is a pending request.

When we run the model in SPIN, all safety and liveness properties pass. SPIN explores ~5 millions states while checking the _safety_delivery_ and _mutex_santa_ and ~7 million states while checking the liveness property and finds no counterexamples. It also reports no deadlocks. Both the model and the instructions for running it are available in the repository [^2].

Let’s now look at an example execution to see how these pieces fit together.

<img loading="lazy" src="{{ site.baseurl }}/images/2025-12-23-how-to-help-santa-concurrently/fourth.png" alt="Santa claus sequence" />

Time flows from top to bottom. When the ninth reindeer arrives, the reindeer room sets _r_request_ and closes the room to further arrivals. While _r_request_ remains true, Santa serves the reindeer, even if elves arrive, because the elf branch is disabled by the guard _!r_request_.

After Santa signals completion by sending on _r_done_, the room releases all nine reindeer via rendezvous. When a group of three elves later forms, the model sets _e_request_. Since _r_request_ is now false, Santa serves the elves.

To transform this model into a computer program, I thought I would try implementing it in Go. I do not know Go particularly well, so I collaborated with an AI to translate the Promela design into a simple, idiomatic Go implementation. I reviewed the code to keep it faithful to the model, but any mistakes are mine.

_Please keep in mind that I’m only human, and there’s a chance this post contains errors. If you notice anything off, I’d appreciate a correction. Please feel free to [send me an email](mailto:waqas.younas@gmail.com)._

## Endnotes:

[^1]: Ben-Ari, M. "How to solve the Santa Claus problem." Concurrency: Practice and Experience, 1998. 
[^2]: Source code: https://github.com/wyounas/model-checking/blob/main/puzzles/