---
title: "A B+Tree Node Underflows: Merge or Borrow?"
date: 2025-08-16
summary: "coalesce nodes or merge nodes, why does it matter?"
layout: layouts/post.njk
draft: true
---

A B+Tree node underflow condition happens when the number of key-value pairs it contains falls below a minimum threshold. This is typically set to 50% of the total node capacity. The delete operation which removes a key-value pair from an half-occupied node will result in a node underflow. 

A node underflow may in turn trigger an expensive tree rebalancing which involves removing a node separator key and pointer pair from the parent node. If the parent node is already at half capacity, this triggers another node underflow condition. In the worst case, this phenomenon can recurse all the way back to the roof of the B+Tree. 

The tree rebalancing is critical for maintaining the B+Tree invariants which guarantees its stable performance.

The B+Tree is made safe for concurrent readers & writers without sacrifcing performance by implementing an optimistic concurrency control (OCC) scheme. In this worst-case scenario, the optimistic approach has to be abandoned at the entire path from root to leaf node has to be exclusively locked until delete completes. This increases latency for all other B+Tree operations along this tree path for the duration of the delete.

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

