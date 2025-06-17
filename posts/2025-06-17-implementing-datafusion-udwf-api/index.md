---
title: "Forged in Practice: How Implementing Window Functions Refined DataFusion's API"
date: 2025-06-17
summary: >
  Designing a good API is hard, but the real test comes when it's used to solve complex problems. This post is a deep-dive
  case study into the evolution of Apache DataFusion's WindowUDF trait. We'll explore how the initial, simple abstraction
  was stress-tested by converting built-in functions, and how the practical challenges of implementing LEAD and LAG
  forced the API to become more powerful and expressive.
layout: layouts/post.njk
draft: true
---

Designing a library's core abstractions is one of the most challenging parts of software engineering. It's a balancing
act between simplicity, power, and future-proofing. But as we found during a major refactor in Apache DataFusion, the
best APIs aren't just designed—they're forged and refined in the fire of real-world implementation.

This post is a case study of that process. We'll trace the evolution of the `WindowUDFImpl` trait, showing how its
initial design was stress-tested and ultimately improved by the practical requirements of converting complex window
functions like `LEAD` and `LAG`.

### The Starting Point: A Clean but Simple Abstraction

The DataFusion community had a clear strategic goal: unify the two separate function interfaces (`BuiltInWindowFunction`
and `WindowUDF`) into a single, trait-based API. Building on the success of a similar refactor for scalar functions, the
initial `WindowUDFImpl` trait was designed for clarity and simplicity, and it served as a solid architectural blueprint.

*(Here, you can show the initial, simpler version of the `WindowUDFImpl` trait, highlighting methods like `return_type`
and `nullable`.)*

### The Challenge: Applying the Blueprint to a New Domain

With a proven pattern from the scalar function refactor, the next challenge was to adapt and apply this blueprint to the
more complex domain of window functions. My work began here: taking the core ideas and being the first to prove them out
on the windowing engine, which had its own unique requirements for stateful evaluation and optimizer integration.

### The First Test: A Successful `row_number` Conversion

The first candidate for this new approach was `row_number`. As a simple function with no arguments and a fixed return
type, it was the perfect vehicle to validate that the core pattern was sound. The conversion was a success and
demonstrated the viability of the unification effort.

*(Here, you can briefly show the clean implementation of `WindowUDFImpl` for `row_number` and how it mapped well to the
initial trait design.)*

### Hitting the Limits: The Challenge of `LEAD` and `LAG`

The real test came when we moved on to more complex functions. The simple API immediately presented several difficult
questions that it was not yet equipped to answer:

1. **How can the query optimizer know that `LEAD` is the semantic inverse of `LAG`?** This is crucial for performance.
2. **How can a function's return type be determined when it depends on multiple, potentially `NULL`, arguments?**
3. **How can the stateful `PartitionEvaluator` access the literal values of its arguments to configure its behavior?**

It became clear that to implement these functions correctly, the API itself had to evolve.

### Reforging the API: The Introduction of `field`, `reverse_expr`, and `PartitionEvaluatorArgs`

This led to a series of patches that fundamentally improved the `WindowUDFImpl` trait, driven directly by the problems
we discovered during implementation.

*(This is your core technical section. Use your patches to show the "before" and "after" for each of these API
additions, explaining what problem each one solved.)*

* **The `field` Method:** Show how this replaced `return_type` and `nullable`, giving the implementer more context.
* **The `reverse_expr` Method:** Explain how this provided a formal hook for the query optimizer.
* **The `PartitionEvaluatorArgs` struct:** Detail how this gave the stateful evaluator crucial information from the
  physical plan.

### The Payoff: An Elegant Implementation

With the new, more expressive API in place, the implementation of `LEAD` and `LAG` became straightforward and robust,
proving the value of the iterative design process.

*(Here, you can show snippets from the final `lead_lag.rs` implementation, highlighting how it elegantly uses the new
API methods.)*

### Lessons Learned on API Design and Community

This journey offers several valuable lessons that extend beyond just DataFusion:

* **Start with a simple abstraction, but stress-test it immediately** with your most complex use cases.
* **Treat implementation feedback as a core part of the design process.** When an abstraction feels clumsy or limiting,
  it's an opportunity to improve it.
* **Good architecture enables community.** A well-designed, repeatable pattern doesn't just make the original author's
  life easier; it creates a "paved path" for new contributors. The clarity of the final `WindowUDF` trait made it
  possible to file well-defined "good first issues," broadening the base of community involvement and turning a large
  solo effort into a parallelizable community one.
