---
title: "Record Shredding: Part 1"
date: 2025-06-06
published: false
---

<style>
th, td {
  text-align: center;
  vertical-align: top;
  padding: 8px 16px;
}

.table-container {
  display: flex;
  justify-content: space-around;
  gap: 20px;
}

.col-view td:nth-child(1) { background-color: #FFFFE0; }
.col-view td:nth-child(2) { background-color: #FFDAB9; }
.col-view td:nth-child(3) { background-color: #E6E6FA; }

.row-view tr:nth-child(1) td { background-color: #FFFFE0; }
.row-view tr:nth-child(2) td { background-color: #FFDAB9; }
.row-view tr:nth-child(3) td { background-color: #E6E6FA; }
.row-view tr:nth-child(4) td { background-color: #DDFADD; }

.col-view {
  border-spacing: 8px 0;
}

.row-view {
  border-spacing: 0 8px;
}

.record-container {
  display: flex;
}

/* to be removed later */
body {
  max-width: 800px;
  font-size: 20px;
  margin: 0 auto 220px;
}
</style>

# Recap: Flat, Relational Data

## Row-Major vs. Column-Major Representation

The data is stored in row-major order in transactional storage engines like PostgreSQL, MySQL or SQLite. This matches
the transactional access patterns which targets to read, modify or delete a specific row of data or a very small set of
rows. The filtering conditions of the query have high-selectivity. When reading a row all the column attributes in
the relation are materialized in main memory. Having the entire row contiguous in memory once fetched minimizes
additional disk I/O.

On the other hand in an analytical database like ClickHouse or DuckDB, the access pattern is different enough that
it makes sense to store the same data in column-major order. Data for each column is stored contiguously. This is
because analytical workloads are primarily read-only, scan-heavy, heavily utilize aggregations with an explicit
goal of summarizing the data. The primary advantage here is that only those columns which are relevant for a query
needs to be materialized in memory. For scan-heavy queries which often have low-selectivity, this has a significant
impact on minimizing disk I/O and reducing memory footprint.

<div class="table-container">
  <table class="col-view">
    <thead>
      <tr>
        <th colspan="3">column-major order</th>
      </tr>
      <tr>
        <th>id</th>
        <th>username</th>
        <th>role</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>101</td>
        <td>Alice</td>
        <td>sender</td>
      </tr>
      <tr>
        <td>102</td>
        <td>Bob</td>
        <td>receiver</td>
      </tr>
      <tr>
        <td>103</td>
        <td>Eve</td>
        <td>eavesdropper</td>
      </tr>
      <tr>
        <td>104</td>
        <td>Trudy</td>
        <td>intruder</td>
      </tr>
    </tbody>
  </table>

  <table class="row-view">
    <thead>
      <tr>
        <th colspan="3">row-major order</th>
      </tr>
      <tr>
        <th>id</th>
        <th>username</th>
        <th>role</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>101</td>
        <td>Alice</td>
        <td>sender</td>
      </tr>
      <tr>
        <td>102</td>
        <td>Bob</td>
        <td>receiver</td>
      </tr>
      <tr>
        <td>103</td>
        <td>Eve</td>
        <td>eavesdropper</td>
      </tr>
      <tr>
        <td>104</td>
        <td>Trudy</td>
        <td>intruder</td>
      </tr>
    </tbody>
  </table>
</div>

## Performance Benefits

The data locality in columnar storage is crucial for query performance. It enables data parallelism where different
blocks of column values can be processed in parallel on multiple CPU cores. It helps with replacing scalar
operations with vectorized processing. Together data parallelism and vectorized processing has a big impact in
making analytical queries execute faster and efficiently.

# Record Shredding

Record shredding is the act of breaking up nested data structures into a flat, relational format. From above, we know
the benefits of columnar representation for analytical workloads. By doing this, the benefits of data parallelism,
vectorization and efficient I/O extends to nested data structures as well. The ability to directly represent nested
data structures also leads to operational simplicity. A nested data structure doesn't have to be manually normalized
into relational form which is ready for ad-hoc querying. There is no human judgement required. The ad-hoc queries
are going to execute on direct columnar representation of the nested data structure. That is the promised land!

# Record Assembly

Record assembly is how the columnar representation is interpreted to reconstruct either fully or partially the
stored nested data structures. Similar to flat, relational data here too we want to read only the relevant columns
and ignore the rest. So the ability to partially reconstruct only the relevant parts of the nested data structure is
an important feature.

# Nested Data Structure

Here is the schema definition for a nested data structure.

![A tree diagram representation for `UserProfile` nested data type](img/user_profile_schema.svg)

```graphql
type UserProfile {
  uid: ID!                  # This field is mandatory
  displayName: String       # This is an optional field
  tags: [String!]           # This is a repeated (array) field
  preferences: Preferences  # This is an optional nested field
}

# All fields in this type are optional
type Preferences {
  theme: String           # Values can be "dark", "light" or "system".
  language: String        # Uses BCP 47 language tags, e.g., "en-US".
  notifications: Boolean
}
```

The `uid` field is the only mandatory field. So a valid instance can be created with the `uid` alone. The other
fields can be progressively added.

The `displayName` and `preferences` are optional fields.

The `tags` is a repeated (array) field. The schema does not convey any information about the cardinality of this
field. That can be known only after parsing a raw nested data structure instance.

The `preferences` adds a level of nesting. All the fields in it are also optional.

This is the schema definition for a nested data structure. The syntax I have used here is the GraphQL Schema
Definition Language. It may as well be a definition written in a programming language or using the syntax of an
interface definition language like Thrift or Protocol Buffers. It does not matter. The relevant part is how we logically
define the fields:

- Is a field optional or mandatory?
- Is this a repeated (array)?
- What is the data type of the field?
- What is the name of the field?

<div class="record-container">

```json
{
  "uid": "a1b2c3d4",
  "displayName": "Alice Wonderland",
  "tags": [
    "reader",
    "dreamer"
  ]
}
```

```json
{
  "uid": "e5f6g7h8",
  "displayName": "Chris Coder",
  "tags": [
    "developer",
    "python",
    "oss"
  ],
  "preferences": {
    "theme": "light"
  }
}
```

```json
{
  "uid": "i9j0k1l2",
  "displayName": "Bob The Builder",
  "tags": [
    "builder",
    "diy"
  ],
  "preferences": {
    "theme": "dark",
    "notifications": true
  }
}
```

</div>


