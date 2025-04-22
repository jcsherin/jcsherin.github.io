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
└─ AltText
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
# AltText.Language.Keywords

values: ["shoes", "athletic", "trainers", "sport", NULL, NULL]
def   : [3, 3, 3, 3, 2, 2] 
```

Next let us compute the repetition levels. This looks complicated, but you
will soon see how this exactly reassembles the original nested data structure.

```
# AltText.Language.Keywords

values: ["shoes", "athletic", "trainers", "sport", NULL, NULL]
def   : [3, 3, 3, 3, 2, 2] 
rep   : [0, 2, 1, 2, 1, 1]
```

---


In columnar storage values of a single column attribute are stored
contiguously.

Nested data structures are tree shaped. In columnar storage values of a
single column attribute is stored contiguously. For flat relational data it

---


The Dremel(Google BigQuery) VLDB 2010 paper introduced the technique for
__record shredding__ or __column striping__ of complex nested data structures
into a columnar storage format. And a few years later the Parquet columnar
file format was created with ground up support for nested data adopting the
ideas described in the Dremel paper.

The challenges involved in flattening a nested data structure into a
columnar storage format are:

- Preserving the structural hierarchy of the data,
- Identifying where a record begins and ends in the column.

Only if the structure is preserved can the process of shredding be reversed
and the original nested value be reassembled back from columnar storage.
This is made possible by deriving two integer values and stored together
with each shredded column value:

1. Definition Level
2. Repetition Level

This is remarkable because without any extra steps, the nested data can be
queried using modern vectorized query execution engines using the same SQL
or dataframe interface available for relational data. This includes increased
I/O efficiency by reading only those columns which are projected in the query.

The trade-off is the extra space to store the derived definition levels and
repetition levels for every value. But in practice efficient encoding
techniques and light-weight compression schemes are applied to reduce the
storage requirements.

## Data Model

what is required, optional, repeated. what is structure. how is that related
to definition level? what is the intuition for repetition levels? how do
they interact together? maybe simple concrete examples will help. but why
are leading with the data model here before talking about either the
definition and repetition levels.

```

ProductImages
│
├─ ProductId [int64]
│
├─ ImageGallery
│ ├─ PrimaryImageId [int64]
│ └─ AdditionalImageId [int64]*
│
└─ AltText*
└─ Language*
├─ Locale [string]
├─ Description [string]?
└─ Keyword [string]*

* = repeated
  ? = optional

```

Fig. schema for product images and available translations of descriptive text

The column `AltText.Language.Description`

## Repetition Level

## Schema

### Protobuf v2

```

message Product {
required int64 ProductId; // def_level: 0, rep_level: 0

group ImageGallery { // def_level: 1, rep_level: 0
required int64 PrimaryImageId; // def_level: 2, rep_level: 0
repeated int64 AdditionalImageId; // def_level: 2, rep_level: 1
}

repeated group AltText { // def_level: 1, rep_level: 1
repeated group Language { // def_level: 2, rep_level: 2
required string Locale; // def_level: 3, rep_level: 2
optional string Description; // def_level: 3, rep_level: 2
repeated string Keyword; // def_level: 3, rep_level: 3
}
}
}

```

### Tree Diagram

```

Product
│
├─ ProductId [int64]
│
├─ ImageGallery?
│ ├─ PrimaryImageId [int64]
│ └─ AdditionalImageId [int64]*
│
└─ AltText*
└─ Language*
├─ Locale [string]
├─ Description [string]?
└─ Keyword [string]*

* = repeated
  ? = optional

```

### R1

```

ProductId: 12345
ImageGallery:
PrimaryImageId: 555
AdditionalImageId:

- 556
- 557
  AltText:

- Language:
    - Locale: en-US
      Description: Athletic running shoes with cushioned soles
      Keyword:
        - shoes
        - running
        - athletic
- Language:
    - Locale: en-GB
      Description: Athletic trainers with cushioned soles
      Keyword:
        - trainers
        - running
        - sport
    - Locale: fr-FR
    - Locale: de-DE
- Language:
    - Locale: en-IN
      Description: Sports running shoes with extra comfort
      Keyword:
        - shoes
        - running
        - sports
        - comfort

```

### R2

```

ProductId: 67890
ImageGallery:
PrimaryImageId: 987
AdditionalImageId:

- 988
- 989
- 990

```

---

Parquet implements repetition/definition levels for nested data. But
primarily it is used for storing and querying relational data. So if I write
nested data into a Parquet file, does querying it from DuckDB, Apache
DataFusion be similar to how querying works for relational data? In Dremel
the query language is modified to run SQL queries on nested data which is
column-striped and return results as nested data with a schema. This has
better developer experience, but I suspect may not be supported in either
DuckDB, DataFusion out of the box. In the case of DataFusion will I be able
to extend the SQL to support querying and returning nested records instead
of table values?

Cross Join vs Lateral Join for nested data

```sql
SELECT ProductId,
       ARRAY_AGG(l.Locale) AS missing_description_locales
FROM Product
         CROSS JOIN UNNEST(AltText) AS a
         CROSS JOIN UNNEST(a.Language) AS l
WHERE l.Description IS NULL
GROUP BY ProductId
ORDER BY ProductId

SELECT ProductId,
       ARRAY_AGG(l.Locale) AS missing_description_locales
FROM Product,
     LATERAL (SELECT * FROM UNNEST(AltText)) AS a,
     LATERAL (SELECT * FROM UNNEST(a.Language) AS l WHERE l.Description IS NULL)
GROUP BY ProductId
```

---

# Introduction

__Nested Data__

```
DocId: 10
Links
  Forward: 20
  Forward: 40
  Forward: 60
Name
  Language
    Code: 'en-us'
    Country: 'us'
  Language
    Code: 'en'
  Url: 'http://A'
Name
  Url: 'http://B'
Name
  Language
    Code: 'en-gb'
    Country: 'gb'
```

It is possible to directly represent nested data in columnar storage without
applying any normalization (conversion of nested data structure into a
relational form).

A column begins at the root and ends at the leaf node. The concrete column
value exists at the leaf node. If the path terminates early, or if it is
missing then a NULL value is used to indicate the absence of a value for
that column.

In columnar st

__Schema__

```yaml
Document:
  DocId: int64  # required
  Links?: # optional
    Backward*: int64[]
    Forward*: int64[]
  Name*: # repeated
    Language*:
      Code: string  # required
      Country?: string
    Url?: string
```

```
message Document {
   required int64 DocId;
   
   optional group Links {
      repeated int64 Backward;
      repeated int64 Forward; 
   }
   
   repeated group Name {
      repeated group Language {
        required string Code;
        optional string Country; 
      }
      optional string Url; 
   }
} 
```

A column is composed of fields from the root to the leaf as per schema
definition. The concrete value exists at the leaf of a path.

__Nested Data As Columns__

```
DocId                 : [10]
Links.Backward        : [NULL]
Links.Forward         : [20, 40, 60]
Name.Language.Code    : ['en-us', 'en', NULL, 'en-gb']
Name.Language.Country : ['us', NULL, NULL, 'gb']
Name.Url              : ['http://A', 'http://B', NULL]
```

This is a columnar representation of nested data without first normalizing
it into a relational form.

The concrete values exist at the leaf of the nested data. A path of fields
from root to leaf maps to a column. This schema maps to the following
columns:

1. DocId
2. Links.Backward
3. Links.Forward
4. Name.Language.Code
5. Name.Language.Country
6. Name.Url

The schema contains

---

The representation of relational data in columnar format is intuitive. Each
column value is stored contiguously. To reassemble a record all you need is
the index of the value in a column.

This is a logical representation of columnar storage for a relation with
four columns - BookID, Title, Author & Year.

```
BookIDs:    [101, 102, 103, 104]
Titles:     ["Deep Work", "Designing Data-Intensive Applications", "The Soul of A New Machine", "Hackers & Painters"]
Authors:    ["Cal Newport", "Martin Kleppmann", "Tracy Kidder", "Paul Graham"]
Years:      [2016, 2017, 1981, 2004]
```

To reassemble the third record, the column values at index 2 are retrieved:

```
BookIDs[2]  = 103
Titles[2]   = "The Soul of A New Machine"
Authors[2]  = "Tracy Kidder"
Years[2]    = 1981
```

A nested value is a tree structure with values found at the leaf node. The
column name is the path from root to leaf node. There are as many columns as
unique paths in the tree. Let us look at a sample nested value:

```
DocId: 10
Links
  Forward: 20
  Forward: 40
  Forward: 60
Name
  Language
    Code: 'en-us'
    Country: 'us'
  Language
    Code: 'en'
  Url: 'http://A'
Name
  Url: 'http://B'
Name
  Language
    Code: 'en-gb'
    Country: 'gb'
```

This value has the following unique columns:

1. DocId
2. Links.Forward
3. Name.Language.Code
4. Name.Language.Country
5. Name.Url

After extracting the values from the leaf nodes it can be represented in a
columnar format like the relational data:

```
DocId:                [10]
Links.Forward:        [20, 40, 60]
Name.Language.Code:   ['en-us', 'en', 'en-gb']
Name.Language.Country:['us', 'gb']
Name.Url:             ['http://A', 'http://B']
```

In the above representation the structural information is lost. It is
impossible to reassemble the nested value from the columns values like this:

```
Name[0].Language[0].Code[0] = 'en-us'
Name[0].Language[1].Code[0] = 'en'
Name[2].Language[0].Code[0] = 'en-gb'
```

Between 'en' and 'en-gb' the path Name[1] terminates early. This
representation contains only values which are present in the value. It does
not capture values which are missing because the path terminated early.

TODO:

-[ ] Data Model
-[ ] Missing Values
-[ ] Definition Levels
-[ ] Repetition Levels

---

So how are nested values represented in columnar storage?

This is one of the novel contributions from
the [VLDB 2010 paper - Dremel: Interactive Analysis of
Web-Scale Datasets](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/36632.pdf).

> We describe a novel columnar storage format for nested
> data. We present algorithms for dissecting nested records
> into columns and reassembling them.

---

## Scratch

```
Name
 Language
  Code: 'en-us'
  Country: 'us'
 Language
  Code: 'en'
Name
Name
 Language
  Code: 'en-gb'
  Country: 'gb'
```

- See both above and below are the same thing
- The bottom encoding looks weird because of the extra columns with some
  numbers. We'll soon get to how it is computed.
- Those numbers make it possible for us to reassemble the original nested
  record shown above given the table below. It's neat!
- NULL is used to signal the absence of a value in the nested record.
- But why do we need this?
- Interactive ad-hoc querying using a database engine built for the purpose of
  analytics like ClickHouse, DuckDB etc. Not PostgreSQL or MySQL.
- Analytics queries are primarily aggregations. If you only need to find all
  the distinct Name.Language.Country you only need to read the Name.Language.
  Country column. Projections are efficient because you don't have to read
  the entire nested value just to get a single column.
- Adopted by Parquet/Arrow though the encoding differs slightly, the
  principles remain the same. The principle is preserving the structure of
  the nested record

<div style="display: flex; gap: 16px;">
<div>

| **Name.Language.Country** |       |       |
|---------------------------|-------|-------|
| **value**                 | **r** | **d** |
| en-us                     | 0     | 2     |
| en                        | 2     | 2     |
| NULL                      | 1     | 1     |
| en-gb                     | 1     | 2     |
| NULL                      | 0     | 1     |

</div>
<div>

| **Name.Language.Code** |       |       |
|------------------------|-------|-------|
| **value**              | **r** | **d** |
| us                     | 0     | 3     |
| NULL                   | 2     | 2     |
| NULL                   | 1     | 1     |
| gb                     | 1     | 3     |
| NULL                   | 0     | 1     |

</div>
</div>
