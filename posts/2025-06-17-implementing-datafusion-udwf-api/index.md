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

The DataFusion community had a clear goal: unify the two separate function interfaces (`BuiltInWindowFunction` and
`WindowUDF`) into a single, trait-based API. Building on the success of a similar refactor for scalar functions, the
initial `WindowUDFImpl` trait was designed for clarity and simplicity.

Its early form looked something like this, with separate methods for determining the return type and nullability:

```rust
// Simplified for illustration
pub trait WindowUDFImpl {
  fn name(&self) -> &str;
  fn signature(&self) -> &Signature;

  // How the return type was determined
  fn return_type(&self, arg_types: &[DataType]) -> Result<DataType>;

  // How nullability was determined
  fn nullable(&self) -> bool { true }

  // How the stateful logic was created
  fn partition_evaluator(&self) -> Result<Box<dyn PartitionEvaluator>>;
}
```

This was a great starting point, and it worked perfectly for the first test case.

# The First Test: A Successful row_number Conversion

The first function to be converted was row_number. As a simple function with no arguments and a fixed return type, it
was a perfect fit for the initial API. The conversion from the built-in enum to the new trait was straightforward and
validated that the core abstraction was sound for simple cases.

(Here, you can briefly show the clean implementation of WindowUDFImpl for row_number, highlighting how easily it mapped
to the simple trait.)

# Hitting the Limits: The Challenge of LEAD and LAG

The real test came when we moved on to more complex functions like LEAD and LAG. The simple API immediately presented
several difficult questions:

How can the query optimizer know that LEAD is the semantic inverse of LAG? Without this knowledge, DataFusion couldn't
apply critical optimizations, like reusing a single sort operation for queries containing both functions.
How can the function correctly determine its return type when it depends on multiple arguments? For LEAD(col, offset,
default_value), the return type should be the type of col, but what if col is NULL? The type should then be inferred
from default_value. The simple return_type method didn't have enough context.
How can the stateful PartitionEvaluator get access to the literal values of arguments like offset and default_value? It
needed these to configure its behavior, but the initial API didn't provide them.
It became clear that to implement these functions correctly and elegantly, the API itself had to evolve.

# Reforging the API: Forging a More Powerful Trait

This led to a series of patches that fundamentally improved the WindowUDFImpl trait, driven directly by the problems we
discovered.

Solution 1: The field Method
To solve the context problem for return types, we replaced return_type() and nullable() with a single, more powerful
field() method.

(Here, you can show the diff from 0001-Add-field-trait-method-to-WindowUDFImpl-remove-retur.patch and explain that
passing the WindowUDFFieldArgs struct gives the implementer all the context needed to determine the final Field—name,
type, and nullability—in one go.)

Solution 2: The reverse_expr Method
To inform the optimizer about function inverses, we added the reverse_expr() method.

(Show the reverse_expr method from 0001-Adds-WindowUDFImpl-reverse_expr-trait-method-Support.patch. Explain how
returning ReversedUDWF::Reversed(lag_udwf()) from within lead creates a formal, machine-readable link between the two
functions.)

Solution 3: The PartitionEvaluatorArgs
Finally, to give the stateful evaluator access to its arguments, we enhanced the partition_evaluator method signature.

(Show the change from 0001-Add-PartitionEvaluatorArgs-to-WindowUDFImpl-partitio.patch. Explain that passing
PartitionEvaluatorArgs allows the evaluator to inspect the physical expressions and configure itself accordingly.)

# The Payoff: An Elegant Implementation

With this new, more expressive API in place, the implementation of LEAD and LAG became straightforward and robust.

(Here, you can show snippets from the final lead_lag.rs implementation from
0001-Convert-BuiltInWindowFunction-Lead-Lag-to-a-user-def.patch. Contrast how clean it is using the new API, compared to
how hacked-together it would have been with the old one.)

# Lessons Learned on API Design

This journey through DataFusion's window function refactor offers several valuable lessons on API design:

- Design for real use cases. A simple API is a good start, but it must be validated against the most complex
  problems you
  intend to solve.
- The implementation is the ultimate stress test. Theoretical API design is no substitute for the feedback you get from
  actually building with it.
- Don't be afraid to evolve your core abstractions. When implementation reveals a weakness in your API, it's a sign of
  progress. Improving the trait is often better than building hacks on top of it.
- The result of this feedback loop is a more resilient, expressive, and ergonomic system for all future developers.
