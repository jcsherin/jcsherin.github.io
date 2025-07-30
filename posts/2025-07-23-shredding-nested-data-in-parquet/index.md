---
title: "The Layman's Guide to Nested Record Shredding"
date: 2025-07-30
summary: >
  The query performance over nested data is almost identical to that of flat
  relational data. The magic lies in the ability to represent nested data in
  a columnar format by "shredding" it. This post covers the inherent
  challenges in shredding nested data into a columnar format which is not
  obvious from studying the shredding algorithm. We analyze the Dremel
  technique for record shredding which is adopted by the Parquet columnar
  file format.
layout: layouts/post.njk
draft: true
published: false
---

### Intro 1
In modern analytical query engines can compute complex aggregates by scanning billions of nested data structures in an instant.

This is the performance available on a single node (your laptop).

This works for SQL queries which contains many aggregates, groups and sort orders.

The performance gap between querying nested data and flat relational data is often negligible.

### Intro 2

The magic lies in a process known as "shredding".

At its core shredding is a process for breaking up nested data structures into a flat, columnar format.

- [x] ~~Insert diagram of a nested schema showing shredded column values collected at the leaf nodes. The user should be able to visualize that a path in the schema maps to a shredded column.~~

<figure>
  <img src="img/figure-1.svg" alt="Visualization of shredded column values of a nested
Contact schema">
  <figcaption>Figure 1: A high-level view of <code>Contact</code> nested data instances shredded into columnar format. Each shredded column corresponds to a unique path in the <code>Contact</code> schema: <code>name</code>, <code>phones.number</code> and <code>phones.phone_type</code>.</figcaption>
</figure>

The development of "shredding" was driven by the need for interactive analytics on very large (> 1 trillion rows) datasets containing nested data structures in the Dremel query engine at Google in the late 2000s.

The massive map-reduce jobs used to take multiple hours to complete even after running compute over thousands of servers in parallel.

The same computation now finished in under 10s when the nested data structure was encoded using the Dremel shredding technique.

It is an understatement to state that the performance impact was significant.

### Intro 3

The novelty of Dremel record shredding lay in directly representing nested data structures in a flat, columnar format.

This is achieved by introducing two key concepts: __definition levels__ and __repetition levels__.

This is an ingenious representation which manages to encode the original heirarchical structure of the nested data as two integer values.

- [ ] Show a shredded column, def, rep values and the values they map to. I am thinking it can show the encoding at the top and the values in a sequence below. An example which will fit this type of illustration well is a nested list which contains a list of numbers. They encoding representation is also terse and we can show the mapping from encoding to each inner list in the diagram.

### Intro 4

The Dremel encoding technique influenced the design of the next generation of file formats and proprietary storage engines of analytical systems which were being built in the early 2010s.

Apache Parquet is the most well known and open source implementation to directly adopt the principles of definition levels and repetition levels to represent nested data structure in its file format.

In Apache Arrow in-memory columnar format the validity bitmap and offset buffers are conceputally similar to the definition and repetition levels in Dremel encoding.

The widespread adoption of the principles from the Dremel encoding for nested data structures proves its effectiveness for analytical workloads.

### Intro Exit

Before we dive into shredding let us first see the problems which come up when we try to store and retrieve nested data without shredding.

### The Opaque Binary Blob Strawman

The direct method is to serialize a nested data structure and then store it as a sequence of bytes.

But now, every time you want to read it back you first load the entire bytes back into memory and deserialize it.

The cost of deserialization alone will dominate query execution time.

This is a query to find the top ten most popular tags.

Each row value is a nested data structure, and the absolute path to a list of tags is `user.post.tags`.

- ~~[x] Show the example values of this schema, instead of describing it to the reader.~~

```text
WITH all_tags(tag) AS (
	-- Defines a CTE (common-table expression) which runs exactly once
	-- and returns the unnested tags from every `user` nested data.
	SELECT UNNEST(user.post.tags) from users
)
SELECT tag, COUNT(*) as tag_count
  FROM all_tags
  GROUP BY tag
  ORDER BY tag_count DESC
  LIMIT 10
;
┌────────────────────┬───────────┐
│        tag         │ tag_count │
│      varchar       │   int64   │
├────────────────────┼───────────┤
│ javascript         │    120000 │
│ python             │     60000 │
│ sql                │     40000 │
│ java               │     30000 │
│ css                │     24000 │
│ aws                │     20000 │
│ docker             │     17142 │
│ data-science       │     15000 │
│ machine-learning   │     13333 │
│ kubernetes         │     12000 │
└────────────────────┴───────────┘
```

This a full table scan which has to deserialize every nested data structure just to access the `tags` list for computing the result.

There is no direct way to access the `tags` field and slice only the required list values from the byte array.

The binary blob column can be compressed using a general purpose compression algorithm like snappy, zstd to reduce the disk storage size.

The total throughput for decompressing the column values, deserializing it, and computing the count aggregate will not exceed the speed at which a fast NVME SSD can read data from disk.

The bottleneck here is never going to be disk I/O.

The size of each nested data structure may fall anywhere between a few kilobytes to a few megabytes.

When data structures are several megabytes, they exceed the size of the CPU's L3 cache.

Processing a large value will trigger a cache miss that stalls the processor, forcing it to wait for data from main memory which is orders of magnitude slower, leaving the CPU idle instead of performing actual work.

There is no option to selectively read only the parts required for computing the results of a query which almost certainly will fit into the lower L1/L2 cpu caches.

There is a deserialization cost associated with brining each value into memory before being able to begin performing any computation relevant to the query.

The majority of the work performed is not in service of computing the results of the query, but rather for accessing the values.

However the nested representation produced by shredding using the Dremel encoding technique manages to successfully avoid the pitfalls of the opaque binary blob representation.

### Nested Schema

Shredding depends on the schema to encode nested data into column values.

Here are a few valid instances of a `contact` nested data structure:

```protobuf
// All properties are defined
{ name: "Alice",
  phones: [
	{ number: "555-1234", phone_type: "Home" },
	{ number: "555-5678", phone_type: "Work" }]}

// The `phones` list is empty.
{ name: "Bob", phones: [] }

// The `phones` property is missing.
{ name: "Charlie" }

// The `names` property is missing.
{ phones: [{number: null, phone_type: "Home"}] }
```

This is the schema used for shredding the above values for storage in Parquet format.

```text
message contact {
  OPTIONAL BINARY name (STRING);
  OPTIONAL group phones (LIST) {
    REPEATED group list {
      OPTIONAL group item {
        OPTIONAL BINARY number (STRING);
        OPTIONAL BINARY phone_type (STRING);
      }
    }
  }
}
```

> Parquet is a self-describing file format which also stores the schema within the metadata located at the footer of the Parquet file.

An `OPTIONAL` field can be present or missing.

A `REPEATED` field is a list with zero or more elements.

Nesting is added using a `group` which contains one or more child fields.

The `LIST` is a nested data type which contains a nested field which defines the elements of the list.

Then there are the primitive data types which represents the valeus at the leaf nodes of the nested data.

As per the above schema there are three shredded columns in it which are:
- `name`
- `phones.list.item.number`
- `phones.list.item.phone_type`

This is a direct physical representation of shredded columns stored in the Parquet file.

The intermediate `list.item` is encapsulated from the user who queries this Parquet file as an internal implementation detail.

> Parquet adds the intermediate fields for list data structures to help remove any ambiguity between physical representation of data states in Dremel encoding like:
> - an empty list
> - a missing list
> - first element of list is `NULL`

The user who writes or reads the `contact` nested data to Parquet has the following logical view of paths in the `contact` nested data for data access:
- `name`
- `phones.number`
- `phones.phone_type`

> The decoupling of the logical and physical representations is a common technique used in database engineering to reduce code complexity and improve the internal implementation without introducing breaking changes to the public interface.

### Definition Levels

The definition level encodes the path structure of the nested data.

Let us see this in action for a path in the schema which contains both `OPTIONAL` and `REPEATED` fields like `phones.list.item.number`.

```text
message contact {
  OPTIONAL BINARY name (STRING);
  OPTIONAL group phones (LIST) {
    REPEATED group list {
      OPTIONAL group item {
        OPTIONAL BINARY number (STRING);
        OPTIONAL BINARY phone_type (STRING);
      }
    }
  }
}
```

The definition level count starts at zero for every path.

For each `OPTIONAL` and `REPEATED` field we encounter in the path from the root, we will increment the count by one.

| Multiplicity | Field Name | Δ | Definition Level |
|--|--|--|--|
| OPTIONAL | phones | +1 | 1 |
| REPEATED | list | +1 | 2 |
| OPTIONAL | item | +1 | 3 |
| OPTIONAL | number | +1 | 4 |


The maximum definition level value for this path is four.

Now let us see how this encodes various data states which are possible for this path.

| Value | Definition Level | Description |
|--|--|--|
| { phones: null } | 0 | `phones` is missing |
| { phones: [] } | 1 | `list` is empty |
| { phones: [null] } | 2 | `item` is missing |
| { phones: [{ number: null }] } | 3 | `number` is missing|
| { phones: [{ number: "555-1234"}] } | 4 | all fields are present |

### Repetition Levels

The repetition level encodes the list boundaries and the offset of an item in a list.

The repetition level count starts at zero for every path.

It is incremented only for `REPEATED` paths in a field.

| Multiplicity | Field Name | Δ | Repetition Level |
|--|--|--|--|
| OPTIONAL | phones | 0 | 0 |
| REPEATED | list | +1 | 1 |
| OPTIONAL | item | 0 | 1 |
| OPTIONAL | number | 0 | 1 |

The maximum repetition level for this path is therefore one.

```text
// Record [0]
{ phones: [
	{ number: "555-1234" },
	{ number: "555-5678" }]}

// Record [1]
{ phones: [] }

// Record [2]
{ phones: null }

// Record [3]
{ phones: [ { number: null } ]}
```

Let us see how the repetition levels are computed for the following values.

| Number (String) | Repetition Level| Definition Level | Description |
|--|--|--|--|
| "555-1234" | 0 | 4| Record 0:<br/> Start a new list |
| "555-5678" | 1 | 4| Record 0:<br/> Continuation of previous list |
| null | 0 | 2 | Record 1: <br/> Start a new list <br/> Definition level shows the list is empty |
| null | 0 | 1 | Record 2: <br/> Start a new list <br/> Definition level shows the `phones` field is missing|
| null | 0 | 3 | Record 3: <br/> Start a new list <br/> Definition level shows the `number` is missing |

The repetition level zero is a special case which always identifies the first list item and also the start of a new record.

The remaining items of this list will all have the same repetition level.

Therefore the repetition level zero also marks the record boundaries.

> Note: The start of a new list can have a non-zero repetition level. This occurs in a schema which contains more than one `REPEATED` field in a path.

The last three values are NULL and all of them have a zero repetition level.

We know a new record started, nothing more.

But by inspecting the definition levels we know exactly at which point the path terminated, and what the data state it represents.

### Data Layout

After shredding, the `NULL` values are not stored in physical storage.

This is primarily done to save space and an optimization for large nested schemas which are sparesely populated.

Missing values do not take up additional storage space.

The shredded column `phones.number` has the following physical layout.

```
values: ["555-1234", "555-5678"]
def: 	[4, 4, 2, 1, 3]
rep: 	[0, 1, 0, 0, 0]
```

The maximum definition level of this path is four.

Therefore we can infer that the last three values are null because the definition level is a value less than four.

### Comparison with Opaque Binary Blob

Can we selectively read only those columns which are required for the query in the shredded representation?

Yes.

Do we have to deserialize column values?

No, they are directly stored as a primitive type. There is no need to deserialize.

Will the column values, definition levels and repetition levels fit into the lower level L1/L2/L3 caches of the CPU?

Yes.

In the binary blob representation a lot of preprocessing work is required to access the right values before the query processing could even begin.

The nested representation using Dremel encoding is already in a columnar format, which allows direct access to columns which are used in the query.

The query execution can begin as soon as the column values are read from disk and materialized into memory.

### Limitation: Point Lookup Queries

A point lookup query is a highly-selective query which returns only a small set of rows.

Say we want to access the 55th non-null value in the nested representation of `phones.phone_number` which also contains `NULL` values.

To find the 55th non-null value we have to sequentially scan the definition levels array to skip the `NULL` values and map it to the correct physical row offset.

This sequential scan is inherently unavoidable.

We cannot jump directly to the value without first counting all the preceding non-null entries.

Neverthless though slightly inefficient this is not disastrous for query performance.

Columnar file formats like Parquet are capable of high-level pruning using metadata statistics which reduces the linear scanning to small chunks of the stored nested data.

#### Limitation: Heterogenous Column Values

The data type of the leaf node of a path, and therefore a shredded column is fixed.

If the value contains a different data type it is runtime error which is schema data type mismatch error.

This makes the Dremel encoding unsuitable for direct use with semi-structured data like JSON where often the same path can store values as different data types depending on some context.

- [x] ~~Show examples of JSON value where a key has a string type in some cases, and is an integer is few other cases.~~

```json
[
  {
    "name": "Alice",
    "phones": [
      { "number": 5551234, "phone_type": "Home" }
    ]
  },
  {
    "name": "Diana",
    "phones": [
      { "number": "555-5678", "phone_type": "Work" }
    ]
  }
]
```

In this JSON list the first `phones.number` value is an `Integer` data type and the second `phones.number` values is `String` data type.

The first value will produce a schema mismatch error and cannot be shredded.

A core limitation of shredding is that it only works with values which conforms to the defined schema.

A workaround here is to write custom code which will coerce the `Integer` value 5551234 into a formatted `String` value like "555-1234".

#### Limitation: No Selective Shredding Support

There are situations where majority of the input value is strongly-typed but then there is a fragment which contains semi-structured data which is contextual.

- [ ] Show examples of an analytics telemetry data like mixpanel where the user can define JSON tags in the client side to dispatch with the telemetry.

Unfortunately shredding is an all or nothing process.

It cannot be skipped for a selected few fields in a value.

A workaround is to define a field which squashes the semi-structured data into a `BINARY` blob field which is can then be shredded.

### Working Title: Parting Thoughts

- [ ] TODO

### References

- [ ] TODO
