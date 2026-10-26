- "One possible learning path from such failures"

  Why: This sounds like a generic AI transition. It introduces the next idea with an abstract phrase instead of saying the direct thing: model the failure.

  Fix: "I started by modeling the failure."

- "A model can help us test the correct properties and invariants we expect the system to satisfy."

  Why: This is broad and abstract. "can help us," "correct properties," "invariants," and "satisfy" are all common AI-style filler when the sentence has not yet named a concrete check.

  Fix: "In this model, the checks are direct: `work_inflight` must stay below the limit, ports must not be exhausted, and a loaded service must eventually recover."

- "Writing the model also forces us to reason about correctness of our system."

  Why: This is a generic formal-methods claim. It sounds true, but it does not say what the model forced the author to notice.

  Fix: "Writing the model makes the missing guard visible: `Service` can start work without checking `work_inflight < work_limit`."

- "From the incident report, we can extract bounded concurrency as a key correctness property of the system in question."

  Why: "extract," "key correctness property," and "system in question" make a simple idea sound more formal than needed.

  Fix: "The incident points to bounded concurrency as the first property to check."

- "The model would also show that the modeled system could not recover on its own."

  Why: This has AI-like repetition around "model/modeled" and a vague phrase, "recover on its own," before the concrete failure is named.

  Fix: "It also checks whether a loaded service can get back below the load threshold."

- "We explain this more below, including a visualization showing precisely how we're able to reproduce the problem using our model."

  Why: This is a meta-transition. AI writing often announces what the next section will do instead of moving straight to the evidence.

  Fix: "The counterexample below shows the exact interleaving."

- "Such models, when run with model checkers, could also be useful for verification purposes and help ship systems that work."

  Why: This is very AI-like: "could also be useful," "verification purposes," and "help ship systems that work" are broad, hedged, and generic.

  Fix: "A small model lets a team test the design before production traffic does."

- "Not to say they always work and never break; sometimes they may fail but you learn from your mistakes and do more verification and such verification can help make your systems more robust, more resilient."

  Why: This has the classic AI caveat pattern. It softens the claim several times and ends with generic values: "robust" and "resilient."

  Fix: "A passing model is not a guarantee. It is a cheap way to catch specific mistakes, such as a missing concurrency guard or a recovery rule that never runs."

- "you could use the model as a validation artifact to ensure that the implementation not only implements those invariants but is also faithful to the model."

  Why: "validation artifact," "ensure," and "not only...but also" are strong AI tells. The sentence turns a concrete use of the model into inflated process language.

  Fix: "If AI writes the implementation, the model gives you something concrete to test against: the code should enforce the same limits and recovery rules."
