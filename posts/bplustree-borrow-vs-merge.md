---
title: "Trade-off between Merging & Borrowing in B+Tree deletes"
date: 2025-08-16
summary: "coalesce nodes or merge nodes, why does it matter?"
layout: layouts/post.njk
draft: true
---

This post is about the design choices and trade-offs involved when deleting a key-value from a B+Tree results in the node having a less than 50% occupancy (the node underflows).

<nav class="toc" aria-labelledby="toc-heading">
  <h2 id="toc-heading">Table of Contents</h2>
  <ol>
    <li>
      <a href="#b-tree-modifications">B-Tree Modifications</a>
      <ul>
        <li><a href="#two-ways-to-resolve-node-undeflow">Two Ways to Resolve Node Undeflow</a></li>
        <li><a href="#reduce-write-latency-in-worst-case">Reduce Write Latency In Worst Case</a></li>
      </ul>
    </li>
    <li>
      <a href="#database-examples">Database Examples</a>
      <ul>
        <li><a href="#what-does-mysql-innodb-implement">What does MySQL (InnoDB) Implement?</a></li>
        <li><a href="#postgresql-does-neither">PostgreSQL Does Neither</a></li>
      </ul>
    </li>
    <li>
      <a href="#a-concrete-example">A Concrete Example</a>
      <ul>
        <li><a href="#an-in-memory-job-queue">An In-Memory Job Queue</a></li>
      </ul>
    </li>
    <li><a href="#key-takeaways">Key Takeaways</a></li>
    <li><a href="#does-it-actually-matter">Does It Actually Matter?</a></li>
  </ol>
</nav>



## B-Tree Modifications


### Two Ways to Resolve Node Undeflow

### Reduce Write Latency In Worst Case



## Database Examples


### What does MySQL (InnoDB) Implement?

### PostgreSQL Does Neither



## A Concrete Example

### An In-Memory Job Queue

## Key Takeaways

## Does It Actually Matter?

