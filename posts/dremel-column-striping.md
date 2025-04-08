---
title: Flattening nested records into columns
description: Dremel encoding of nested records into columnar values
date: 2025-04-04
tags:
  - dremel
  - column striping
layout: layouts/post.njk
---

# Column Format

| BookID | Title                                 | Author           | Year |
|--------|---------------------------------------|------------------|------|
| 101    | Deep Work                             | Cal Newport      | 2016 |
| 102    | Designing Data-Intensive Applications | Martin Kleppmann | 2017 |
| 103    | The Soul of A New Machine             | Tracy Kidder     | 1981 |
| 104    | Hackers & Painters                    | Paul Graham      | 2004 |

This data is stored in a row format in a transactional, database management
system like PostgreSQL, or MySQL.

A logical view of the row data layout (not how it is physically stored on disk):

```
[106, "Deep Work", "Cal Newport", 2016]
[104, "Designing Data-Intensive Applications", "Martin Kleppmann", 2017]
[112, "The Soul of A New Machine", "Tracy Kidder", 1981]
[113, "Hackers & Painters", "Paul Graham", 2004]
```

In analytical, column-oriented database management system like ClickHouse or
DuckDB, the data is stored in a columnar format.

A logical view of the column data layout:

```
BookIDs:    [106, 104, 112, 113]
Titles:     ["Deep Work", "Designing Data-Intensive Applications", "The Soul of A New Machine", "Hackers & Painters"]
Authors:    ["Cal Newport", "Martin Kleppmann", "Tracy Kidder", "Paul Graham"]
Years:      [2016, 2017, 1981, 2004]
```

Reassembling a record for the book "Hackers & Painters" is as simple as
fetching the values at index 3 from all the columns.

```
[113, "Hackers & Painters", "Paul Graham", 2004]
```

If we are interested in only a subset of the columns: `Titles`, `Years` then
there is no need to scan the other columns to reassemble the record. This
reduces the amount of I/O needed for scanning the data and makes query
processing
efficient.

```
["Hackers & Painters", 2004]
```

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
