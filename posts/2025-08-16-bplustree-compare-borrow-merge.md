---
title: 'A B+Tree Node Underflows: Merge or Borrow?'
date: 2025-08-16
summary: >
  The stable logarithmic performance of a B+Tree is an outcome of its self-balancing property. When an inner node or leaf node falls below its minimum occupancy threshold during deletions, the tree rebalancing procedure is activated. The first choice is to either merge with a sibling, or borrow (redistribute) keys from a sibling. Neither choice is inherently better. Instead, the design decision signals the fundamental trade-off for that specific implementation. A survey of highly tuned OLTP systems reveals a design space that is far richer, more complex than this simple binary choice suggests.
layout: layouts/post.njk
draft: false
---

A B+Tree's stable algorithmic performance relies on a single invariant: the path from its root to any leaf is always the same length. However, a delete operation can cause a node to underflow (falling below its minimum occupancy), triggering a rebalancing procedure to maintain this critical invariant.

Modern B+Trees use fast, optimistic latching protocols which operate under the assumption that rebalancing happens rarely. The mere possibility of an underflow can force the rebalancing into the slow, pessimistic path, acquiring exclusive locks that stall other operations.

How the implementation decides to fix the underflow is therefore a critical design choice: merge with a sibling node to reclaim free space, or borrow keys from a sibling node to minimize the impact on write latency. Simply put, it's a classic trade-off between space and time. In this post, we will also explore how major OLTP systems expertly implement sophisticated strategies which go beyond this classic trade-off.

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
    <li>
      <a href="#real-world-implementations">Real-World Implementations</a>
      <ul>
        <li><a href="#background-merge-in-mysql-(innodb)">Background Merge In MySQL (InnoDB)</a></li>
        <li><a href="#do-nothing-strategy-in-postgresql">Do Nothing Strategy In PostgreSQL</a></li>
      </ul>
    </li>
    <li>
      <a href="#key-takeaways">Key Takeaways</a>
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

Here we prioritize minimizing data movement, and finishing the tree rebalancing as soon as possible. When a leaf node underflows, we check if one of the siblings (left or right) nodes can supply enough key-value entries without violating its own minimum degree constraint.

If that is possible, then the key-value entries are added to the underflow node to satisfy the minimum occupancy constraint. The key-value entries can be equally split between both nodes. The navigation key in the parent will have to be rewritten to represent the new key ranges after the borrow operation completes. At this point, the tree rebalancing is completed.

If a borrow succeeds, then it is guaranteed that a cascade rebalance will never occur. This is a good strategy for B+Tree implementations which needs to prioritize high write throughput. The minimal data movement, and mutable update in the parent node reduces the risk of unpredictable latency spikes which is possible during a merge operation.

On the other hand, a borrow may not be possible because it will violate the minimum degree constraint of either sibling nodes. In that scenario, we revert to merging with a sibling node. A merge is guaranteed to be possible because a borrow only fails when the sibling has no keys to spare. This means both nodes are at minimum capacity, so their combined entries will fit into a single node.

The borrow-first approach makes sense for B+Tree implementations which completely fits into main memory. The trade-off is predictable write throughput, but reduced cache locality. In a range scan operation the key ranges may have to be fetched from multiple nodes, which are not contiguous in main memory.

We gain predictable, faster writes by trading off a potential minimal hit to read performance and space efficiency.

## Real-World Implementations

### Background Merge In MySQL (InnoDB)

[MySQL] provides a configurable knob and implements merging two pages if the amount of data in a page falls below the threshold when deleting a row. But the docs do no make any indication regarding what happens if a merge is not possible. So MySQL does not seem to implement a merge-then-borrow, but rather a merge-only strategy to deal with node underflow.

> If the “page-full” percentage for an index page falls below the MERGE_THRESHOLD value when a row is deleted or when a row is shortened by an UPDATE operation, InnoDB attempts to merge the index page with a neighboring index page. The default MERGE_THRESHOLD value is 50, which is the previously hard-coded value. The minimum MERGE_THRESHOLD value is 1 and the maximum value is 50.

[MySQL]: https://dev.mysql.com/doc/refman/8.4/en/index-page-merge-threshold.html

It will make a best to attempt to merge two nodes asynchronously in the background which may not always succeed. This is evident from the metrics exposed by InnoDB.

```text
mysql> SELECT NAME, COMMENT FROM INFORMATION_SCHEMA.INNODB_METRICS
       WHERE NAME like '%index_page_merge%';
+-----------------------------+----------------------------------------+
| NAME                        | COMMENT                                |
+-----------------------------+----------------------------------------+
| index_page_merge_attempts   | Number of index page merge attempts    |
| index_page_merge_successful | Number of successful index page merges |
+-----------------------------+----------------------------------------+
```

The pathological case MYSQL tries to actively prevent is recurring merge-split behavior, a kind of trashing.

> If both pages are close to 50% full, a page split can occur soon after the pages are merged. If this merge-split behavior occurs frequently, it can have an adverse affect on performance. To avoid frequent merge-splits, you can lower the MERGE_THRESHOLD value so that InnoDB attempts page merges at a lower “page-full” percentage. Merging pages at a lower page-full percentage leaves more room in index pages and helps reduce merge-split behavior.

### Do Nothing Strategy In PostgreSQL

The [PostgreSQL] B+Tree implementation does not attempt to reclaim space after underflow. A node is only deleted after it becomes empty. So it does not implement neither the typical borrow-first nor merge-first strategy outlined above.

> We consider deleting an entire page from the btree only when it's become
> completely empty of items. (Merging partly-full pages would allow better
> space reuse, but it seems impractical to move existing data items left or
> right to make this happen --- a scan moving in the opposite direction
> might miss the items if so.) Also, we _never_ delete the rightmost page
> on a tree level (this restriction simplifies the traversal algorithms, as
> explained below).

But there is an exception. Removing an empty node which is the right-most child of a parent is explicitly prohibited. The reason for the prohibition is to prevent cascading updates which could potentially propagate up the tree because it involves a different parent node.

> To preserve consistency on the parent level, we cannot merge the key space
> of a page into its right sibling unless the right sibling is a child of
> the same parent --- otherwise, the parent's key space assignment changes
> too, meaning we'd have to make bounding-key updates in its parent, and
> perhaps all the way up the tree. Since we can't possibly do that
> atomically, we forbid this case.

When a node is deleted from the B+Tree, the free space is not immediately garbage collected. This is tightly coupled with the MVCC implementation. Until all transactions to which the node is visible are completed, it is not garbage collected and put back into circulation.

> Recycling a page is decoupled from page deletion. A deleted page can only
> be put in the FSM to be recycled once there is no possible scan or search
> that has a reference to it; until then, it must stay in place with its
> sibling links undisturbed, as a tombstone that allows concurrent searches
> to detect and then recover from concurrent deletions (which are rather
> like concurrent page splits to searchers).

<!-- https://gemini.google.com/app/0d2f20e0194501ab?hl=en-IN -->

[PostgreSQL]: https://github.com/postgres/postgres/blob/master/src/backend/access/nbtree/README

## Key Takeaways

The B+Tree implementations in PostgreSQL or MySQL (InnoDB) employs sophisticated strategies for reclaiming free space after deletions. The end goal is better performance for a wide range of workloads and different access patterns. MySQL (InnoDB) moves merging to happen asynchronously in the background. PostgreSQL avoids tree rebalancing and delays reuse of a deleted node (page). The trade-off in both cases is accepting index bloat and moving the burden to operational side of database management. This is not necessarily a bad thing, as it puts the operator in control. The implementation is also more complex, requiring an in-depth understanding of the engine quirks and harder for developers to modify.

Though OLTP systems are the primary users of a B+Tree data structure for indexes they are not the only ones. They are widely used in embedded key-value stores, search indexes and as library code in custom data management tools. These systems are not burdened by the complexity of interleaving the B+Tree implementation which is both correct and performant with the transaction manager and the recovery algorithms. In these cases, the above simpler strategies can unlock higher performance with lower code complexity for most use cases.

Finally, it depends on the workload and the specific access pattern. But knowing the tricks production-grade OLTP systems will come in handy in getting every ounce of performance out of the B+Tree data structures.
