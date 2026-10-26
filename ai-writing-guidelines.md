# AI Writing Guidelines

Big sins of AI writing:

1. Saying the obvious

AI often writes sentences that sound true but add no value.

Bad:

> This incident shows that distributed systems can be complex.

Better:

> This incident shows how one missing concurrency limit can turn a small internal traffic pattern into resource exhaustion.

2. Abstract nouns everywhere

Words like "robustness," "resilience," "correctness," "scalability," "efficiency," and "reliability" are useful, but AI often uses them without concrete detail.

Bad:

> This improves system resilience.

Better:

> This lets the service recover after load drops instead of staying stuck in the loaded state forever.

3. Fake balance

AI likes phrases such as "not only X, but also Y," "while this is useful, it is important to note," and "in today's complex landscape."

Usually, delete them. Say the actual thing.

4. Over-explaining transitions

Bad:

> Now that we have discussed X, let us turn to Y.

Better:

> Next, let's model port exhaustion.

5. Generic praise

Bad:

> This is a great example of engineering excellence.

Better:

> Publishing this post-mortem gives other teams a concrete failure pattern to study.

6. Too many hedges

AI overuses "can," "may," "might," "potentially," "often," and "in some cases."

Bad:

> This can potentially help teams avoid some possible issues.

Better:

> This helps teams catch missing limits before production traffic finds them.

7. No point of view

AI writing is often careful but bloodless. Technical writing still needs a claim.

Weak:

> Model checking can be useful for incidents.

Strong:

> A model is useful here because it turns a vague lesson, "add limits," into a property Spin can falsify.

8. Fancy words for simple actions

Bad:

> The system initiates request processing.

Better:

> The service starts work.

9. Repetition with synonyms

Bad:

> The system became overloaded, under stress, and unable to handle the increased pressure.

Better:

> The system became overloaded.

10. Grand conclusions

Bad:

> This demonstrates the importance of formal methods in building the systems of tomorrow.

Better:

> The model does not prove the production system correct, but it catches the missing limit that caused this class of failure.

## How To Avoid It

- Use concrete nouns: `goroutine`, `port`, `idle pool`, `TIME_WAIT`, `work_inflight`.
- Prefer verbs over concepts: blocks, opens, closes, retries, queues.
- Make every paragraph answer: "What changed in the system?"
- Delete sentences that only sound wise.
- Replace "can help improve resilience" with the exact failure it prevents.
- Keep one idea per sentence.
- Use small words unless the technical term is the precise word.
- Let examples carry the explanation.
- Read each paragraph and ask: "Could this appear in any engineering blog post?" If yes, make it more specific.

For the Bluesky post, the main risk is not that it sounds fake. It has real technical substance. The risk is that some sections repeat the same setup and use abstract phrases like "correctness properties," "verification purposes," and "resilient systems" more than needed. Keep the Spin details, but make the prose more direct and concrete.
