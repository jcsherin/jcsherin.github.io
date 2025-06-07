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
.col-view td:nth-child(4) { background-color: #DDFADD; }
.col-view td:nth-child(5) { background-color: #F0F8FF; }

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

This is a place for claims:

- structural variation
- sparse
- can't shred without schema
- columns fall out from schema

## Schema: Optional & Repeated Fields

The figure 1. shows all the fields of a _UserProfile_ nested data structure.

A field has a name and a data type.

An _optional_ field is not mandatory. It does not have to be present in the instance of data. Consider _preferences.
theme_ which is composed of two optional fields. There are three possible variations:

1. Both the fields are present: _preferences.theme_.
2. The _theme_ field is not present: _preferences_.
3. Both fields are not present.

A _repeated_ field is an array of values and it can be empty. The element data type of repeated field can be a
primitive type like Integer, String or Boolean. The _tags_ field belongs to this category. The element data type can
also be a _Struct_.

![A tree diagram representation for `UserProfile` nested data type](img/user_profile_schema.svg)
Figure 1. Schema for UserProfile

## Column Mapping

The schema is necessary to identify all the columns of a nested data structure. The primitive values exist in the
leaf nodes. A column name is represented by the field names from root to leaf separated by dot notation. The
_UserProfile_ schema contains the following set of columns:

1. _uid_
2. _displayName_
3. _tags_
4. _preferences.theme_
5. _preferences.language_
6. _preferences.notifications_

## Logical Columnar View

These are examples of concrete instances of the _UserProfile_ schema defined in Figure 1. They have varying levels
of completeness.

<div class="record-container">

```json
{
  "uid": "1234",
  "displayName": "Alice Wonderland",
  "tags": [
    "reader",
    "dreamer"
  ]
}
```

```json
{
  "uid": "5678",
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
  "uid": "9012",
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

Let see how these examples map to a logical columnar view.

<table class="col-view">
  <thead>
    <tr>
      <th>uid</th>
      <th>displayName</th>
      <th>tags</th>
      <th>preferences.theme</th>
      <th>preferences.notifications</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1234</td>
      <td>Alice Wonderland</td>
      <td>[reader, dreamer]</td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>5678</td>
      <td>Chris Coder</td>
      <td>[developer, python, oss]</td>
      <td>light</td>
      <td></td>
    </tr>
    <tr>
      <td>9012</td>
      <td>Bob The Builder</td>
      <td>[builder, diy]</td>
      <td>dark</td>
      <td>true</td>
    </tr>
  </tbody>
</table>

The _tag_ array values can be expanded further so that each value is in its own separate row. This is similar to the
functionality of the _unnest_ function in SQL.

<table class="col-view">
  <thead>
    <tr>
      <th>uid</th>
      <th>displayName</th>
      <th>tags</th>
      <th>preferences.theme</th>
      <th>preferences.notifications</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1234</td>
      <td>Alice Wonderland</td>
      <td>reader</td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>1234</td>
      <td>Alice Wonderland</td>
      <td>dreamer</td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>5678</td>
      <td>Chris Coder</td>
      <td>developer</td>
      <td>light</td>
      <td></td>
    </tr>
    <tr>
      <td>5678</td>
      <td>Chris Coder</td>
      <td>python</td>
      <td>light</td>
      <td></td>
    </tr>
    <tr>
      <td>5678</td>
      <td>Chris Coder</td>
      <td>oss</td>
      <td>light</td>
      <td></td>
    </tr>
    <tr>
      <td>9012</td>
      <td>Bob The Builder</td>
      <td>builder</td>
      <td>dark</td>
      <td>true</td>
    </tr>
    <tr>
      <td>9012</td>
      <td>Bob The Builder</td>
      <td>diy</td>
      <td>dark</td>
      <td>true</td>
    </tr>
  </tbody>
</table>

After expanding the _tags_ the total number of rows exploded. There are more holes (properties which are not present)
in _preferences.theme_ and _preferences.notifications_.

Maybe we can fill the holes by defining sensible default values. Let the default values for _preferences.theme_ be
_system_ and _preferences.notifications_ be true.

<table class="col-view">
  <thead>
    <tr>
      <th>uid</th>
      <th>displayName</th>
      <th>tags</th>
      <th>preferences.theme</th>
      <th>preferences.notifications</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1234</td>
      <td>Alice Wonderland</td>
      <td>reader</td>
      <td>system</td>
      <td>true</td>
    </tr>
    <tr>
      <td>1234</td>
      <td>Alice Wonderland</td>
      <td>dreamer</td>
      <td>system</td>
      <td>true</td>
    </tr>
    <tr>
      <td>5678</td>
      <td>Chris Coder</td>
      <td>developer</td>
      <td>light</td>
      <td>true</td>
    </tr>
    <tr>
      <td>5678</td>
      <td>Chris Coder</td>
      <td>python</td>
      <td>light</td>
      <td>true</td>
    </tr>
    <tr>
      <td>5678</td>
      <td>Chris Coder</td>
      <td>oss</td>
      <td>light</td>
      <td>true</td>
    </tr>
    <tr>
      <td>9012</td>
      <td>Bob The Builder</td>
      <td>builder</td>
      <td>dark</td>
      <td>true</td>
    </tr>
    <tr>
      <td>9012</td>
      <td>Bob The Builder</td>
      <td>diy</td>
      <td>dark</td>
      <td>true</td>
    </tr>
  </tbody>
</table>

[//]: # (```graphql)

[//]: # (type UserProfile {)

[//]: # (  uid: ID!                  # This field is mandatory)

[//]: # (  displayName: String       # This is an optional field)

[//]: # (  tags: [String!]           # This is a repeated &#40;array&#41; field)

[//]: # (  preferences: Preferences  # This is an optional nested field)

[//]: # (})

[//]: # ()

[//]: # (# All fields in this type are optional)

[//]: # (type Preferences {)

[//]: # (  theme: String           # Values can be "dark", "light" or "system".)

[//]: # (  language: String        # Uses BCP 47 language tags, e.g., "en-US".)

[//]: # (  notifications: Boolean)

[//]: # (})

[//]: # (```)

[//]: # ()

[//]: # (The `uid` field is the only mandatory field. So a valid instance can be created with the `uid` alone. The other)

[//]: # (fields can be progressively added.)

[//]: # ()

[//]: # (The `displayName` and `preferences` are optional fields.)

[//]: # ()

[//]: # (The `tags` is a repeated &#40;array&#41; field. The schema does not convey any information about the cardinality of this)

[//]: # (field. That can be known only after parsing a raw nested data structure instance.)

[//]: # ()

[//]: # (The `preferences` adds a level of nesting. All the fields in it are also optional.)

[//]: # ()

[//]: # (This is the schema definition for a nested data structure. The syntax I have used here is the GraphQL Schema)

[//]: # (Definition Language. It may as well be a definition written in a programming language or using the syntax of an)

[//]: # (interface definition language like Thrift or Protocol Buffers. It does not matter. The relevant part is how we logically)

[//]: # (define the fields:)

[//]: # ()

[//]: # (- Is a field optional or mandatory?)

[//]: # (- Is this a repeated &#40;array&#41;?)

[//]: # (- What is the data type of the field?)

[//]: # (- What is the name of the field?)

[//]: # ()

[//]: # (<div class="record-container">)

[//]: # ()

[//]: # (```json)

[//]: # ({)

[//]: # (  "uid": "a1b2c3d4",)

[//]: # (  "displayName": "Alice Wonderland",)

[//]: # (  "tags": [)

[//]: # (    "reader",)

[//]: # (    "dreamer")

[//]: # (  ])

[//]: # (})

[//]: # (```)

[//]: # ()

[//]: # (```json)

[//]: # ({)

[//]: # (  "uid": "e5f6g7h8",)

[//]: # (  "displayName": "Chris Coder",)

[//]: # (  "tags": [)

[//]: # (    "developer",)

[//]: # (    "python",)

[//]: # (    "oss")

[//]: # (  ],)

[//]: # (  "preferences": {)

[//]: # (    "theme": "light")

[//]: # (  })

[//]: # (})

[//]: # (```)

[//]: # ()

[//]: # (```json)

[//]: # ({)

[//]: # (  "uid": "i9j0k1l2",)

[//]: # (  "displayName": "Bob The Builder",)

[//]: # (  "tags": [)

[//]: # (    "builder",)

[//]: # (    "diy")

[//]: # (  ],)

[//]: # (  "preferences": {)

[//]: # (    "theme": "dark",)

[//]: # (    "notifications": true)

[//]: # (  })

[//]: # (})

[//]: # (```)

[//]: # ()

[//]: # (</div>)

[//]: # ()

[//]: # ()
