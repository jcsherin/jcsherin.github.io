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

### Dremel Shredding

When shredding nested data the values of a path are stored together. This is similar to flat relational data where values of a single column are stored together. So a path maps to a column in storage. The distinction comes from the hierarchical structure of nested data. This structure is metadata which needs to be written together with the values, so that the shredded representation is lossless.

The first challenge is finding an efficient encoding which perfectly describes the original structure of the nested data.

The Dremel technique distills the structure of a nested data into two integer values termed as: __definition levels__ and __repetition levels__.

The definition level identifies the exact point in a nested path at which it terminates. If a path terminated early without reaching the leaf node then we know that the value for that instance is a `NULL`.

The repetition level is used for tracking the position of a primitive value or a nested object in an array.

For each value or `NULL` value obtained in record shredding a corresponding definition level and repetition level value is derived and stored separately. When reading the column values for a path, all three value are used in tandem to reassemble the original structure.



