---
title: 'A B+Tree Node Underflows: Merge or Borrow?'
date: 2025-08-16
summary: >
  The stable logarithmic performance of a B+Tree is an outcome of its self-balancing property. When an inner node or leaf node falls below its minimum occupancy threshold during deletions, the tree rebalancing procedure is activated. The first choice is to either merge with a sibling, or borrow (redistribute) keys from a sibling. Neither choice is inherently better. Instead, the design decision signals the fundamental trade-off preferred by the specific implementation. A survey of major OLTP systems reveals a design space that is far more nuanced than this simple binary choice suggests.
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
      <a href="#in-oltp-systems">In OLTP systems</a>
      <ul>
        <li><a href="#background-merge-in-mysql-innodb">Background Merge In MySQL InnoDB</a></li>
        <li><a href="#do-nothing-strategy-in-postgresql">Do Nothing Strategy In PostgreSQL</a></li>
      </ul>
    </li>
    <li>
      <a href="#key-takeaways">Key Takeaways</a>
    </li>
  </ol>
</nav>

## Fixing Node Underflow

A node underflow happens when the used space (or occupancy) within a node falls below a minimum threshold. This is a possibility after removing an entry from the node. A viable solution is to do nothing. By doing nothing, a tree balancing procedure is never activated. The major downside though is index bloat. A failure to garbage collect the unused memory results in the nodes becoming progressively sparse as entries continue to be added and removed.

> In contrast, a node overflow will always trigger a tree rebalancing because it creates a new node whose reference needs to be inserted in the parent node. Here, doing nothing is not even an available option.

The two basic strategies for fixing node underflow both involve merging and borrowing. They differ by which operation is executed first: a merge or a borrow. The merge-first approach prioritizes immediate garbage collection of unused space. It trades-off write speed for more efficient utilization of space. In contrast, the borrow-first approach prioritizes write throughput through redistribution of keys within existing nodes avoiding a merge whenever possible, trading-off long term space-efficiency for short-term write speed.

### The Merge-First Approach

For a merge to work, the combined entries in the underflowing node and the target sibling node must fit within a single node. After merging two nodes into one, the memory belonging to the underflowing node can be garbage collected.

> An efficient implementation will avoid allocation for a new node, and reuse the memory of the sibling node.

The problem with the merge-first approach is that in the worst-case it can recursively cascade all the way back to the root of the B+Tree. In practice though this should happen rarely. The reason for this behavior is that merge eliminates an existing node, and its reference has to be removed from the parent inner node. Removing an entry from a node, has the potential however small to again cause an underflow.

What if the combined entries will not fit into a single node?

Then we fallback to borrowing entries from a sibling to fix the underflow. This will not cause a cascading rebalance, as there is no change in nodes, only a redistribution of keys.

> _Disclaimer_: The following is a simplified view of how the B+Tree algorithm and concurrency protocols interleave with each other. An implementation in code is more complex.

In optimistic (crab latching) concurrency, if removing an entry will cause a node to underflow it is categorized as an "unsafe" node. The optimistic approach is abandoned and we restart traversal back from the root. It involves acquiring a chain of exclusive latches along the search path to safely complete the operation. This significantly slower path blocks other readers and writers from accessing the latched nodes until the rebalancing operation completes.

The merge-first approach compresses nodes and therefore more keys are stored within the minimum amount of nodes. This occupies less space on disk and therefore requires less read I/O to be performed. For range scans then a minimum number of nodes needs to be read from disk for a given key range. The higher node density also results in better cache locality and this further improves compute performance.

### The Borrow-First Approach

For borrowing to work, removing the entries from the sibling node must not result in an underflow. Since there is no change in nodes the tree rebalancing is guaranteed not to cascade and therefore completes faster. If borrowing will cause a node underflow, we fallback to merging nodes.

The concurrency aspects are similar to the merge-first approach. In comparison, it is reasonable to expect that the exclusive latches held on the nodes in the search path segment, will be for a relatively shorter duration. It is still orders of magnitude slower than the optimistic approach.

This approach prioritizes faster writes and predictable latency by avoiding merging nodes unless strictly necessary. The downside is that node density is lower. The range scans now require more node I/O because the same key range is now spread over a wide span of leaf nodes.

## In OLTP Systems

The discussion so far is confined to a stand-alone thread-safe B+Tree implementation. We are looking at behavior and performance at the data structure level. In major OLTP database management systems, the B+Tree index is also tightly integrated with the transaction manager, write ahead log and recovery manager. So the scope of the design decisions are not limited to the data structure level, rather how it impacts the overall systems performance.

### Background Merge In MySQL InnoDB

In MYSQL's InnoDB, node underflow is handling through merging. Rather than performing an online tree balancing, it is offloaded as a separate asynchronous process in the background. The minimum occupancy of a node is configurable through the [MERGE_THRESHOLD] parameter.

> If the “page-full” percentage for an index page falls below the MERGE_THRESHOLD value when a row is deleted or when a row is shortened by an UPDATE operation, InnoDB attempts to merge the index page with a neighboring index page. The default MERGE_THRESHOLD value is 50, which is the previously hard-coded value. The minimum MERGE_THRESHOLD value is 1 and the maximum value is 50.

[MERGE_THRESHOLD]: https://dev.mysql.com/doc/refman/8.4/en/index-page-merge-threshold.html

You can monitor the background merge by querying the following InnoDB metrics:

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

Merging entries from a sibling reduces the unused space remaining within a node. If new insertions land on this node, it can immediately overflow. The overflow forces a node split which requires a new node allocation and redistribution of entries. Now both nodes are half-full, and a delete from either node can tip another underflow creating a cycle of merging and splitting. This thrashing merge-split behavior is terrible news for index performance.

> If both pages are close to 50% full, a page split can occur soon after the pages are merged. If this merge-split behavior occurs frequently, it can have an adverse affect on performance. To avoid frequent merge-splits, you can lower the MERGE_THRESHOLD value so that InnoDB attempts page merges at a lower “page-full” percentage. Merging pages at a lower page-full percentage leaves more room in index pages and helps reduce merge-split behavior.

What happens if you over-tune the `MERGE_THRESHOLD` knob for your workload?

> A MERGE_THRESHOLD setting that is too small could result in large data files due to an excessive amount of empty page space.

This results in index bloat, which directly harms read efficiency. The same key space instead of being densely packed, is now spread over a larger number of sparse nodes, requiring more I/O for range scans, and takes up more buffer pool capacity to keep the index in memory.

### Do Nothing Strategy In PostgreSQL

In PostgreSQL deleting an index entry happens in two separate phases: a logical deletion followed by a physical deletion. The logical deletion creates a tombstone marker, and is lightweight. The physical deletion is more expensive, and happens in the background.

When a node underflows, PostgreSQL does nothing to rectify the situation. The only time it attempts to reclaim space is when all the entries are removed from a leaf node and it becomes empty.

> We consider deleting an entire page from the btree only when it's become
> completely empty of items. (Merging partly-full pages would allow better
> space reuse, but it seems impractical to move existing data items left or
> right to make this happen --- a scan moving in the opposite direction
> might miss the items if so.)

There is an exception to this rule. PostgreSQL will never delete the right-most node of parent, even if it becomes empty. Removing the right-most node will have to be followed by transferring the key space used for navigation to the next or previous parent. Since we do not hold latches on those nodes yet, they will have to be freshly acquired. All of this is avoided by sticking to this rule, and it simplifies the implementation.

> To preserve consistency on the parent level, we cannot merge the key space
> of a page into its right sibling unless the right sibling is a child of
> the same parent --- otherwise, the parent's key space assignment changes
> too, meaning we'd have to make bounding-key updates in its parent, and
> perhaps all the way up the tree. Since we can't possibly do that
> atomically, we forbid this case.

The deletion of a node is also separated into logical and physical phases. A logical deletion happens first which marks a tombstone. A physical deletion, which reclaims the space for reuse is only performed when this node is not visible to any active transactions.

> Recycling a page is decoupled from page deletion. A deleted page can only
> be put in the FSM to be recycled once there is no possible scan or search
> that has a reference to it; until then, it must stay in place with its
> sibling links undisturbed, as a tombstone that allows concurrent searches
> to detect and then recover from concurrent deletions (which are rather
> like concurrent page splits to searchers).

[Source]\: `src/backend/access/nbtree/README`

[Source]: https://github.com/postgres/postgres/blob/master/src/backend/access/nbtree/README

## Key Takeaways

The B+Tree implementations in PostgreSQL or MySQL (InnoDB) employs sophisticated strategies for reclaiming free space after deletions. The end goal is better performance for a wide range of workloads and different access patterns. MySQL (InnoDB) moves merging to happen asynchronously in the background. PostgreSQL avoids tree rebalancing and delays reuse of a deleted node (page). The trade-off in both cases is accepting index bloat and moving the burden to operational side of database management. This is not necessarily a bad thing, as it puts the operator in control. The implementation is also more complex, requiring an in-depth understanding of the engine quirks and harder for developers to modify.

Though OLTP systems are the primary users of a B+Tree data structure for indexes they are not the only ones. They are widely used in embedded key-value stores, search indexes and as library code in custom data management tools. These systems are not burdened by the complexity of interleaving the B+Tree implementation which is both correct and performant with the transaction manager and the recovery algorithms. In these cases, the above simpler strategies can unlock higher performance with lower code complexity for most use cases.

Finally, it depends on the workload and the specific access pattern. But knowing the tricks production-grade OLTP systems will come in handy in getting every ounce of performance out of the B+Tree data structures.
-->
