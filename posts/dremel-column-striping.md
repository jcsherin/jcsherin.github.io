---
title: Flattening nested records into columns
description: Dremel encoding of nested records into columnar values
date: 2025-04-04
tags:
  - dremel
  - column striping
layout: layouts/post.njk
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
│  ├─ PrimaryImageId [int64]
│  └─ AdditionalImageId [int64]*
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
  required int64 ProductId;                 // def_level: 0, rep_level: 0
  
  group ImageGallery {                      // def_level: 1, rep_level: 0
    required int64 PrimaryImageId;          // def_level: 2, rep_level: 0
    repeated int64 AdditionalImageId;       // def_level: 2, rep_level: 1
  }
  
  repeated group AltText {                  // def_level: 1, rep_level: 1
    repeated group Language {               // def_level: 2, rep_level: 2
      required string Locale;               // def_level: 3, rep_level: 2
      optional string Description;          // def_level: 3, rep_level: 2
      repeated string Keyword;              // def_level: 3, rep_level: 3
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
├─ ImageGallery
│  ├─ PrimaryImageId [int64]
│  └─ AdditionalImageId [int64]*
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
