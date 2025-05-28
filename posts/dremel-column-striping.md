---
title: An Efficient Representation for Nested Data Structures
description: Shredding nested data for columnar storage
date: 2025-04-21
tags:
  - record shredding
  - column striping
  - nested data
layout: layouts/post.njk
---

Revision
--------

The Dremel paper, "Dremel: Interactive Analysis of Web-Scale Datasets"
introduced a novel representation for nested data structures in columnar
storage. The data extracted from the nested data structure is annotated with
derived metadata: repetition and definition levels. This is known as record
shredding. The reconstruction of the original nested data structure is then
completed by reading back the column data together with their corresponding
repetition and definition levels. This is known as record assembly.

parquet, orc, ground breaking

----

# Introduction

This post is about how strongly-typed nested data structures are
This post is about efficiently representing strongly-typed nested data
structures in column-oriented

When the Parquet columnar file format for data analytics was created, it
incorporated first-class support for strongly-typed nested data structures.
It adopted a novel representation introduced by Dremel in this influential
[VLDB 2010 Dremel Paper]. Dremel is the underlying query execution engine for
Google BigQuery.

[VLDB 2010 Dremel Paper]:https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/36632.pdf

A strongly-typed nested data structure contains

# Old: Introduction

A nested data structure with optional and repeated (arrays) fields is
flattened into a column by column representation. The problem with naive
flattening is that it does not convey information about the original
hierarchical structure of the data. Across multiple instances of nested data
structures it becomes difficult to determine to which instance a flattened
value belongs. If a nested path contains one or more optional fields, it can
abruptly terminate because one of the optional fields is not present.
Therefore, the value itself will not be present in the nested data structure.
The trouble does not end there as nested repeated (arrays) fields introduce
substantial complexity.

When Parquet was conceived, it adopted the columnar representation described
in the [Dremel: VLDB 2010 paper] for nested data structures. Dremel
is the underlying query execution engine for Google BigQuery.


[Dremel: VLDB 2010 paper]:https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/36632.pdf

The defining characteristic of analytical queries is that it scans a large
part of the dataset, involves aggregations and often have high selectivity.
If the data is stored in a columnar format like Parquet, the queries read only
the relevant columns, dramatically reducing the data scanned and improving query
speeds.

This means a query only has to
read the relevant columns, and skip the remaining data.

Without an efficient columnar representation for nested data structures, an
explicit denormalization step is required

- maintenance cost
- run query directly over nested data
- transformation job
- schema evolution (maintenance)
- explicit denormalization
  This significantly lowers the barrier as you can now run analytical queries
  directly over the nested data structures in storage.

You can skip steps in the data pipeline to transform
a nested schema into a flat schema and the continued cost of maintenance as
the nested schema evolves.

- flattening
- terminology
- projections

A naive flattening of nested data structures into columnar format is not
enough. The original hierarchical structure of the data is lost

A naive flattening of nested data structures into columnar format fails to
preserve its original hierarchical structure.

The Dremel paper uses the term
column striping, while Parquet calls

The Dremel technique involves
deriving two metadata values which encodes the original structure and
storing it together with the data. With the metadata it becomes possible to
reconstruct even

# Nested Data Structure

```
+-----------------+
| ProductImages   |
+-----------------+
├──[ product_id ]: 103
├──[ images ]
│  ├──[ primary_id ]: 4400
│  └──[ secondary_image_ids ]
│     ├──[0]: 4401
│     ├──[1]: 4402
│     └──[2]: 4403
└──[ at_text ]
   └──[ localizations ]
      ├──[0]
      │  ├──[ locale ]: "en-us"
      │  ├──[ description ]: "red running shoe, side view."
      │  └──[ keywords ]
      │     ├── [0]: "red shoe"
      │     ├── [1]: "running"
      │     └── [2]: "sport"
      ├──[1]
      │  ├──[ locale ]: "en-au"
      │  └──[ keywords ]
      │     ├── [0]: "red runner"
      │     └── [1]: "jogging"
      └──[2]
         ├──[ locale ]: "en-gb"
         ├──[ description ]: "red trainer, profile."
         └──[ keywords ]
            ├── [0]: "trainer"
            └── [1]: "athletics"
```

Column shredding (aka record shredding) is a technique for flattening deeply
nested data structures and storing them in a columnar format.

| Identifier          | Full Field Path                    |
|:--------------------|:-----------------------------------|
| product_id          | product_id                         |
| primary_id          | images.primary_id                  |
| secondary_image_ids | images.secondary_image_ids         |
| locale              | alt_text.localizations.locale      |
| description         | alt_text.localizations.description |
| keywords            | alt_text.localizations.keywords    |

| product_id | primary_id | secondary_image_ids | locale  | description           | keywords                         |
|:-----------|:-----------|:--------------------|:--------|:----------------------|:---------------------------------|
| 103        | 4400       | 4401                | "en-us" | "red running shoe..." | ["red shoe", "running", "sport"] |
|            |            | 4402                | "en-au" | NULL                  | ["red runner", "jogging"]        |
|            |            | 4403                | "en-gb" | "red trainer, pr..."  | ["trainer", "athletics"]         |

# Introduction

The main issue with flattening nested data structures is ensuring the
process is reversible. The flattened representation needs to contain both data
and the metadata which encodes the structure. From this representation, one
should be able to correctly reconstruct the original nested data structure.

Typically, a query reads only a few attributes. Therefore, it is more
efficient to read only those specific attributes, rather than the entire
nested data structure. Consequently, a core desirable property of
such flattening is the ability to partially reconstruct the nested data
structure, skipping any attributes not mentioned in the query.

# Nested Data Structure

```
+-----------------+
| ProductImages   |
+-----------------+
├── [ product_id ]: 103
├── [ images ]
│   ├── [ primary_id ]: 4400
│   └── [ secondary_image_ids ]
│       ├── [0]: 4401
│       ├── [1]: 4402
│       └── [2]: 4403
└── [ alt_text ]
    └── [ localizations ]
        ├── [0]
        │   ├── [ locale ]: "en-us"
        │   ├── [ description ]: "red running shoe, side view."
        │   └── [ keywords ]
        │       ├── [0]: "red shoe"
        │       ├── [1]: "running"
        │       └── [2]: "sport"
        ├── [1]
        │   ├── [ locale ]: "en-au"
        │   └── [ keywords ]
        │       ├── [0]: "red runner"
        │       └── [1]: "jogging"
        └── [2]
            ├── [ locale ]: "en-gb"
            ├── [ description ]: "red trainer, profile."
            └── [ keywords ]
                ├── [0]: "trainer"
                └── [1]: "athletics"
```

## ProductImages: Illustrative Partial Projections

```
ProductImages_AltText
├── product_id: 103
└── alt_text
    └── localizations
        ├── [0]
        │   ├── locale: "en-us"
        │   └── description: "red running shoe, side view."
        ├── [1]
        │   ├── locale: "en-au"
        └── [2]
            ├── locale: "en-gb"
            └── description: "red trainer, profile."
```

```
ProductImages_References
├── product_id: 103
└── images
    ├── primary_id: 4400
    └── secondary_image_ids
        ├── [0]: 4401
        ├── [1]: 4402
        └── [2]: 4403
```

```
ProductImages_LocalizedKeywords
├── product_id: 103
└── alt_text
    └── localizations
        ├── [0]
        │   ├── locale: "en-us"
        │   └── keywords
        │       ├── [0]: "red shoe"
        │       ├── [1]: "running"
        │       └── [2]: "sport"
        ├── [1]
        │   ├── locale: "en-au"
        │   └── keywords
        │       ├── [0]: "red runner"
        │       └── [1]: "jogging"
        └── [2]
            ├── locale: "en-gb"
            └── keywords
                ├── [0]: "trainer"
                └── [1]: "athletics"
```

# Data Model

For a moment consider a nested data model with the following restrictions:

1. All fields are mandatory. There are never any null values.
2. This model disallows list or array types, permitting only struct types and
   scalar types (includes integers, floating-point, boolean, characters and
   strings).

In such a model, the structure of any data instance perfectly mirrors its
schema. Consequently, the schema alone is sufficient to reconstruct the
original nested data structure from its flattened representation.

## Fixed Schema

```
ProductImages_Fixed_Schema
├── product_id (u64)
├── images
│   └── image_id (u64)
└── alt_text
    ├── locale (String)
    └── description (String)
```

```
ProductImages_Fixed_1
├── product_id: 10785
├── images
│   └── image_id: 55001
└── alt_text
    ├── locale: "en-in"
    └── description: "MacBook Air 13.3\" (M1 chip, 8GB RAM, 256GB SSD), Space Grey."
```

```
ProductImages_Fixed_2
├── product_id: 20488
├── images
│   └── image_id: 60773
└── alt_text
    ├── locale: "en-in"
    └── description: "PortaShell 13.3\" Laptop Sleeve, Grey, with pocket & handle."
```

Here we do not have to track the presence of a field because it can never be
null. In this model, since all fields are mandatory and the model disallows
lists, the cardinality of every field is therefore always one. Each field can be
either a scalar or a struct value, never a collection. We do not have to
determine if a list is empty.

If both restrictions are removed then different concrete values of the same
nested schema can have many possible structures. In these cases, one cannot
reverse the flattened values using the schema alone.

## With Optional Field

```
ProductImages_Optional_Schema
├── product_id (u64)
├── images
│   └── image_id (u64)
└── alt_text
    ├── locale (String)
    └── description (String, optional)  // <--- This field is now optional
```

```
ProductImages_Optional_1
├── product_id: 20488
├── images
│   └── image_id: 60773
└── alt_text
    ├── locale: "en-in"
    // description is not present in this example
```

## With List Field

```
ProductImages_List_Schema
├── product_id (u64)
└── images
    ├── primary_id (u64)
    └── secondary_image_ids (List<u64>)
```

```
ProductImages_List_1
├── product_id: 30501
└── images
    ├── primary_id: 77001
    └── secondary_image_ids
        ├── [0]: 77002
        ├── [1]: 77003
        └── [2]: 77004
```

```
ProductImages_List_2
├── product_id: 30502
└── images
    ├── primary_id: 77005
    └── secondary_image_ids: [] // List is empty
```

This simplified model restricts our ability to express most real-world
nested datasets because it lacks optional fields and disallows lists. But it
helps us narrow down the sources of structural variations.

1. Is an optional field present or not?
2. Is a list empty or not?

Consider a data model with these restrictions removed. It allows both
optional fields and lists - the very features that introduce structural
variations. Reconstructing any flattened nested data structure with a
well-defined schema back to its original form is indeed possible, but this
capability depends entirely on encoding these two structural properties
(optional field presence and list status) as metadata.

# Flattening

To flatten a nested data structure, we first enumerate its potential column
attributes. This involves traversing the schema in depth-first order. Each
column attribute corresponds to a unique path of fields from the document's
root up to a leaf node. The field definition at this leaf node determines
the optionality and data type for that column attribute.

- [ ] **TODO:** Diagram enumerate paths in a schema

Next, we traverse an actual data instance of the nested structure, again in
depth-first order. As we reach the leaf nodes in the data, we extract the
scalar values and append them to the lists associated with their
corresponding column attributes (as identified from the schema).

- [ ] **TODO:** Diagram flattening nested value to column values

# Metadata: Definition Level

A path may terminate early before it reaches the leaf node. This happens
when an optional field is not present in the value tree. There is also a
possibility that a path defined in the schema is not present in the value
tree. Here the first field in the path is an optional field and is not
present in the value. This will go undetected without the schema.

- [ ] **TODO:** Diagram various legal constructions of terminating paths

A path may also contain list fields which maybe empty in the value. So we
need to know in the case of a list field whether it is empty or contains at
least one element.

We can track both cases by counting the optional fields which are present
and list fields which are not empty. If an optional field is present the
count is incremented by one. If a list field contains at least one element
the count is incremented by one. Note that the count is only incremented by
one for the list field and does not depend on the number of elements in the
list.

The terminology used by Dremel for this count metadata field is **definition
level**. If a path contains N optional fields and list fields, then its
definition level will be in the range (inclusive) [0, N]. The upper bound
can be derived from the schema.

The definition level of a non-NULL value will always be N. For null values
the definition level tell us the exact point at which the path terminates.
This allows us to reconstruct partially terminated paths. If the definition
level is 0, we know that path is entirely missing from the value.

# Metadata: Repetition Level

A list field is variable length and maps to multiple values in a flattened
representation. Here it should be possible to determine list boundaries so
that the flattened values can be reverse mapped back to the correct list field.

A schema path may define multiple list fields (nested lists). This poses
a serious challenge in also having to identify at which level of nesting a
flattened value belongs.

Here we need to consider only the non-empty list fields because an empty
list field is already handled by the definition level.

Dremel uses a clever trick to encode both the list boundary and nesting
level of a flattened list element into a single metadata value known as
repetition level.

There are similarities to how definition level tracks the presence of
optional/list fields along a path. The maximum possible repetition level for
any value along a given schema path (let's call it R) is equal to the number
of list fields defined in that schema path. The actual repetition level (r)
assigned to a flattened value will then fall within the range (inclusive) [0,
R]. If R = 2, then a flattened value can have a repetition level of 0, 1 or 2.

From interpreting the repetition level of a flattened value we can identify
the record it belongs to and the nesting level within the record if the
schema path defines nested lists.

The rules for interpreting repetition levels are:
If r = 0: The value marks the beginning a new record. It is the first
occurrence of this specific field path within that new record.
If r < R (max repetition level for a path): This is the first element of a
new instance of a nested list. The value of 'r' tells us which list in the
path (from root to leaf) this new instance belongs to.
If r = R: This value is a subsequent item belonging to the same instance
of most recent list that was previously started.

# Metadata: required fields only path

Consider a schema path that contains no optional fields and no list fields.
For any value present along such a path the definition level will always be
zero as there are no optional or list fields. The repetition level will
always be zero as there are no list fields.

In this specific scenario, which mirrors our simplified data model, the
schema alone is indeed sufficient to reconstruct the data, as no structural
variations due to optionality or repetition exist.

# Putting it all together

Let us now see how this works in practice.

## Schema

```
ProductImages (doc)
|- product_id (u64)
|- images
|  |- primary_id (u64)
|  |- secondary_image_ids (list u64)
|- alt_text
   |- localizations (list)
      |- locale (string)
      |- description (optional string)
      |- keywords (list string)
```

This schema represents a nested data structure which contains references to
the display images belonging to a product. It also provides zero or more
localized alt text entries for internationalization (i18n). Each entry has a
locale, description (the alt text itself), and keywords (for seo).

The individual attributes present in this schema are:

| No. | Schema Path                        | Data Type | Optional |
|-----|------------------------------------|-----------|----------|
| 1   | product_id                         | u64       | N        |
| 2   | images.primary_id                  | u64       | N        |
| 3   | images.secondary_image_ids         | u64       | N        |
| 4   | alt_text.localizations.locale      | string    | N        |
| 5   | alt_text.localizations.description | string    | Y        |
| 6   | alt_text.localizations.keywords    | string    | N        |

Each path in our schema has a maximum possible definition level (D) and
repetition level (R). The maximum D is calculated by counting how many
fields along the path are either optional or list types. The maximum R is
the count of just the list type fields along the path. For our ProductImages
schema, these maximums are:

| No. | Schema Path                        | D | R |
|-----|------------------------------------|---|---|
| 1   | product_id                         | 0 | 0 |
| 2   | images.primary_id                  | 0 | 0 |
| 3   | images.secondary_image_ids         | 1 | 1 |
| 4   | alt_text.localizations.locale      | 1 | 1 |
| 5   | alt_text.localizations.description | 2 | 1 |
| 6   | alt_text.localizations.keywords    | 2 | 2 |

D - Definition Level
R - Repetition Level

### document 1 (D1)

Multiple holes are present in this document. the secondary images list is
empty. the single localization which is present does not have any keywords
defined yet.

```yaml
product_id: 101
images:
  primary_id: 2001
  secondary_image_ids: [ ]
alt_text:
  localizations:
    - locale: "en-us"
      description: "blue casual t-shirt."
      keywords: [ ]
```

### document 2 (D2)

This document contains only the product id and primary image id. the
remaining properties are either missing or empty.

```yaml
product_id: 102
images:
  primary_id: 3010
  secondary_image_ids: [ ]
alt_text:
  localizations: [ ]
```

### document 3 (D3)

This is a fairly complete document with multiple secondary image ids and
multiple localizations with varied content.

```yaml
product_id: 103
images:
  primary_id: 4400
  secondary_image_ids:
    - 4401
    - 4402
    - 4403
alt_text:
  localizations:
    - locale: "en-us"
      description: "red running shoe, side view."
      keywords:
        - "red shoe"
        - "running"
        - "sport"
    - locale: "en-au"
      # placeholder locale does not yet have a description
      keywords:
        - "red runner"
        - "jogging"
    - locale: "en-gb"
      description: "red trainer, profile."
      keywords:
        - "trainer"
        - "athletics"
```

After flattening the documents, the values in each path are stored contiguously.

```yaml
product_id:
  values: [ 101, 102, 103 ]
  def: [ 0, 0, 0 ]
  rep: [ 0, 0, 0 ]

images.primary_id:
  values: [ 2001, 3010, 4400 ]
  def: [ 0, 0, 0 ]
  rep: [ 0, 0, 0 ]
```

The paths product_id and images.primary_id contain only mandatory fields. So
the derived definition and repetition level values is going to be zero for
all values. So it is equivalent to the following storage representation.

```yaml
product_id:
  values: [ 101, 102, 103 ]

images.primary_id:
  values: [ 2001, 3010, 4400 ]
```

## Path: images.secondary_image_ids

Step 1: Flatten D1

```yaml
images.secondary_image_ids:
  values: [ NULL ]
  def: [ 0 ]
  rep: [ 0 ]
```

The definition level is zero because secondary_image_ids list is empty. The
repetition level is zero here for the same reason. The repetition level for
the path images.secondary_image_ids is in the range (inclusive) [0, 1]. Here
the interpretation of zero is not that this is the first element in the list,
which is not possible as the list is empty. But we have to read it together
with the NULL value and the definition level value which happens to be zero
and signals that the list is empty.

Step 2: Flatten D2

```yaml
images.secondary_image_ids:
  values: [ NULL, NULL ]
  def: [ 0, 0 ]
  rep: [ 0, 0 ]
```

The same reasoning as above applies here.

Step 3: Flatten D3

```yaml
images.secondary_image_ids:
  values: [ NULL, NULL, 4401, 4402, 4403 ]
  def: [ 0, 0, 1, 1, 1 ]
  rep: [ 0, 0, 0, 1, 1 ]
```

As the list is not empty, the definition level for all values in this list
is 1. The repetition level for the first element 4401 is zero, and for
subsequent elements the repetition level is one. This marks 4401 as the
beginning of a new list instance in a new document.

## Path 4: alt_text.localizations.locale

The struct alt_text is required/mandatory and locale is a required/mandatory
string. The localizations is a list. So this has a maximum definition level
of 1 and maximum repetition levels is also 1.

The inner data type of localizations is a struct with three properties:

- locale: a required/mandatory string
- description: an optional string
- keywords: a list of strings

Step 1: Flatten D1

```yaml
alt_text.localizations.locale:
  values: [ "en-us" ]
  def: [ 1 ]
  rep: [ 0 ]
```

In D1 a single locale is defined. So the derived definition level is 1
because the localizations list is present. The repetition level is 0 which
indicates that this is the first occurrence of the locale struct property in
a new instance of the localizations list in a new document.

Step 2: Flatten D2

```yaml
alt_text.localizations.locale:
  values: [ "en-us", NULL ]
  def: [ 1, 0 ]
  rep: [ 0, 0 ]
```

In D2 the localizations list is empty. So the definition level becomes 0 and
NULL value is inserted. The repetition level is zero, but has to interpreted
together with the NULL value and definition level which shows that the
localizations list is empty in this document.

Step 3: Flatten D3

```yaml
alt_text.localizations.locale:
  values: [ "en-us", NULL, "en-us", "en-au", "en-gb" ]
  def: [ 1, 0, 1, 1, 1 ]
  rep: [ 0, 0, 0, 1, 1 ]
```

There are three locale values in D3. They are added to values. The
definition level is 1 because the localizations is not empty. The
repetition level is 0 for "en-us" which signals that this is the beginning
of a new localizations list instance in a new document and this is the
first element. The subsequent elements in the list therefore have the
repetition level of 1.

## Path 5: alt_text.localizations.description

In this path localizations is a list and description is an optional string.
So the maximum definition level is 2. The maximum value for repetition level
is 1 as there is only a single list in this path.

Step 1: Flatten D1

```yaml
alt_text.localizations.description:
  values: [ "blue casual t-shirt" ]
  def: [ 2 ]
  rep: [ 0 ]
```

The optional description is present in D1. So the definition level is 2, and
the repetition level is 0. This is the first description in a new instance
of localizations, at the start of the document.

Step 2: Flatten D2

```yaml
alt_text.localizations.description:
  values: [ "blue casual t-shirt", NULL ]
  def: [ 2, 1 ]
  rep: [ 0, 0 ]
```

In D2, the localizations list is empty. A NULL value is inserted for
description. The definition level is 1 indicating that the path is defined
up to the localizations list field, but the list is empty so no actual
description value exists within a list item. The repetition level is 0, as
the entry corresponds to a new record (D2).

Step 3: Flatten D3

```yaml
alt_text.localizations.description:
  values:
    - "blue casual t-shirt"         # D1
    - NULL                          # D2
    - "red running shoe, side view" # D3
    - NULL                          # D3
    - "red trainer, profile"        # D3
  def: [ 2, 1, 2, 1, 2 ]
  rep: [ 0, 0, 0, 1, 1 ]
```

The second description is not present, so a NULL value is inserted in its
place and its definition level is 1. The first description and third
description are present, so they have a definition level of 2. The
repetition level of the first description is 0 to indicate that this is a
new instance of a localizations list, and this is the first description
property in the first struct element. The subsequent descriptions have a
repetition level of 1 to indicate that they are elements belonging to this same
list.

## Path 6: alt_text.localizations.keywords

This is the first time we encounter a nested list. Both localizations and
keywords are list fields. The maximum definition level is 2 and the maximum
repetition level is also 2 for this path.

Step 1: Flatten D1

```yaml
alt_text:
  localizations:
    - locale: "en-us"
      description: "blue casual t-shirt."
      keywords: [ ]
```

In D1, keywords list is present, but it is empty.

```yaml
alt_text.localizations.keywords:
  values: [ NULL ]
  def: [ 1 ]
  rep: [ 0 ]
```

We insert a NULL value because keywords list is empty. The definition level
is 1 as localizations list is present and not empty, but the keywords list
is empty. The repetition level is zero because this is the start of a new
record (D1).

Step 2: Flatten D2

```yaml
alt_text:
  localizations: [ ]
```

In D2, the localizations list is empty.

```yaml
alt_text.localizations.keywords:
  values: [ NULL, NULL ]
  def: [ 1, 0 ]
  rep: [ 0, 0 ]
```

The definition level in this case is 0 because the localizations list is
empty. The repetition level is 0 because this is the start of a new record (D2).

Step 3: Flatten D3

```yaml
alt_text:
  localizations:
    - locale: "en-us"
      description: "red running shoe, side view."
      keywords:
        - "red shoe"
        - "running"
        - "sport"
    - locale: "en-au"
      # placeholder locale does not yet have a description
      keywords:
        - "red runner"
        - "jogging"
    - locale: "en-gb"
      description: "red trainer, profile."
      keywords:
        - "trainer"
        - "athletics"
```

The localizations list contains 3 items. So let us break this up by item so
we can clearly see how repetition levels varies across keywords for each item.

After processing the first item in localizations:

```yaml
alt_text.localizations.keywords:
  values: [ "red shoe", "running", "sport" ]
  def: [ 2, 2, 2 ]
  rep: [ 0, 2, 2 ]
```

The localizations list is present and the keywords list is present, and both
are not empty. So the definition level is 2. The first item in the list has
a repetition level of 0 because this is the beginning of a new record (D3).
The other items have a repetition level of 2 to indicate that this is a
continuation of the current keywords list.

Now the second item,

```yaml
alt_text.localizations.keywords:
  values: [ "red runner", "jogging" ]
  def: [ 2, 2 ]
  rep: [ 1, 2 ]
```

The definition level is 2 as both lists in the path are present. Note that
the repetition level for "red runner" is not zero. This is significant! This
keywords field is part of the second struct item in localizations. And
localizations has a repetition level in the range (inclusive) [0, 1]. The 0
repetition level marks the beginning of a new record, but here this is the
second item in the localizations list. The computed repetition level is
therefore 1 for all remaining structs in this list. Now when we reach the
keywords list, this is a new list instance and "red runner" is the first
element in the list. So the repetition level for "red runner" is 1. The
remaining items in the list have a repetition level of 2 to indicate that it
is a continuation of the newly opened list.

And the final item in localizations,

```yaml
alt_text.localizations.keywords:
  values: [ "trainer", "athletics" ]
  def: [ 2, 2 ]
  rep: [ 1, 2 ]
```

The repetition level of "trainer" is 1 because it is child property of the
third struct item in localizations list. The repetition level for the third
item can only be 1. Also, "trainer" is the first item in the new instance of
keywords list so we give it a repetition level of 1 to identify it as the
first element in a new keywords list. The remaining items in keywords get a
repetition level of 2.

The final aggregation of keywords after processing D1, D2 and all three
localization items from D3:

```yaml
alt_text.localizations.keywords:
  values: [
    NULL,                           # D1: keywords list is empty
    NULL,                           # D2: localizations list is empty
    "red shoe", "running", "sport", # D3: 1st localization ("en-us")
    "red runner", "jogging",        # D3: 2nd localization ("en-au")
    "trainer", "athletics"          # D3: 3rd localization ("en-gb")
  ]
  def: [
    1,        # D1: (localizations present, keywords empty)
    0,        # D2: (localizations empty)
    2, 2, 2,  # D3: (both list present and non-empty)
    2, 2,
    2, 2
  ]
  rep: [
    0,        # D1: new record
    0,        # D2: new record
    0, 2, 2,  # D3: 1st localization: r=0 for first keyword, r=2 for remaining
    1, 2,     # D3: 2nd localization: r=1 for first keyword, r=2 for remaining
    1, 2      # D3: 3rd localization: r=1 for first keyword, r=2 for remaining
  ]
```

# Summary

- [ ] Condense main ideas, supporting arguments, opinions, recap

