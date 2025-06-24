---
title: "The Clever Design in Apache Parquet For Storing Nested Data Efficiently In Columns"
date: 2025-06-24
summary: >
  Parquet uses the Dremel representation. Under the hood it derives two metadata columns for each value. This post
  is not simply about how to compute the metadata. It is complicated, but mechanical once you understand the core
  algorithm. The interesting part involves the challenges which are unique to nested data structures, and marvel
  at the elegance of the Dremel representation.
layout: layouts/post.njk
draft: true
---

Modern state of the art OLAP query execution engines can execute medium-high selectivity queries in under a second
on petabyte-scale(trillion rows) data. The foundation of this high-performance is columnar storage. Here is a
surprising truth: the performance is virtually identical whether you are querying nested data structures in Apache
Parquet or flat, relational tables. Many, mistakenly believe that nested data structures inherently leads to slower
query performance. This belief probably stems from a comparison with OLTP systems.

In the time before Parquet storing and querying nested data structures at scale was slow. The first system to fix
the performance gap was Google BigQuery powered by the Dremel engine. Parquet adopted the Dremel representation for
nested data structures in its own file format. This was not done as an afterthought. So Parquet came with ground up
support for storing and querying both nested data structures and relational, flat data.

The inner workings of the Dremel representation is complicated, but mechanical. There are many high quality posts
written about it. A curated set of links is available at the end of this blog post. So I am going to leave out the
how to compute the Dremel encoding for nested data structures out of this post.

Instead, the main thrust of this post is explore the inherent challenges with nested data structures that makes it
so hard to store them efficiently and improving query performance. This will help us better understand the ingenious
design choices in the Dremel representation.

## It starts with the Schema

We are storing bytes of ones and zeroes to disk. Now you want to read it back. But values of different types can end
up with the same byte representation on disk. So reading back the data depends on how you interpret the bytes. To
read the data that you wrote to disk, and read it back in its original form you need metadata which describes the
logical type of the value in physical storage. Without this metadata context bytes are ambiguous.

In database storage at a minimum three metadata values are required to describe any value.

1. The field (or column) name
2. The data type
3. Is this value nullable?

A relation (or table) is a collection of fields (or columns).

A schema describes a relation.

