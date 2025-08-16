---
title: "A B+Tree Node Underflows: Merge or Borrow?"
date: 2025-08-16
summary: "coalesce nodes or merge nodes, why does it matter?"
layout: layouts/post.njk
draft: true
---

When the number of entries in a B+Tree node falls below a minimum threshold, it is classified as a node underflow situation. The underflow threshold is typically set to 50% of the node fanout. A delete operation can therefore trigger a node underflow.

When a node underflow happens, there are two choices. Either merge it with a sibling node, or borrow few entries from a sibling. If the strategy is to first try merging, there is a non-zero chance that this causes the parent node to underflow. In the worst-case scenario, this can cascade all the way to the root node.

A B+Tree is a balanced data structure, meaning the path from root to every leaf node is exactly the same height. This is the property which guarantees stable performance. The merging of nodes is therefore tree rebalancing in action. It may be expensive, but unavoidable to maintain invariants.

One way of making a B+Tree thread-safe is to implement an optimistic latch crabbing scheme. This makes it possible for concurrent modifications to the B+Tree data structure to happen and still service readers without blocking (optimistic) most of the times. This is critical for performance.

But in the worst-case scenario, the optimistic approach has to be abandoned and pessimistic latch crabbing is used. This involves taking an exclusive latch for every node from the root to the leaf node. In a B+Tree the root node is the entry point for every operation and a contention hotspot. So for the duration of the delete and then rebalancing, the performance degrades significantly.

To be fair, this is a pathological case which should happen rarely and the cost of read/write access is amortized over the long term. But practically this is a weakness in B+Tree implementations which degrades performance.

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

