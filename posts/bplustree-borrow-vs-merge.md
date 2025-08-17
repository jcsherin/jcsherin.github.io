---
title: 'A B+Tree Node Underflows: Merge or Borrow?'
date: 2025-08-16
summary: 'coalesce nodes or merge nodes, why does it matter?'
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
      <a href="#fixing-node-underflow">Fixing Node Underflow</a>
      <ul>
        <li><a href="#the-merge-first-approach">The Merge-First Approach</a></li>
        <li><a href="#the-borrow-first-approach">The Borrow-First Approach</a></li>
      </ul>
    </li>
  </ol>
</nav>

## Fixing Node Underflow

Each B+Tree node has a minimum degree (number of entries) constraint. A node underflow happens when the occupancy of node falls below the configured minimum degree. This is a critical knob to prevent index bloat, where over the lifetime of a B+Tree the nodes become progressively sparse because of constantly adding and removing key-value entries.

The strategy implemented for fixing node underflow is a classic case of choosing your trade-off. The merge-first approach optimizes for space, while the borrow-first approach optimizes for reducing write latency.

### The Merge-First Approach

When a leaf node underflow occurs after removing a key-value entry, the first attempt is to see if this leaf node can be merged with one of its neighboring sibling (left or right) nodes. The remaining key-value entries have to fit into one of the sibling nodes for this to work.

Since the node does not exist anymore, the key-pointer entry in the parent inner node has to be removed. If the parent inner node satisfies the minimum degree constraint, the write operation completes here. But if the removal causes the parent inner node to underflow, the process of rebalancing the tree has to continue. In rare situations this can propagate all the way back to the root node.

Now if the key-value entries will not fit into a sibling node without causing it to overflow, then we borrow enough entries to fix the underflow. This sometimes requires an update to the key in the parent inner node. The update ensures that search continues to work correctly after the borrow completes.

Crucially, a borrow operation will never trigger a cascading rebalance that propagates up the tree, which is the key difference from a merge.

The merge-first approach optimizes for space by trading-off write latency in pathological cases where deleting a single key-value entry from a leaf node triggers node modifications which propagates up along the tree path.

The bet here is that this worst-case scenario will happen rarely to be a problem in production workloads.

The B+Tree is primarily a disk-based data structure, and therefore a compact disk size is a desirable property because it minimizes disk I/O. A denser node with more key-value entries is better for cache locality.

Therefore by prioritizing node occupancy, this strategy relies on long-term read performance wins. This gain comes from reduced disk I/O and better cache locality. This comes with a small risk of occasional higher write latency for a few write operations.

### The Borrow-First Approach

<!-- When a leaf node underflows, it first tries to merge its remaining key-value entries with either the right or left sibling. The existence of bidirectional links between leaf nodes makes traversals to siblings easy. It is just a single step away. The entries in both nodes should not exceed the maximum degree of the B+Tree node. This is the opposite situation, a node overflow.

So if the key-value entries will fit into one of the sibling nodes on the left or right, we merge the nodes together. This is also a fast operation because the keys are already in sorted order. All the key-value pairs are appended to the existing entries in the sibling node.

Now the parent node contains a key-pointer entry which points to our original node which is outdated, and has to be removed. All the key-pointer entries to the right of the deleted entry has to be shifted by one position to the left to readjust the entries in the parent inner node. The average-case cost here is the same as the minimum degree, and the worst case is the maximum degree of the node.

If parent node did not underflow after removing the key-pointer entry, then we are done. But if an underflow happens here, we continue the same process as outlined above. In rare cases this can cascade all the way back to the root node.

Now if the density of key-value entries in the sibling nodes is already high, then a merge is not possible without overflow. In this case, borrowing a certain number of entries from either left or right sibling will fix the underflow issue. Since this involves no node removal, the parent inner node requires no modifications.

 -->

### Reduce Write Latency In Worst Case

## Database Examples

### What does MySQL (InnoDB) Implement?

### PostgreSQL Does Neither

## A Concrete Example

### An In-Memory Job Queue

## Key Takeaways

## Does It Actually Matter?
