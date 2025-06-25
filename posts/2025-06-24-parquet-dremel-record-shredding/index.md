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

## Extract values from leaf node, now can't go back

The naive approach to storing nested data structure is a direct serialization, like JSON.stringify. It will preserve the
entire structural information (the objects, keys, and their nesting) along with the values. Reconstructing the original
value from it is as easy as reading the data and casting it in memory to its original type.

However, to store nested data for efficient OLAP querying, we want to save only the most essential parts of the data.
Which means -- just the values. We want to elide all structural information as well as the names of individual object
keys.

The genius of Dreml is that it found a way serialize deeply nested data structures by pulling in their values alone, and
storing them as a linear array. It intelligently encodes the structural information in this linear array, letting it
recreate the original data structure without losing fidelity.

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

The number of valid structures compounds fast for real-world nested data structures which contain many more
optional and list fields.

## Problem 2: Missing Values

In a nested data structures the primitive value is always found at the leaf nodes. Take `phones.number` where `phones`
is a list field and `number` is a optional field. The primitive value will be found in the `number` field.

There are many possible states for missing values:

1. The `phones` field is not present.
2. The `phones` field is empty.
3. The `number` field is not present.

The more levels of nesting and longer the path, the possible states explode. We have to explicitly track at which
nested level a path terminated.

Say I took the example from earlier where only `phone.phone_type` is present and serialized it to JSON.

```rust
// @formatter:off
Contact {
  name: None,
  phones: Some(vec![
    Phone {
      number: None,
      phone_type: Some(PhoneType::Home),
    },
  ]),
},
// @formatter:on
```

To keep the JSON compact for transmission I decide to remove all the fields which contains null values. The compact
serialized representation looks like this.

```json
{
  "phones": [
    {
      "phone_type": "Home"
    }
  ]
}
```

Here `name` field and `phones.number` fields are missing. Without a shared `Contact` schema it now becomes impossible to
identify which fields or paths are missing in this value instance. The schema is the single source of truth.

## Problem 3: Different Lists, Identical Representation

We are going to look at an example of nested lists. There is an outer list and an inner list, so there are two
levels of nesting in these values. The examples are deliberate. They all have the same sequence of numbers
appearing in the same order. The only difference between these examples is in the organization of numbers in the
innermost list.

```json
// @formatter:off
[[1, 2], [3], [4, 5, 6]]        // 3 inner lists

[[1, 2, 3, 4, 5, 6]]            // 1 inner lists

[[1], [2], [3], [4], [5], [6]]  // 6 inner lists

[[1, 2, 3], [4, 5, 6]]          // 2 inner lists

[[1], [2, 3], [4], [5, 6]]      // 4 inner lists
// @formatter:on
```

The schema definition for the above values looks like this.

```rust
let number_item_field = Field::new("item", DataType::Int32, true);

// @formatter:off
let nested_numbers_field = Field::new(
    "nested_numbers",
    DataType::List(Arc::new(number_item_field)),
    true
);
// @formatter:on

let schema = Schema::new(vec![nested_numbers_field]);
```

If we recursively unnest the above examples we get an identical sequence from 1 to 6. Without additional metadata it
now impossible to reconstruct the original values. You can take a step back for a moment and consider how will you
design a metadata to encode the list structure.

Here again like earlier we are dealing with small examples. A schema may have more than 2 nested list fields in a
path. There is no restriction that they should appear next to each other. The nested lists can be separated by one
or more levels of nested struct fields. Any combination is possible. If you came up with a scheme for the earlier
simple example maybe now you can consider how well it accommodates arbitrary depth and these conditions.

## Problem 4: Empty Lists

A list instance has these valid states.

1. It can have 1 or more items.
2. It can be empty.
3. It is not present and is `NULL` (assuming the outer list field is defined as nullable)

```json
[[1, 2], [3], [4, 5, 6]]        // 3 inner lists

[[1, 2], [], ,[3], [4, 5, 6]]        // 3 inner lists
```

## Problem 5: Sparse Values, Storage inefficiency

## Problem 6: Partial access (read a single path)

## Problem 7: Predicate Pruning/Skipping

## Shifting burden of normalization in storage, join in querying to storage layer

