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

Notes:

- A lot of engineering effort goes into building a correct, performant
  vectorized query execution engine for analytical workloads. You can use
  either SQL or DataFrames to run interactive analysis over very large
  datasets. The data is ready to query in a relational form, and PAX storage
  model on disk.

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
the array will have the repetition level zero. So `556` has repetition level
of zero, and `557` has a repetition level of one.

For the next value `988` we can infer that it belongs to the second record
because it has a repetition level of zero. This means it has to be the first
value in the array. And the remaining values in the second record `989`, `999`
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
