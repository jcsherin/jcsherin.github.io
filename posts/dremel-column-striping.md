---
title: Flattening nested records into columns
description: Dremel encoding of nested records into columnar values
date: 2025-04-04
tags:
  - dremel
  - column striping
layout: layouts/post.njk
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
