---
title: "Dremel: Interactive Analysis of Web-Scale Datasets"
description: Column-striping of nested records
date: 2025-02-16
tags:
  - columnar storage
layout: layouts/post.njk
---

- What is Dremel?
- This paper introduces a novel column storage representation for nested records
- Nested records are non-relational
- What kind of data tends naturally to a nested representation?
  - Data-structures in programming languages
  - Messages exchanged by distributed systems
  - Structured documents
- Why is ad-hoc query difficult on nested records?
> Normalizing and recombining such data at web scale is usually prohibitive.
  - Normalizing -> splitting a record into multiple relations
  - Recombining -> multi join queries
- Dremel provides a SQL-like language to express ad-hoc queries
> Dremel uses a column-striped storage representation, which enables it to
> read less data from secondary storage and reduce CPU cost due to cheaper
> compression.

- The main contribution of this paper is a columnar storage for nested data.
- It describes algorithms for column-striping strongly-typed nested data and
  also reassembly of nested data from a given projection of columns
- The Dremel query execution can operate directly on the column-striped
  values to run ad-hoc aggregation queries over many nested records without
  any further restructuring of the data.
- How does this look?
  - record wise vs columnar representation of data
- What are the challenges?
  - How to preserve all the structural information?
  - Be able to reconstruct records from an arbitrary subset of fields
- Data Model
  - Strongly-typed nested records
  - Protocol Buffers syntax because that is used in Google
  - Atomic types and nested records
  - Records have one or more fields
    - Field has an optional multiplicity label
      - Optional fields maybe missing
      - Otherwise a field is required and should be present
    - Repeated fields may occur multiple times in a record
      - list of values
      - order of field occurrences (list item position) is significant

## Repetition Levels
## Definition Levels
## Encoding
## Splitting Records into Columns
## Record Assembly

## Not Particularly Interesting To Me (So maybe)
- Query Language
- Distributed Query Execution
