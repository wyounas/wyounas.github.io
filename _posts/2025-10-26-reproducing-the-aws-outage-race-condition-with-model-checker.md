---
layout: post
title:  "Reproducing the AWS Outage Race Condition with a Model Checker"
date: 2025-10-26
categories: aws, concurrency
---

AWS published a post-mortem about a recent outage [^postmortem]. Big systems like theirs are complicated mixes of software and hardware, and things sometimes go wrong. 

The post-mortem mentioned a race condition, which caught my eye. I don’t know the details of AWS’s internal setup, but using the information in the post-mortem and a few assumptions, we can try to reproduce a simplified version of the problem. As a small learning exercise, we’ll use a model checker to see how such a race could happen. Formal verification can’t prevent every failure, but it helps us think more clearly before coding and catch subtle concurrency bugs that ordinary testing might miss. For this, we’ll use the Spin model checker, which uses the Promela language.

There’s a lot of detail in the post-mortem, but for simplicity we’ll focus only on the race-condition aspect. The incident was triggered by a defect in DynamoDB’s automated DNS management system. The components of this system involved in the incident were the DNS Planner, DNS Enactor, and Amazon Route 53 service.

The DNS Planner creates DNS plans, and the DNS Enactors look for new plans and apply them to the Amazon Route 53 service. Three Enactors operate independently.

Here is an illustration showing these components (if the images appear small, please open them in a new browser tab):

<img loading="lazy" src="{{ site.baseurl }}/images/2025-10-26-aws-outage-race-conditions/aws-dns.png" alt="aws dns management system"  />

My understanding of how the DNS Enactor works is as follows: it picks up the latest plan and, before applying it, performs a one-time check to ensure the plan is newer than the previously applied one. It then applies the plan and invokes a clean-up process. During the clean-up, it identifies plans older than the one it just applied and deletes them.

Using the details from the incident report, we could sketch an interleaving that could explain the race condition. Two Enactors running side by side: one applies a new plan and starts cleaning up, while the other, running just a little behind, applies an older plan, making it an active one. When the Enactor 2 finishes its cleanup, it deletes that plan, and the DNS entries disappear. Here’s what that sequence looks like:

<img loading="lazy" src="{{ site.baseurl }}/images/2025-10-26-aws-outage-race-conditions/race-condition-interleaving.png" alt="aws dns race condition"  />


Let's try to uncover this interleaving using a model checker. We’ll create a DNS Planner process that produces plans, and a DNS Enactor process that picks them up. The Enactor will check whether the plan it’s about to apply is newer than the previous one, update the state of certain variables to simulate changes in Route 53, and finally clean up the older plans.

In our simplified model, we’ll run one DNS Planner process and two DNS Enactor processes. (AWS appears to run three across zones; we abstract that detail here.) The Planner generates plans, and through Promela channels, these plans are sent to the Enactors for processing.

Inside each DNS Enactor, we track the key aspects of system state. The Enactor keeps the current plan in _current_plan_, and it represents DNS health using _dns_valid_. It also records the highest plan applied so far in _highest_plan_applied_. To simulate the deletion of an active plan, the Enactor’s clean-up process checks whether _current_plan_ equals the plan being deleted. If it does, we simulate DNS deletion by setting _dns_valid_ to false.

Here’s the code for the DNS Planner:

```
active proctype Planner() {
    byte plan = 1;
    
    do
    :: (plan <= MAX_PLAN) ->
        latest_plan = plan;
        plan_channel ! plan; 
        printf("Planner: Generated Plan v%d\n", plan);
        plan++;
    :: (plan > MAX_PLAN) -> break;
    od;
    
    printf("Planner: Completed\n");
}
```

It creates plans and sends them over a channel to be picked up later by the DNS Enactor.

We start two DNS Enactor processes by specifying the number of enactors after the _active_ keyword.

```
active [NUM_ENACTORS] proctype Enactor() 
```

The DNS Enactor waits for plans and receives them (? indicates receiving from a channel). It then performs a staleness check, updates the state of certain variables to simulate changes in Route 53, and finally cleans up the older plans.

```
:: plan_channel ? my_plan ->
    snapshot_current = current_plan;

    // staleness check    
    if
    :: (my_plan > snapshot_current || snapshot_current == 0) ->

        if
            :: !plan_deleted[my_plan] ->
                /* Apply the plan to Route53 */
                
                current_plan = my_plan;
                dns_valid = true;
                initialized = true;
               /* Track highest plan applied for regression detection */
                if 
                :: (my_plan > highest_plan_applied) ->
                    highest_plan_applied = my_plan;
                fi 
            
            // runs the clean-up process (omitted for brevity, included in the 
            // code linked below)
        fi
    fi

```


How do we discover the race condition? The idea is this: we express as an invariant what must always be true of the system, and then ask the model checker to confirm that it holds in every possible state. In this case, our invariant states that the DNS should never be deleted once a newer plan has been applied. (With more information about the real system, we could refine this rule further.)

We specify this invariant formally as follows:


```
/*

A quick note on some of the keywords used in the invariant below:

ltl - keyword that declares a temporal property to verify (ltl: linear temporal logic lets you specify properties about all possible executions of your program.)

[] - "always" operator (this must be true at every step forever)

-> - "implies" (if left side is true, then right side must be true)

*/

ltl no_dns_deletion_on_regression {
    [] ( (initialized && highest_plan_applied > current_plan 
            && current_plan > 0) -> dns_valid )
}



```

When we run the model checker, it finds a violation of the invariant. Spin reports one error and writes a trail file that shows, step by step, how the system reached the bad state.
```

$ spin -a aws-dns-race.pml
$ gcc -O2 -o pan pan.c                                                       
$ ./pan -a -N no_dns_deletion_on_regression  

pan: wrote aws-dns-race.pml.trail

(Spin Version 6.5.2 -- 6 December 2019)

State-vector 64 byte, depth reached 285, errors: 1
    23201 states, stored
    11239 states, matched
    34440 transitions (= stored+matched)
  (truncated for brevity....)

```

The trail file tells the story of the race. Two Enactors work in parallel: one moves ahead with plan 4 and begins to clean up, while the other lags behind and applies plan 3 (an older plan). When the Enactor 2 circles back to remove “old” data, it ends up deleting the very plan now in use, bringing the DNS down.

Here’s an illustration of the interleaving reconstructed from the trail:

<img loading="lazy" src="{{ site.baseurl }}/images/2025-10-26-aws-outage-race-conditions/race-condition-spin-inteleaving.jpg" alt="aws dns race condition simulated by spin" />

To fix the code, we execute the problematic statements atomically. You can find both versions of the code, the one with the race and the fixed one, along with the interleaving trail in the accompanying repository [^code]. I’ve included detailed comments to make it self-explanatory, as well as instructions on how to run the model and explore the trail.

Some of the assumptions in this model are necessarily simplified, since I don’t have access to AWS’s internal design details. Without that context, there will naturally be gaps between this abstraction and the real system. This model was created in a short time frame purely for educational purposes. With more time and context, one could certainly build a more accurate and refined version. 

_Please keep in mind that I’m only human, and there’s a chance this post contains errors. If you notice anything off, I’d appreciate a correction. Please feel free to [send me an email](mailto:waqas.younas@gmail.com)._

## Endnotes:

[^postmortem]: [AWS Post-Incident Summary — October 2025 Outage](https://aws.amazon.com/message/101925/)

[^code]: [Source code repository](https://github.com/wyounas/aws-dns-outage-oct-2025-modeling/blob/main/aws-dns-race.pml)
