---
title: "The Trait Transformation: Unpacking DataFusion's Evolved Window Function API"
date: 2025-06-18
summary: >
  Explore DataFusion's journey to a modern, trait-based window function API. Learn how moving beyond old, imperative
  methods and confronting the complexities of functions like LEAD and LAG led to a more modular, consistent, and
  extensible query engine. Discover how Rust's powerful traits were forged in practice to simplify development and
  empower a vibrant open-source community.
layout: layouts/post.njk
draft: true
---

APIs aren't just designed in a vacuum; they're forged and refined through real-world implementation. This post will be a
deep dive into how the practical challenges of unifying DataFusion's window function interface and implementing complex
functions like `LEAD` and `LAG` drove the evolution of the `WindowUDFImpl` trait, resulting in a more modular,
extensible, and developer-friendly query engine.

DataFusion is an extensible query engine, and its `WindowUDF` API has undergone a significant evolution. Let's set the
stage for a case study on this transformation.

---

### The Strategic Imperative: Why Unify Functions in DataFusion?

This refactor wasn't an isolated event; it was part of a larger strategic effort across DataFusion to unify all function
interfaces (scalar, aggregate, and now window).

Here were the core motivations, addressing previous pain points and incorporating new details:

* **Modular Packages & Reduced Footprint:** Previously, a growing number of built-in functions increased the core binary
  size. The vision was to split functions into optional packages, allowing users to pull in only what they needed.
* **From Awkward Imperative to Idiomatic Rust Traits:** The old way of defining user-defined functions (like `ScalarUDF`
  and `WindowUDF` using complex `Arc<dyn Fn(...)>` function pointers) was awkward, cumbersome, and hard to extend in
  backwards-compatible ways. The goal was to shift to a more declarative, trait-based model (e.g., `WindowUDFImpl`),
  which is an idiomatic Rust pattern for extensibility and cleaner API design.
* **Unified Code Paths & Consistent Experience:** We highlighted the inefficiency and inconsistency of having separate "
  built-in" and "user-defined" function implementations. Unifying them under a single `WindowUDFImpl` trait ensures all
  functions (internal or external) use the same code paths, leading to simpler maintenance, better consistency, and
  future optimization opportunities.
* **Enhanced Extensibility & Contribution:** The enum-based approach for built-ins required modifying massive central
  functions with pattern matching for every new function. A trait-based model inherently promotes extensibility, making
  it easier for new contributors to add or extend functionality without touching core logic.

The initial, simpler version of the `WindowUDFImpl` trait had basic methods like `return_type` and `nullable`.

---

### Proving the Pattern: The `row_number` Pathfinder

`row_number` was chosen as the initial candidate for conversion to the new `WindowUDFImpl` trait because it's a simple
function (no arguments, fixed return type) perfect for validating the core pattern. The initial success with
`row_number` under the new trait demonstrated that the basic unification strategy was sound.

---

### The Crucible: When `LEAD` and `LAG` Stress-Tested the API

The real challenge emerged with `LEAD` and `LAG`, complex functions that exposed the limitations of the initial, simple
`WindowUDFImpl` design.

Here are the specific problems uncovered, along with explanations of "why":

* **Optimizer's Dilemma (Reversal):** "How can the query optimizer know that `LEAD` is the semantic inverse of `LAG`?"
  This relationship is vital for performance optimizations when evaluating window frames in reverse (e.g.,
  `UNBOUNDED PRECEDING`). The initial API had no formal way to declare this.
* **Context-Dependent Return Types & Nullability:** "How can a function's return type be determined when it depends on
  multiple, potentially `NULL`, arguments?" `LEAD`/`LAG` can have optional offset and default values. The result's type
  and nullability might depend on the type of the default argument, even if the primary argument is `NULL`. The basic
  `return_type` and `nullable` methods lacked the necessary context.
* **Runtime Argument Access for Evaluators:** "How can the stateful `PartitionEvaluator` access the literal values of
  its arguments (like offset and default) to configure its behavior?" The runtime evaluation logic needs to know the
  actual offset and default values, not just their types, to perform the shift. The initial `partition_evaluator`
  signature provided no direct way to pass these runtime-evaluated arguments.

---

### Forging the API: Iterative Refinement Through Implementation

The challenges posed by `LEAD` and `LAG` directly led to specific, crucial additions and modifications to the
`WindowUDFImpl` trait. This was a true problem-driven evolution.

* **The `field` Method:**
  * **Problem Solved:** Context-dependent return types and nullability.
  * **How it Works:** It replaced `return_type` and `nullable`. `WindowUDFFieldArgs` provides the needed context (
    function name, input types) to precisely define the output `Field` (type + nullability).
* **The `reverse_expr` Method:**
  * **Problem Solved:** Informing the optimizer about semantic inverses for performance.
  * **How it Works:** It allows a UDWF to declare if it's `Identical`, `NotSupported` for reversal, or `Reversed` (
    mapping to another UDWF), enabling powerful optimizer transformations.
* **The `PartitionEvaluatorArgs` Struct:**
  * **Problem Solved:** Providing runtime context to the `PartitionEvaluator`.
  * **How it Works:** This new struct delivers essential runtime information (`input_exprs`, `input_types`,
    `is_reversed`, `ignore_nulls`) to the `PartitionEvaluator`, allowing it to dynamically configure its logic (e.g.,
    fetching actual offset and default values for `LEAD`/`LAG`).
* **The `simplify()` Method for Optimizer Rules:**
  * **Problem Solved:** Enabling UDWFs to provide their own optimizer simplification rules.
  * **Background:** This pattern was first established for `ScalarUDFImpl` (e.g., `arrow_cast` simplifying to
    `Expr::Cast`), allowing functions to "inline" their definitions or provide specialized rewrites. This capability was
    then adopted for `WindowUDFImpl`.
  * **How it Works:** `simplify()` allows UDWFs to return a `Rewritten` expression or `Original` arguments, influencing
    query planning.

The role of macros in developer experience is also important. While the core trait evolved, the macro patterns (
`get_or_init_udwf!`, `define_udwf_and_expr!`) were adopted from previous function unification epics (scalar and
aggregate functions). This ensures a consistent and declarative developer experience across DataFusion's function types,
making new UDWF creation intuitive, less verbose, and significantly less prone to common errors associated with manual
struct construction.

---

### The Payoff: Robustness, Performance, and a Paved Path

The refined API allowed for robust and elegant implementations of complex functions like `LEAD` and `LAG`. Features like
`reverse_expr` and `simplify` directly enable query optimizer gains, and the unified interface means both internal and
external functions now benefit from the same robust framework.

---

### Lessons Learned: Designing for Extensibility and Community

* **Iterative API Design is Key:** Start with a simple abstraction, but don't be afraid to rigorously stress-test it
  with your most complex, real-world use cases. Treat implementation challenges not as roadblocks, but as direct,
  invaluable feedback loops for API improvement. The most elegant and powerful APIs are often those that have been
  forged and refined through this iterative process.
* **Embrace Trait-Based Extensibility:** DataFusion's experience powerfully demonstrates how adopting Rust's trait
  pattern allows a project to move beyond rigid, enum-based systems. This shift creates a flexible, modular, and
  inherently future-proof architecture that can easily incorporate new functionality without disrupting existing code.
  It's a prime example of building for the unknown.
* **A Paved Path for Community:** A well-designed, consistent, and declarative API, supported by intuitive tools like
  the shared macro patterns, significantly lowers the barrier to entry for new contributors. What might otherwise be a
  daunting, monolithic refactor becomes a series of well-defined, "good first issues" that can be parallelized across a
  community. This not only accelerates development but also fosters a vibrant, engaged open-source ecosystem, turning
  complex solo efforts into collaborative community achievements.
