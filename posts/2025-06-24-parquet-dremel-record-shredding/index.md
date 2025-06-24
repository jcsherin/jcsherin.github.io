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


