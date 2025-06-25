---
title: "The Clever Design in Apache Parquet For Storing Nested Data Efficiently In Columns"
date: 2025-06-24
summary: >
  Parquet uses the Dremel representation. Under the hood it derives two metadata columns for each value. This post
  is not simply about how to compute the metadata. It is complicated, but mechanical once you understand the core
  algorithm. The interesting part involves the challenges which are unique to nested data structures, and marvel
  at the elegance of the Dremel representation.
layout: layouts/post.njk
draft: true
---

Modern state of the art OLAP query execution engines can execute medium-high selectivity queries in under a second
on petabyte-scale(trillion rows) data. The foundation of this high-performance is columnar storage. Here is a
surprising truth: the performance is virtually identical whether you are querying nested data structures in Apache
Parquet or flat, relational tables. Many, mistakenly believe that nested data structures inherently leads to slower
query performance. This belief probably stems from a comparison with OLTP systems.

In the time before Parquet storing and querying nested data structures at scale was slow. The first system to fix
the performance gap was Google BigQuery powered by the Dremel engine. Parquet adopted the Dremel representation for
nested data structures in its own file format. This was not done as an afterthought. So Parquet came with ground up
support for storing and querying both nested data structures and relational, flat data.

The inner workings of the Dremel representation is complicated, but mechanical. There are many high quality posts
written about it. A curated set of links is available at the end of this blog post. So I am going to leave out the
how to compute the Dremel encoding for nested data structures out of this post.

Instead, the main thrust of this post is explore the inherent challenges with nested data structures that makes it
so hard to store them efficiently and improving query performance. This will help us better understand the ingenious
design choices in the Dremel representation.

## What is a Schema?

We are storing bytes of ones and zeroes to disk. Now you want to read it back. But values of different types can end
up with the same byte representation on disk. So reading back the data depends on how you interpret the bytes. To
read the data that you wrote to disk, and read it back in its original form you need metadata which describes the
logical type of the value in physical storage. Without this metadata context bytes are ambiguous.

In database storage at a minimum three metadata values are required to describe any value.

1. The field (or column) name
2. The data type
3. Is this value nullable?

A relation (or table) is a collection of fields (or columns).

A schema describes a relation.

## How to define a Nested Schema?

The nesting levels are primarily described with two data types - a struct and a list.

The struct data type is a collection of fields. To have nesting it should contain at least one field which is a
struct or a list.

A list data type is a container which also has an inner field to describe the elements of the list. If the inner
field happens to be a struct or a list data type then another level of nesting is created.

This is easier to comprehend with a concrete example.

### Example: data structure with a single level of nesting

```rust
struct Contact {
  name: option<string>,
  phones: option<vec<Phone>>,  // <-- this field introduces a nesting level
}

struct Phone {
  number: option<string>,
  phone_type: option<PhoneType>,
}

enum PhoneType {
  Home, // "Home"
  Work, // "Work"
}
```

### Example (continued): Schema Definition

We will now define the schema using Apache Arrow. Let us build the schema bottom-up so that we can compose the `Contact`
struct from its member fields.

```rust
// struct Phone {
//   number: option<string>,
//   phone_type: option<Phone>,
// }

let phone_type_field = Field::new("phone_type", Datatype::Utf8, true);
let phone_number_field = Field::new("number", Datatype::Utf8, true);

let phone_struct = Datatype::Struct (vec![phone_type_field, phone_number_field]);
```

With `phone_struct` data type we can now define the field for list items.

```rust
// phones: option<vec<Phone>>

let phone_list_item = Field::new("item", phone_struct, true);

// @formatter:off
let phones_list_field = Field::new(
    "phones",
    Datatype::list(Arc::new(phone_list_item)),
    true
);
// @formatter:on
```

We now have all the pieces in places to compose the schema for `Contact`.

```rust
// struct Contact {
//   name: option<string>,
//   phones: option<vec<Phone>>,
// }

let name_field = Field::new("name", Datatype::Utf8, true);
// phones_list_field (see previous code block)

let schema = Schema::new(vec![name_field, phones_list_field]);
```

## Problem 1: One Schema, Many Possible Structures

Even with a straightforward schema like `Contact`, the combination of optional fields and nested collections leads
to rich variety of valid data states. These instances can exhibit significant structural variations based on the
presence or absence of data in their fields. This makes most values potentially unique in structure despite adhering
to the same base schema.

These examples demonstrate the different structural variations that are all valid instances of the `Contact` schema.

```rust
vec![
  // 0: Has a name and 2 phones
  Contact {
    name: Some("Alice".to_string()),
    phones: Some(vec![
      Phone {
        number: Some("555-1234".to_string()),
        phone_type: Some(PhoneType::Home),
      },
      Phone {
        number: Some("555-5678".to_string()),
        phone_type: Some(PhoneType::Work),
      },
    ]),
  },
  // 1: phones is not present
  Contact {
    name: Some("Bob".to_string()),
    phones: None,
  },
  // 2: phones is present, but list is empty
  Contact {
    name: Some("Charlie".to_string()),
    phones: Some(Vec::<Phone>::new()),
  },
  // 3: No name, 1 phone but number is not present
  Contact {
    name: None,
    phones: Some(vec![
      Phone {
        number: None,
        phone_type: Some(PhoneType::Home),
      },
    ]),
  },
];
```

The problem compounds fast for real-world nested data structures with many optional and list
fields.

## Problem 2: Missing Values

## Problem 3: Different Lists, Identical Representation

## Problem 4: Empty Lists

A list instance where the outer list field is nullable can be in three states: `[1, 2, 3]`, `[]` and `NULL`. A
non-nullable list on the other hand has two valid states: `[1, 2, 3]` and `[]`. The list can contain null values but
that depends on the nullability of the list elements datatype defined within the list.

## Problem 5: Sparse Values, Storage inefficiency

## Problem 6: Partial access (read a single path)

## Problem 7: Predicate Pruning/Skipping

## Shifting burden of normalization in storage, join in querying to storage layer

