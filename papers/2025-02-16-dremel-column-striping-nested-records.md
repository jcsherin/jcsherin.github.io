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

## Nested Columnar Storage

Challenges:

1. Lossless representation of record structure in columnar format
2. fast encoding
3. efficient reassembly

> Values alone do not convey the structure of a record. Given two values of a
> repeated field, we do not know at what ‘level’ the value repeated (e.g.,
> whether
> these values are from two different records, or two repeated values in the
> same
> record). Likewise, given a missing optional field, we do not know which
> enclosing records were defined explicitly. (p. 3)

## Repetition Levels

- Repeated fields are represented as a list of items

> It tells us at what repeated field in the field’s path the value has
> repeated. (p. 3)

- The simplest case is,
    - there are no nested repeated fields
    - list items are scalar values
- The repetition level then tells us,
    - the order of values in a list
    - it also encodes record boundaries so we know which record a list item
      belongs to
- In the nested repeated case a path may contain multiple repeated fields
- A path can terminate early if the list is empty, but this is not encoded
  by repetition levels
- How does that all look? (a concrete example for illustration)
- And repetition levels then tell us,
    - all the same as in the case of no nesting of repeated fields
    - and also which paths in the tree a list item belongs to

> In general though, determining the level up to which nested records exist
> requires extra information.

- Is that a forward reference to definition levels?

## Definition Levels

> Each value of a field with path p, esp. every NULL, has a definition level
> specifying how many fields in p that could be undefined (because they are
> optional or repeated) are actually present in the record.

> We use integer definition levels as opposed to is-null bits so that the data
> for a leaf field (e.g., Name.Language.Country) contains the information about
> the occurrences of its parent fields; (p. 3)

- A concrete example for illustration

## Encoding

## Splitting Records into Columns

## Record Assembly

## Not Particularly Interesting To Me (So maybe)

- Query Language
- Distributed Query Execution
