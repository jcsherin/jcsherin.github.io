---
title: "Shredding Nested Data In Parquet"
date: 2025-07-23
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

## Shredding Nested Data in Parquet

OLAP database engines are able to execute aggregations, sorting and grouping over millions or billions of rows of data blazingly fast. The foundation for this extreme performance is a columnar storage format. Parquet is a columnar file format designed for efficient storage and fast retrieveal.

This performance is __not__ limited to flat relational data which is trivial to map into a columnar format. It also works for nested data structures. But it requires some clever encoding techniques to store nested data structures in a columnar format and be able to reconstruct them back.

The performance of querying flat relational data is generally significantly better than querying nested data structures in OLTP database engines for transactional workloads.

It may feel counter-intuitive that in OLAP engines the difference between querying flat and nested data is often minimal.

The magic lies in the Dremel shredding technique adopted by Parquet for storing nested data. Dremel is the query execution engine in Google BigQuery. This novel techinque for improving analytical query performance over nested data was introduced by Dremel to the rest of the world.

Shredding is the term now commonly used in the context of the Parquet format and proprietary columnar formats of OLAP databases for the process of converting nested data into columnar representation.

The shredded data can be reassembled back to its original form. By design it is also possible to partially reconstruct the nested data which is generally applicable for analytical workloads. So during analysis you can write queries which only target a handful of paths in the nested data.

This post will cover the various challenges inherent in mapping nested data structures to a columnar storage format. We will try to uncover the ingenious design decisions which makes shredding work perfectly in every case.

Before we see how shredding nested data leads to efficient storage and retrieval, we need to first understand how columnar format affects query execution performance. But if you are already familiar with columnar formats, you can safely skip the next section.

### Columnar Format

In a columnar format the values of a single column are stored together. The values are homogenous. They have the same datatype. So data-aware light-weight compression schemes like run-length encoding, dictionary encoding, delta encoding etc. can be applied to compact the data.

#### Data-aware compression

Consider a `timestamp` column which is in sorted order. It is stored as 64-bit integer in Parquet. Storing a million timestamp values alone will occupy 64 million bits or 8 million bytes (8 MB) of storage.

We can apply delta-encoding. The first timestamp (64-bits) is stored as reference. Then we store only the difference. Let us assume that the difference between timestamp values is small enough that it can be represented in 16-bits. If the unit of timestamp is milliseconds (ms), then 16-bits represents 65,536ms ~ 1 minute. So this will work if all the adjacent values are separated by less than a minute from each other.

With delta-encoding now it occupies only 16 million bits or 2 million bytes (2 MB) of storage. That is a 4x improvement in storage size over the uncompressed data.

#### Efficient Read I/O

In the SQL query if you specify the columns you want to read like: `SELECT column1, column2 * from t` because of columnar layout by design you only need to do disk I/O for reading exactly those columns: `column1` and `column2`. The other columns do not have to be scanned if they do not appear in the query. This improves I/O efficiency of queries and less data is read during query execution. This is another contributor to query execution speed.

#### Data Parallelism

If you execute an aggregation query: `SELECT SUM(column1) from t` it will scan all the values in `column1` from the Parquet file on disk. The implementation of sum will be looping over the values and accumulating the sum by adding one number after another until all the `column1` values are added up. The accumulated sum is returned as the query result.

Normally, the addition of two numbers in the CPU is a scalar instruction. But here the compiler can optimize this hot loop and convert the scalar sum into a vectorized sum instruction. In a vectorized sum instruction you can add more than two numbers in a single instruction.

If the CPU supports a 256-bit width register and we are adding together 32-bit integers. This register can hold eight 32-bit integers. The CPU can do 8 additions in parallel. To find the sum of 16 numbers you need 16 `ADD` operations. For large numbers vectorization will be dramatically more efficient in terms of the number of instructions executed.


### Direct Storage As Binary Blob

The direct method is to stored the nested data (probably JSON format) values in a column as a binary blob. Since the physical storage type is `BINARY` (a byte array) we cannot apply any light-weight compression schemes on this column.

The JSON path expression is the commonly used syntax for identifying the columns (paths) in the nested data structure that we want to query. This expression `$.user.posts[:3].tags[0:]` returns all the tags from the first three posts of a user object. The binary blob has to be deserialized to access the underlying JSON data. Even though we are only interested in the `tags` array, all the data must be deserialized to get to it.

Shredding solves this problem by transparently representing the nested data in storage as column values. The user can therefore can continue to query the nested data by using the fluent interface which refers to the path structure without considering how its represented internally in columnar format.

### Shredding Requires Data Type

Nested data is a tree data structure. A path in the tree maps to a column in storage. Similar to how column values in flat relational data is stored together, here values which have the same path are stored together.

In columnar format values of column are stored together because they share the same data type. So for shredding a tree path into column values, the data type of the tree path should be known.

The query performance and storage efficiencies in columnar format stems from storing homogeneous values which share a data type together in storage.

For shredding nested data into columnar format the schema should be defined. Shredding works only with strongly-typed data. If the types are unknown we can fallback to `BINARY` blob column storage forgoeing performance and storage efficiency.

### Schema & Data Model

The nested data model includes:
- `OPTIONAL`: This field can appear zero or one time.
- `REPEATED`: This field represents a list or array which contains zero or more elements.
- `group`: This is a nested struct field with contains one or more other fields.
- `REPEATED group`: This is an array of struct elements.

This is the schema definition for a nested `Contact`:
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


You can read this: `OPTIONAL BINARY phone_type (STRING)` as a field which has the name `phone_type` with the logical data type being a UTF-8 string, and the physical representation being binary. The field value may not always be present.

Here are a few valid examples of nested data of the `Contact` schema:

```text
{name: "Alice", phones: [
	{number: "555-1234", phone_type: "Home"},
	{number: "555-5678", phone_type: "Work"}]}
{name: "Bob"}
{name: "Charlie"}
{name: "Diana", phones: [
	{number: "555-9999", phone_type: "Work"}]}
{phones: [{number: null, phone_type: "Home"}]}
```

### The Dremel Encoding

In flat relational data storing the values together in columns suffice. It is not that simple with nested data where the structure of the nested data needs to be preserved.

The structure is metadata and it needs to be encoded and stored together with the shredded values. This is the fundamental principle of Dremel Encoding.

For each value shredded from nested data, the structure can be encoded using only two integer values. These two integers fully describe the structure and helps in reassembling the nested data back from its shredded form.

The two metadata values for describing the structure of nested data are: __definition level__ and __repetition level__.

#### Definition Level

This metadata maintains a count of all the optional and repeated fields which are present in a path. Let us look at a breakdown for the path: `phones.list.item.number`.

| field  | multiplicity | definition level|
|--------|--------------|-----------------|
| phones | OPTIONAL     | 1               |
| list	 | REPEATED     | 2               |
| item   | OPTIONAL		| 3               |
| number | OPTIONAL		| 4	 		   	  |

The definition level of a value shredded from this path can have a maximum value of 4.

| state                  | def | description |
|------------------------|-----|-------------|
| NULL                   | 0   | `phones` is not defined |
| []                     | 1   | `list` is empty |
| [NULL]                 | 2   | `item` is not defined |
| [{number: NULL}]       | 3   | `number` is not defined|
| [{number: "555-9999"}] | 4   | all the fields are defined |

> Note: In the SQL or DataFrame frontend the `list` and `item` fields need not be explicitly mentioned. This is a storage level concern. The user can directly use the expression `$.phones[*].number` to get all the phone numbers from the nested data.

#### Repetition Level

This metadata deals with only repeated fields. In the shredded form all we have is all the values of a path stored together in a column. The repetition level is critical for identifying to which nested data, the exact list and index of the shredded value. It is the final piece in being able reassemble complex nested data which often contains multiple repeated fields in a path.

In the `Contact` schema there is only a single repeated field. The repetition level is computed only for paths which contains at least one repeated field. So it applies to these paths in `Contact`:
- `phones.list.item.number` and,
- `phones.list.item.phone_type`

The max possible repetition level for both paths is 1.


### Putting it all Together

This is a partial projection of `Contact` values shown earlier:

```text
[0] { phones: [
		{number: "555-1234", phone_type: "Home"},
		{number: "555-5678", phone_type: "Work"}]}
	]}
[1] {}
[2] {}
[3] { phones: [
		{number: "555-9999", phone_type: "Work"}
	]}
[4] { phones: [
		{number: null, phone_type: "Home"}
	]}
```

Let us now take a look at the shredded form for the `number` column.

| `phones.list.item.number` | rep | def | description |
|---------------------------|-----|-----|-------------|
| "555-1234" | 0 | 4 | first phone number in first value |
| "555-5678" | 1 | 4 | next phone number in first value  |
| NULL       | 0 | 0 | `phones` field is not defined  |
| NULL       | 0 | 0 | `phones` field is not defined  |
| "555-9999" | 0 | 4 | first phone number in 4th value   |
| NULL       | 0 | 3 | first phone number is not defined in 5th value |

The first `Contact` contains two `number` items. The first item is identified by a repetition level of 0, and the remaining items have a repetition level of 1. The definition level is 4 which means all the fields are present for both values. This can be reassembled as following:

```text
{ phones: [
	{ number: "555-1234"},  // rep=0; def=4;
	{ number: "555-5678"},  // rep=0; def=4;
]}
```

The next two values are `NULL`, but this is never really physically stored. This is an optimisation which has a high impact when dealing with sparse nested data. Even though there is no actual `NULL` stored physically, it can be inferred from the definition level not being equal to 4. The definition level value tells us exactly where the path terminated, so we know how to recreate it. Since the definition level is zero in both cases, we can infer that `phones` field itself is not present in the nested data.

The next contact contains only a single item. We know this because there is no successor item with a repetition level of 1. The definition level is 4, so we know all the fields in the path are present. This is reassembled into:

```text
{ phones: [{number: "555-999"}] }
```

The final shredded value is also a `NULL`, but the definition level is 3. So we know that `phones.list.item` is present, but the final `number` field is not. So this is reassemble into:

```text
{ phones: [{number: null}] }
```

### Storage Optimizations

If the nested dataset is extremely sparse, there are going to be a lot `NULL` values present in the values array. But this is not an issue because `NULL` values are never stored in physical storage. When reading back the data this can be inferred from the definition level values. If the definition level is less than the max definition level for that path, then it can only mean that the path terminated early and therefore the value can only be `NULL`. This works out really well for sparse datasets as we incur no additional costs in storage.

You would have noticed by now that the definition level and repetition level values are dependent on the depth of the tree for any given path. If the nested data has a max depth of 7, then the level values will not exceed 7. In binary 7 is `0b111` which requires only 3-bits to encode. So both the definition and repetition level values can be bit-packed instead of representing them as 32-bit or 64-bit integers which will take up 4 to 8 bytes of storage per metdata value.

If a list contains a large number of elements, the first item will have a repetition level which identifies the beginning of the list. And the remaining elements will have the same repetition level. So if there are 1001 elements in the list, we can use run-length encoding to compress it further: `(M, 1), (N, 1000)`, where `M` and `N` represents the repetition level values and their corresponding run lengths.

If a path does not contain any repeated fields, we do not have to store the repetition levels for that column. Similarly if a top-level field is a required field (meaning it cannot be `NULL`) then the definition and repetitionl levels is going to be zero for all values. Here too we do not need to store both repetition levels or definition levels. This is because the top-level required field does not have any nesting.

The optimizations are significant in improving storage efficiency and reducing the amount of I/O required to read the shredded columns back from storage. We can have good things like shredding without trading off too much in storage space because of definition and repetition levels metadata.

### Querying Nested Data

So now we have the nested data in shredded form, let us see how to query it.

```text
SELECT *
  FROM read_parquet("contacts.parquet");
┌─────────┬──────────────────────────────────────────────────────────────────────────────────────┐
│  name   │                                        phones                                        │
│ varchar │                     struct(number varchar, phone_type varchar)[]                     │
├─────────┼──────────────────────────────────────────────────────────────────────────────────────┤
│ Alice   │ [{'number': 555-1234, 'phone_type': Home}, {'number': 555-5678, 'phone_type': Work}] │
│ Bob     │ NULL                                                                                 │
│ Charlie │ []                                                                                   │
│ Diana   │ [{'number': 555-9999, 'phone_type': Work}]                                           │
│ NULL    │ [{'number': NULL, 'phone_type': Home}]                                               │
└─────────┴──────────────────────────────────────────────────────────────────────────────────────┘
```

#### Flattening Nested `phones` list

The `UNNEST` function is useful here to unnest the `phones` list into one item per row. This yields a struct row item: `{'number': 555-1234, 'phone_type': Home}` which can be further unnested into its individual fields: `number` and `phone_type`.

The DuckDB SQL dialect supports [recursive unnesting](https://duckdb.org/docs/stable/sql/query_syntax/unnest.html#recursive-unnest).

```text
SELECT
    name,
    unnest(phones, recursive := true)
FROM
    read_parquet('contacts.parquet');
┌─────────┬──────────┬────────────┐
│  name   │  number  │ phone_type │
│ varchar │ varchar  │  varchar   │
├─────────┼──────────┼────────────┤
│ Alice   │ 555-1234 │ Home       │
│ Alice   │ 555-5678 │ Work       │
│ Diana   │ 555-9999 │ Work       │
│ NULL    │ NULL     │ Home       │
└─────────┴──────────┴────────────┘
```

#### Summarizing Phone Types

```text
SELECT
  phone.phone_type AS phone_type,
  COUNT(*) AS total
FROM
  read_parquet('contacts.parquet'),
  UNNEST(phones) AS t(phone)
GROUP BY
  phone_type
ORDER BY
  total DESC;
┌────────────┬───────┐
│ phone_type │ total │
│  varchar   │ int64 │
├────────────┼───────┤
│ Home       │     2 │
│ Work       │     2 │
└────────────┴───────┘
```

#### Summarizing Phones Per User

```text
SELECT
  name,
  len(phones) AS phone_count
FROM
  read_parquet('~/Documents/contacts.parquet')
ORDER BY
  phone_count DESC NULLS FIRST;
┌─────────┬─────────────┐
│  name   │ phone_count │
│ varchar │    int64    │
├─────────┼─────────────┤
│ Bob     │        NULL │
│ Alice   │           2 │
│ Diana   │           1 │
│ NULL    │           1 │
│ Charlie │           0 │
└─────────┴─────────────┘
```

### Point Lookups Involve Scanning and Decoding

An example is if we want to find the 1000th `phone_number`. The shredding flattens the `phone_number` into contiguous block of all phone numbers from all contacts. We cannot directly jump to the offset 1000 because `NULL` values are omitted from physical storage. Therefore the logical offset will not match the physical position in the stored column values.

The definition levels will have to be decoded and scanned from the beginning to map the logical offset 1000 to the physical offset in storage.

Consider a variation of this simple point lookup query but we also make it dependent on the `name` column where we want to find the 3rd non-null number for `Troy`.

Here we have to first scan the repetition levels to identify the beginning of `phone_numbers` for `Troy`. Then use the definition levels in tandem to skip any null values to find the target. Here the lookup process involves scanning and decoding both definition and repetition levels because of the dependency.

We are forced into sequential scans and full decoding of levels metadata to compute the result.

### All or Nothing: Selective Shredding

The alternatives are to store nested data as a binary blob and forgoe any of the native optimizations available in columnar formats. The other is to completely shred the nested data with a strongly-typed schema.

There is no in-between. You cannot choose to selectively shred only a fraction of the nested data and then store the rest as a binary blob. This is useful for semi-structured data like JSON where sometimes the same key may have different data types in the value.

This case is not supported in traditional shredding of nested data.

### Conclusion

Nested data has structure which needs to be preserved during shredding. The structural metadata is critical to being able to reassemble the full nested value or a partial projection.

Shredding requires knowing the column data type. So it works only if a schema definition is provided for the nested data. The data model contains optional, repeated and group fields which makes it expressive enough to model any real-world nested data.

Even though a single schema can lead to many variations in tree shapes of nested data instances, they can all be encoded with precision and fidelity during shredding by adding the structural metadata - definition & repetition levels. By interpreting the definition levels & repetition levels it is possible to reassemble the full or partial projection as required by the query.
