---
title: Nested Record Shredding
description: Shredding nested data for columnar storage
date: 2025-04-21
tags:
  - record shredding
  - column striping
  - nested data
layout: layouts/post.njk
---

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

## ProductImages: A Hierarchical View of a Nested Record

```
ProductImages
├── product_id: 103
├── images
│   ├── primary_id: 4400
│   └── secondary_image_ids
│       ├── [0]: 4401
│       ├── [1]: 4402
│       └── [2]: 4403
└── alt_text
    └── localizations
        ├── [0]
        │   ├── locale: "en-us"
        │   ├── description: "red running shoe, side view."
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
            ├── description: "red trainer, profile."
            └── keywords
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

- [ ] **TODO:** Diagram value mirrors schema structure perfectly

Here we do not have to track the presence of a field because it can never be
null. In this model, since all fields are mandatory and the model disallows
lists, the cardinality of every field is therefore always one. Each field can be
either a scalar or a struct value, never a collection. We do not have to
determine if a list is empty.

If both restrictions are removed then different concrete values of the same
nested schema can have many possible structures. In these cases, one cannot
reverse the flattened values using the schema alone.

- [ ] **TODO:** Diagram concrete values of schema with optional fields
- [ ] **TODO:** Diagram concrete value of schema with list field

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

---

In columnar storage values of a single column attribute are stored
contiguously. In analytics databases the query optimizer can apply
projection pushdown directly to the data source. This means only those columns
which are specified in the query are read from storage. Analytical queries are
often aggregations over the entire data source. So this can reduce the I/O
required and make the queries run faster.

It is easy to map flat relational data to columns. Given a projection of
columns the original record can be reassembled by index or offset. A record
has the same index or offset across all columns. This is not the case with
nested data structures.

If nested data structures can be shredded into columns, then it is possible
to use a SQL or DataFrame interface to query nested data. All the built-in
optimizations which are available for relational data also then becomes
available to nested data structures. The ability to interactively query millions
or billions of nested data becomes possible in a single node using a vectorized
query execution engines like DuckDB, ClickHouse or Apache DataFusion.

```
# First record
ProductId: 123
ImageGallery:
  PrimaryImageId: 555
  AdditionalImageId:
    - 556
    - 557

# Second record
ProductId: 678
ImageGallery:
  PrimaryImageId: 987
  AdditionalImageId:
    - 988
    - 989
    - 990
```

Nested data structures are tree shaped. The atomic or primitive value is
found at the leaf of the tree. And columns in the nested data structure is
the path from root to leaf. The columns with their data type are:

1. ProductId - Integer
2. ImageGallery.PrimaryImageId - Integer
3. ImageGallery.AdditionalImageId - Array[Integer]

The two records above after being shredded into column values will look like
this:

```
ProductId                       : [123, 678]
ImageGallery.PrimaryImageId     : [555, 987]
ImageGallery.AdditionalImageId  : [556, 557, 988, 989, 990]
```

In the absence of other metadata it is now impossible for us to reassemble
the original records. The structural information is lost with this encoding.
We are unable to identify where a record begins or ends when the nested data
structure contains repeated (array) values. In this representation it is not
possible anymore to know which values in `ImageGallery.AdditionalImageId`
belongs to which records.

```
# First record
ProductId: 123
ImageGallery:
  PrimaryImageId: 555
  AdditionalImageId:
    - 556
    - 557
AltText:
  - Language:
      - Locale: en-US
        Description: Athletic running shoes
        Keyword:
          - shoes
          - running
          - athletic

# Second record
ProductId: 678
ImageGallery:
  PrimaryImageId: 987
  AdditionalImageId:
    - 988
    - 989
    - 990
```

Real world nested data structures are also sparse. In this example the
first nested data contains descriptive text columns, but the second record
does not. For partially or completely missing paths in a nested data
structure NULL values are inserted. The more sparse the data because of
missing column values, the more NULL values there will be.

```
ProductId                       : [123, 678]
ImageGallery.PrimaryImageId     : [555, 987]
ImageGallery.AdditionalImageId  : [556, 557, 988, 989, 990]

# Columns present only in the first record
AltText.Language.Locale         : ["en-US", NULL]
AltText.Language.Description    : ["Athletic running shoes", NULL]
AltText.Language.Keyword        : ["shoes", "running", "athletic", NULL]
```

The Dremel (Google BigQuery) paper (VLDB 2010) introduced a new
representation for nested data in columnar storage which also stored the
structural hierarchy of the nested data side by side with the column values.
This metadata made it possible to reassemble the original nested data
structure back from columnar format.

The ability to represent nested data directly in a columnar format meant
increased developer productivity. There is no need to normalize the nested
data by extracting entities and joining multiple relations using foreign
keys in some star or snowflake schema for data analysis. Developers could
use the SQL query execution for interactive analysis of very large nested
datasets.

Later when Parquet was created it added ground up support nested data
structures in its file format using the techniques and principles described
in the Dremel paper.

For the `ImageGallery.AdditionalImageId` it was impossible to reassemble the
original two records by looking at only the stored column values. In Dremel
they introduced metadata which encodes the structure of the values in the
nested data. They are definition level and repetition level.

In the below example by reading `d` (definition level) and `r` (repetition
level) in tandem with the column values the original nested values can be
reassembled.

```
# ImageGallery.AdditionalImageId Column

d       : [1, 1, 1, 1, 1]             # definition level
r       : [0, 1, 0, 1, 1]             # repetition level
values  : [556, 557, 988, 989, 990]
```

To compute the definition level of `ImageGallery.AdditionalImageId` we need
to count all the optional and repeated fields in it. To compute the
repetition level the index of the value must be known. If there are multiple
repeated fields in column path, then the computed repetition level of the
nearest repeated ancestor.

The schema of the nested data is required for us to know if a field is
defined as required, optional or repeated. So let us inspect the schema for
`ProductImages` document before formalizing the computation of definition
and repetition levels from the nested data.

The schema for `ProductImages` is given below. From the schema we can see that
this is a nested document which contains the display images for a product and
language translations of the image descriptions.

The data model is,

- A field is either a struct type or a primitive type like an integer,
  string, float, boolean etc.
- A field with no explicit multiplicity labels is a required field. A
  required field will always be present in the nested data.
- An optional field is explicitly marked in the schema. In nested data this
  field maybe present or absent.
- A repeated field is represented as an array of values. The type of
  repeated field can be either a struct type or a primitive type.
- The ordering of repeated values is significant.
- The leaf node is always a primitive type, or a repeated field of a
  primitive type.
- A column name is represented using dot notation by joining the field names
  from root to leaf. Eg. `AltText.Language.Keyword`
-

```
ProductImages                     # Document Name
├─ ProductId [int64]
├─ ImageGallery
│  ├─ PrimaryImageId [int64]
│  └─ AdditionalImageId [int64]*  # repeated
└─ AltText?                       # optional
   └─ Language*                   # repeated
      ├─ Locale [string]
      ├─ Description [string]?    # optional
      └─ Keyword [string]*        # repeated

* = repeated
? = optional
```

A definition level for a column value is computed by counting the occurrence
of optional and repeated fields which are present in the value. If an optional
field is absent then we do not increment the definition level. If a repeated
field is empty or missing we do not increment the definition level. So the
definition level can tell us where the path in a tree terminated for any
given column value.

But this is not enough for us to reassemble repeated values. The repetition
level is used to identify the beginning of an array from the rest of the
array values. For computing repetition levels, only repeated fields in a
path are counted.

In `ImageGallery.AdditionalImageId`,

- `ImageGallery` is a required field
- `AdditionalImageId` is a repeated field

```
# ImageGallery.AdditionalImageId Column

definition_levels : [1, 1, 1, 1, 1]
repetition_levels : [0, 1, 0, 1, 1]
values            : [556, 557, 988, 989, 990]
```

From the definition levels we can see that for all values the path is
`ImageGallery.AdditionalImageId` because the definition level is 1 which
means the repeated field `AdditionalImageId` in the path is always present.

There is only a single repeated field, so the repetition levels can be
either zero or one. To identify the start of the array, the first element in
this example will have a repetition level of zero. The remaining values in
the array will have the repetition level of one. So `556` has repetition level
of zero, and `557` has a repetition level of one.

For the next value `988` we can infer that it belongs to the second record
because it has a repetition level of zero. This means it has to be the first
value in the array. And the remaining values in the second record `989`, `989`
because they have a repetition level of 1.

In this example we were able to identify that the repeated values belonged
to two separate nested values using the repetition levels.

Next let us look at a example which contains null values.

```
ProductId: 123
ImageGallery:
  PrimaryImageId: 555
  AdditionalImageId:
    - 556
    - 557
AltText:
  - Language:
      - Locale: en-US
        Description: Athletic running shoes
        Keyword:
          - shoes
          - athletic
  - Language:
      - Locale: en-GB
        Description: Athletic trainers
        Keyword:
          - trainers
          - sport
  - Language:
      - Locale: fr-FR
  - Language:
      - Locale: de-DE
```

The column `AltText.Language.Description` contains a repeated field and
exactly two optional fields. The definition level therefore can be between 0
and 3.

- AltText: optional
- Language: repeated
- Description: optional

After compiling the column values, there are two NULL values. This
represents the missing `Description` in the 2nd and 3rd `Language`
repetition which corresponds to the `Locale`: `fr-FR` and `de-DE`.

```
# AltText.Language.Description Column

values: ["Athletic running shoes", "Athletic trainers", NULL, NULL]
```

Next let us compute the definition levels. The definition level for both the
NULL values is two because the path terminates at `AltText.Language` as the
`Description` field is missing in both cases.

```
# AltText.Language.Description Column

definition_levels : [3, 3, 2, 2]
values            : ["Athletic running shoes", "Athletic trainers", NULL, NULL]
```

Next let us compute the repetition levels. This column has a single repeated
field which is `Language`. So repetition levels will be between 0 and 1 for
all values.

Here the repetition level of zero clearly identifies the first element in
the repeated field `Language`, from the rest.

```
# AltText.Language.Description Column

repetition_levels : [0, 1, 1, 1]
definition_levels : [3, 3, 2, 2]
values            : ["Athletic running shoes", "Athletic trainers", NULL, NULL]
```

Next let us look at an example where there is more than one repeated field
in a column. The `AltText.Language.Keyword` column has two repeated fields
and a single optional field.

Let us compile the values first. The final two NULL values represent the
missing `Keyword` in the second and third repetition of `Language`.

```
# AltText.Language.Keywords

values: ["shoes", "athletic", "trainers", "sport", NULL, NULL]
```

Next let us compute the definition levels. The NULL values have a definition
level of two because `Keyword` field is missing.

```
# AltText.Language.Keyword

values: ["shoes", "athletic", "trainers", "sport", NULL, NULL]
def   : [3, 3, 3, 3, 2, 2]
```

Next let us compute the repetition levels. This looks complicated, but you
will soon see how this exactly reassembles the original nested data structure.

```
# AltText.Language.Keyword

values: ["shoes", "athletic", "trainers", "sport", NULL, NULL]
def   : [3, 3, 3, 3, 2, 2]
rep   : [0, 2, 1, 2, 1, 1]
```

Let us look at the nested column in isolation and give it index numbers.

```
AltText:
  - Language:         # Language[0]
        - Keyword:
          - shoes     # Language[0].Keyword[0]
          - athletic  # Language[0].Keyword[0]
  - Language:         # Language[1]
        - Keyword:
          - trainers  # Language[1].Keyword[0]
          - sport     # Language[1].Keyword[1]
  - Language:         # Language[2]
  - Language:         # Language[3]
```

There are two repeated fields `Alt.Language.Keyword` which are `Language`
and then `Keyword`. So values in this column may have repetition levels - 0,
1 or 2.

The complete path for `shoes` is `Language[0].Keyword[0]`. This value is the
first repeated value in the path of this nested data structure. The
repetition level of `Language[0]` is zero. The repetition level of `Keyword
[0]` is also zero. It inherits the repetition level of the nearest repeated
ancestor.

The second value is `athletic` with path `Language[0].Keyword[1]`. The
computed repetition level is two so that we can distinctly identify that
this is not the first item in `Keyword`. Because this is not the first item
we do not have to consider the repetition level of a repeated ancestor. Here
`Keyword` field is the second repeated field of this column which is present
and therefore the repetition level is two.

The third value is `trainers`. It has the path `Language[1].Keyword[0]`.
Even though this is the first Keyword, it is the second repetition of
Language. So the repetition level of `Language[1]` is one. And since it is
the first keyword, we inherit that value. So the computed repetition level
is one.

The fourth value is `sport` and the path is `Language[1].Keyword[1]`. The
computed repetition level is two here. This is the second keyword, and so
the repetition level is same as the number of repeated fields in this path
which happens to be two.

The fifth and sixth values are both NULL. They have the paths `Language[2]`
and `Language[3]`. The `Keyword` field is empty or missing. So we compute
the repetition level up to `Language` field. And the value is one.

```
# AltText.Language.Keyword

values: ["shoes", "athletic", "trainers", "sport", NULL, NULL]
def   : [3, 3, 3, 3, 2, 2]
rep   : [0, 2, 1, 2, 1, 1]
```

Now that we know how definition and repetition levels are computed, it is
possible to reassemble the nested data structure from the column values,
definition and repetition levels.

From the column storage values we can also reassemble a partial projection of
the nested data structure in its original form. For example if only the
following columns are selected - [ProductId, AltText.Language.Locale] which
is stored in columnar format as,

```
# ProductId Column
values            : [123, 456]
definition_level  : [0, 0]
repetition_level  : [0, 0]

# AltText.Language.Locale
values            : ['en-US', 'en-GB', 'fr-FR', 'de-DE', NULL]
definition_level  : [2, 2, 2, 2, 0]
repetition_level  : [0, 1, 1, 1, 0]
```

The reassembled nested data structure resembles the original but contains
only the selected columns.

```
# Record 1
ProductId: 123
AltText:
  - Language:
      - Locale: en-US
  - Language:
      - Locale: en-GB
  - Language:
      - Locale: fr-FR
  - Language:
      - Locale: de-DE

# Record 2
ProductId: 678
```

This is just a physical representation. In physical storage the NULL values
can be omitted. Because we know that for the column `AltText.Language.
Keyword` has a max definition level of 3. It has an optional field `AltText`
and two repeated fields `Language` and `Keyword`. So when we see a
definition level value lower than 3, we know that it stands for a NULL value.
This way we can avoid storing the NULL values. This is a useful property for
real world nested data structures which are sparse, and therefore has many
NULL values need not be physically stored saving space.

```
# AltText.Language.Keyword

# Logical representation
# values: ["shoes", "athletic", "trainers", "sport", NULL, NULL]

# Physical representation which does not store NULL values
values: ["shoes", "athletic", "trainers", "sport"]
def   : [3, 3, 3, 3, 2, 2]
rep   : [0, 2, 1, 2, 1, 1]
```

In this example `ProductId` is a required field so there is no need to store
the definition levels. The definition level is always zero for all values.
Similarly, a definition level of zero implies that the repetition level is
also zero. So we do not also need to store repetition levels.

```
# ProductId Column
values            : [123, 456]
definition_level  : [0, 0]
repetition_level  : [0, 0]
```

So in this encoding in physical storage we only write the column values.

```
# ProductId Column
values            : [123, 456]
```

---

Draft Notes:

- Improve definition level explanation: counting number of optional and
  repeated fields in the path from root to where the value is found, or
  where the path terminates for NULL values or paths which are entirely
  missing which can happen if the first field in a path is optional and
  therefore is a NULL value.
- In `AltText.Language.Keyword` the computation of repetition levels needs
  improvement. For e.g. in the case of "athletic" the second repeated field
  `Keyword` is repeating, therefore the repetition level switches from zero
  to two.
- The path with index `Language[0].Keyword[1]` for demonstrating the
  relationship with repetition levels can be compiled as a table or a nested
  list. The key is to explicitly outline the repetition level no matter
  which format is used for visualizing the relationship.
- A definition level less than maximum value means a NULL value at that
  specific position in the schema. The count provides clues as to where the
  path terminated.
- Add visual diagrams like in the Twitter blog post - Dremel made simple
  with Parquet.
- Include an example of a flat schema with a required, optional and repeated
  field.
- Include bit-level packing of definition, repetition levels.
- The twitter blog post includes examples of how useful data structures like
  map (key, value) are implemented. It maybe useful to show the separation
  between logical and physical types. Map is a logical type which is
  rewritten to a struct physical type with a required key of type string,
  and an optional value of a primitive type or a record type.
- The twitter blog post also includes a nested list example for
  demonstrating repetition levels. I need to reference it again to check how
  it is represented logically similar to the map type.
- Include a couple of practical SQL query examples over the column
  shredded nested data structures using Apache DataFusion. The test data can
  be generated and written using python to Parquet format.
- Parquet precomputed offset index which helps with deeply nested
  documents where otherwise we have to scan the definition, repetition
  levels from beginning to end to jump to a record. Include a concrete
  example.
- Include record reassembly as a separate example instead of merging it with
  the explanation of definition, and repetition levels.
- Schema merging in Parquet. Include an example.
- In proto3 there is a significant change. Fields are optional by default
  and need not be marked as optional. They don't use the optional keyword
  anymore. The paper uses the proto2 syntax and semantics. The optional
  fields are explicitly marked. So proto3 removed required fields and uses
  default values for missing fields instead of explicitly tracking presence.
  This means it is not possible anymore to identify if a field which was
  explicitly set to default value vs one which was not set at all. But this
  change was included to make it possible to mark an optional field as
  required (dangerous) made it challenging for schema evolution. But proto3
  reintroduced the optional keyword again.
- Include concrete SQL example for predicate pushdown.
- Include Zero-Copy Optimization when reading from Parquet to Arrow.
  Remapping definition levels to validity bitmaps, and repetition levels to
  offset indexes. The last entry (n+1 for n items) indicates position after
  the last element. This also has to do with point access when you want to
  read all the values for record N, you can do offset[N-1] - offset[N] and
  directly read only those values from offset[N-1].
- This is not a post about light-weight compression schemes, so I am not
  adding anything about dictionary encoding, run-length encoding etc.

---

Predicate Pushdown for Nested Fields - Concrete Example
Consider a schema with nested e-commerce orders:
Order
├─ OrderId
├─ Customer
│ ├─ CustomerId
│ ├─ Name
│ └─ PremiumStatus
└─ Items [array]
├─ ProductId
├─ Quantity
└─ Price
Let's say we want all orders where any item has a price over $100:

```sql
SELECT *
FROM orders
WHERE EXISTS (SELECT 1 FROM UNNEST(items) WHERE price > 100)
```

Traditional Approach:

Read all columns for all Order records
Reconstruct the full nested structure
Apply the filter to each record
Return matching records

Predicate Pushdown with Nested Fields:

The query engine identifies that only the Items.Price column needs examining
first
It reads only the Items.Price column with its definition and repetition levels
It creates a bitmap of which Orders have at least one item with price > $100
It then only reads the remaining columns for Orders that matched the filter

For a dataset with 1 million orders but only 5% having items over $100, this
approach reads only 5% of the data for most columns.

---
