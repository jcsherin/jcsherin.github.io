---
title: 'Practical Hurdles In Crab Latching Concurrency'
date: 2025-08-21
summary: >
  For high performance, concurrency protocols must keep critical sections short by holding node latches for a minimal duration. For correctness, these protocols (like crab latching) enforce a strict order for acquiring and releasing latches. However, features like bi-directional range scans conflict with this strict ordering and are not covered by the base protocols. Supporting them requires workarounds to maintain safety without sacrificing performance, but this often comes at the cost of a more complex API.
layout: layouts/post.njk
draft: true
---

An implementation of the crab latching protocol enforces a strict top-down order for acquiring latches on a B+Tree node. This avoid deadlocks from ever occurring during concurrent operations. This is distinct from deadlock detection and resolution which is a runtime mechanism.

Deadlock avoidance is guaranteed by design through careful programming of critical sections in the code. Any mistakes here will result in deadlocks. Even worse, a data race which silently corrupts the index.

The main strength of a B+Tree index (compared to a hash index) is its unique capability to perform range scans. This is possible because all the entries are stored in key lexicographic order in the leaf nodes, and the leaf nodes themselves are connected to each other like a doubly-linked list. So scanning is efficient once you locate the starting leaf node. Scanning in ascending or descending key order is as simple as following the left or right sibling pointers.

This forwards or backwards movement during index scans violates the strict top-down ordering required for safety and correctness by the crab latching protocol.

A delete algorithm which implements a symmetrical tree rebalancing procedure requires acquiring an exclusive latch on either a left or right sibling for merging nodes. There is an equal chance of nodes merging left-right and right-left. This too violates the strict ordering requirement.

Therefore, an implementation has to come up with practical methods to avoid serial execution order and preserve concurrency. There is no formal verification of correctness via proof in these scenarios. We can improve our confidence in the implementation through engineering effort: code reviews, test suites, analyzers (ThreadSanitizer). Though the existence of data races cannot be ruled out, in practice this is sufficient for a robust and reliable implementation as evidenced by major OLTP systems.

<nav class="toc" aria-labelledby="toc-heading">
  <h2 id="toc-heading">Table of Contents</h2>
  <ol>
    <li>
      <a href="#an-overview-of-crab-latching">An Overview Of Crab Latching</a>
      <ul>
        <li><a href="#the-classic-deadlock-scenario">The Classic Deadlock Scenario</a></li>
        <li><a href="#crab-latching-is-efficient">Crab Latching is Efficient</a></li>
        <li><a href="#optimistic-concurrency">Optimistic Concurrency</a></li>
      </ul>
    </li>
    <li>
      <a href="#concurrent-index-scans">Concurrent Index Scans</a>
      <ul>
        <li><a href="#extension-for-concurrent-bi-directional-scans">Extension For Concurrent Bi-directional Scans</a></li>
        <li><a href="#deadlock:-lock-order-inversion">Deadlock: Lock Order Inversion</a></li>
      </ul>
    </li>
    <li><a href="#extension-for-symmetric-deletion">Extension For Symmetric Deletion</a></li>
    <li><a href="#concurrent-scans-can-miss-entries">Concurrent Scans Can Miss Entries</a></li>
  </ol>
</nav>

## An Overview Of Crab Latching

A thread can acquire either a shared read-only latch (s-latch) or an exclusive write latch (x-latch). A thread holding an x-latch blocks other threads until it completes its mutation of the node. Multiple reader threads holding an s-latch can concurrently access a node but are prevented from mutating the node itself. This is a basic single writer, multiple readers concurrency pattern.

In the crab latching protocol, during traversal a thread holding a latch on a B+Tree node attempts to acquire a latch on the next node. It must release the latch on the previous node only after it has acquired the latch on the next node. The name likely comes from how this mimics the latching movement of a crab.

### The Classic Deadlock Scenario

A deadlock occurs when two threads attempt to crab latch going in opposing directions. Consider a simple scenario with a parent Node-P and a child Node-C:

1. Thread-1 traverses _top-down_ from Node-P to Node-C to insert a new element.
2. Thread-2 moves _bottom-up_, from Node-C to update a pivot key in its parent Node-P.

Remember that in crab latching protocol, a thread has to first acquire the latch on the next node before it can release the existing latch it holds on the current node. Also, an x-latch allows access to only a single writer thread. Therefore both write operations will block each other indefinitely as they cross paths, creating a deadlock as shown below.

{% figure "img/btree-deadlock-wait.svg", "circular wait deadlock" %}
Figure 1. Thread-1 has acquired an exclusive write latch (x-latch) on Node-P and it is blocked waiting to acquire an x-latch on Node-C. Thread-2 already holds an x-latch on Node-C and it is blocked waiting to acquire an x-latch on Node P.
{% endfigure %}

A simple rule prevents this situation: acquire latches moving in only one direction. Since all B+Tree operations begin with a top-down traversals, enforcing a top-down order is the natural way to prevent deadlocks from occurring by design.

### Crab Latching is Efficient

> Latches are held only during a critical section, that is, while a data structure is read or updated. Deadlocks are avoided by appropriate coding disciplines, for example, requesting multiple latches in carefully designed sequences. Deadlock resolution requires a facility to rollback prior actions, whereas deadlock avoidance does not. Thus, deadlock avoidance is more appropriate for latches, which are designed for minimal overhead
> and maximal performance and scalability. Latch acquisition and release may
> require tens of instructions only, usually with no additional cache faults since a latch can be embedded in the data structure it protects.
>
> Goetz Graefe, [A Survey of B-Tree Locking Techniques (2010)]

[A Survey of B-Tree Locking Techniques (2010)]: https://15721.courses.cs.cmu.edu/spring2016/papers/a16-graefe.pdf

As Graefe notes, latches are designed for performance. By embedding a latch within each B+Tree node and holding it only for the brief duration of the operation, the protocol achieves fine-grained latching. This approach minimizes contention between threads and maximizes throughput for concurrent operations.

### Optimistic Concurrency

Crab latching is a pessimistic concurrency protocol. During a write, a node may split or merge, cascading changes up to the root. In this approach, a thread acquires an x-latch along the entire path from root to leaf. A single writer holding the root's x-latch blocks all other threads, effectively reducing concurrency to sequential execution.

The optimistic approach takes advantage of the fact that, in practice, node splits and merges are rare. In this "fast path", the writer thread traverses the tree using s-latches, only upgrading to an x-latch on the final leaf node to perform the modification.

This requires minimal overhead to check if a child node is "safe" or "unsafe". A node is unsafe if the write will cause a split or merge. If it encounters an unsafe node, the thread abandons the optimistic approach. It upgrades its latch on the parent to an x-latch and then acquires x-latches for the rest of the path segment. This "slow path" ensures rebalancing operations can proceed safely.

A notable benefit, even in the slow path, is that the x-latch path segment may be held on only a subsection of the tree, not extending all the way back to the root. This allows other operations to concurrently access different parts of the B+Tree.

## Concurrent Index Scans

The fundamental problem is that the concurrency models do not take into consideration B+Tree iterators. At the leaf node, traversing to a sibling uses the bi-directional links between leaf nodes. An ascending scan moves from left-right, while a descending scan moves from right-left. This conflicts with the safety property for avoiding deadlocks that traversals have a strictly enforced direction. Following the protocol exactly means the implementation can provide only one type of scan, either forward (ascending) or reverse (descending), but never both together.

```cpp
// A forward index scan
for (auto iter = index.Begin(); iter != index.End(); ++iter) {
  int key = (*iter).first;
  int value = (*iter).second;
}

// A reverse index scan
for (auto iter = index.RBegin(); iter != index.REnd(); --iter) {
  int key = (*iter).first;
  int value = (*iter).second;
}

/**
 * index.Begin()  : first element
 * index.End()    : one-past-the-last element
 * index.RBegin() : last element
 * index.REnd()   : one-past-the-first element
 */
```

### Extension For Concurrent Bi-directional Scans

A shared (read-only) or exclusive (write) latch blocks execution until the latch is acquired. For scans which are sideways traversals, we do not want to limit the traversal to any one direction. A blocking latch will create deadlocks if two concurrent scans proceed in opposite directions.

Therefore we need to use a non-blocking latch to prevent blocking and avoid deadlocks. A non-blocking latch will try to acquire a latch, and will return immediately.

```cpp
#include <shared_mutex>

class SharedLatch {
  // Blocking
  //
  // If another thread holds the latch, execution will block
  // until the latch is acquired.
  //
  // Used in insert, delete & search index operations
  void LockShared() {
      latch_.lock_shared();
  }

  // Non-blocking
  //
  // Tries to acquire a latch. If successful returns `true`,
  // otherwise returns `false`.
  //
  // Used in ascending & descending index scans
  bool TryLockShared() {
    return latch_.try_lock_shared();
  }

  private:
    // A wrapper around std::shared_mutex
    std::shared_mutex latch_;
}
```

Using `TryLockShared()` forces us rethink how a scan implementation should work if latch acquisition fails. In contrast, the concurrent insert, delete and search implementations are always expected to return a result matching its output type in the signature.

```cpp
// Returns `std::nullopt` if the key is not found in the index
std::optional<KeyType> MaybeGet(KeyType key);

// Returns `false` if key is a duplicate.
//
// Note: This prevents overwriting an existing key. The handling of
// duplicate keys is an implementation specific detail.
bool Insert(const KeyValuePair element);

// Returns `false` if the key does not exist.
bool Delete(const KeyType key_to_remove);
```

We can introduce a failure mode, where if latch acquisition fails during a scan we set its internal state to `RETRY`.

```cpp
enum IteratorState {
    VALID, INVALID, END, REND, RETRY
};
```

The implementation for forward scan which uses a non-blocking latch and retriable iterator looks like this:

```cpp
// Move forward one element at a time. If latch acquisition
// failed, then set internal state to `RETRY`.
void operator++() {
  // ...

  // The `TrySharedLock()` is a non-blocking read-only latch
  // which returns `true` or `false` immediately.
  if (!(current_node_->TrySharedLock())) {
    previous_node->ReleaseNodeSharedLatch();

    // Invalidates the iterator.
    // Sets internal state to `RETRY`.
    SetRetryIterator();
    return;
  }

  // ...

}


void SetRetryIterator() {
  // Resets internal state
  current_node_ = nullptr;
  current_element_ = nullptr;

  state_ = RETRY;
}
```

We have a working bi-directional iterator implementation which will avoid deadlocks, but it is not yet free from data races.

### Deadlock: Lock Order Inversion

The current API introduces a deadlock if the user initializes two iterators within the same scope, within the same thread.

```cpp
auto iter_forward = index.Begin();

// The second iterator creates a lock order inversion.
auto iter_reverse = index.RBegin();
```

A deadlock is prevented by enforcing a strict direction for latching. Any concurrent operation must therefore acquire a latch on an ancestor node before acquiring a latch on a descendant node (top-down traversal).

The iterator here holds a shared (read-only) latch on the leaf it points to. If that iterator remains alive while another operation begins a new top-down traversal from the root, we can get a deadlock.

(`Thread 1`): Creates a forward iterator, which holds a shared latch on a leaf node.

(`Thread 2`): Begins an insert in the pessimistic path acquiring an exclusive latch beginning at the root node all the way down to the parent of the same leaf node.

(`Thread 2`): Now attempts to acquire an exclusive latch on the leaf node but it blocks, waiting for the forward iterator to complete and release its shared latch on the leaf node.

(`Thread 1`): Creates a second reverse iterator, which is now blocking to acquire a shared lock on the root. It is waiting for the insert operation to release the exclusive latch.

This creates a deadlock, even though in implementation we enforced a strict ordering. We can ensure that this does not happen by ensuring that iterator lifetimes do not intersect each other, by introducing a local scope.

Most importantly, the shared latch is released at the end of the scope and therefore it also enforces a global ordering for latch acquisition and will not result in a deadlock like described above.

```cpp
{
  auto iter_forward = index.Begin();
}

// The lifetime of the first iterator ends before this
// scope starts. This guarantees that the shared latch
// on the leaf node is released before we start traversal
// from the root node.
{
  auto iter_reverse = index.RBegin();
}
```

To ensure safety, the latching protocol has to be enforced for concurrent operations within the same thread. Unfortunately, our non-blocking, retriable concurrent scan iterators has introduced an API which is easy for the user to incorrectly implement, and must come with warnings.

The pattern of creating two iterators in the same scope creates a lock-order-inversion within a single thread. While this does not create a deadlock by itself, because of shared latches, it creates the precondition for a deadlock with any concurrent operation which falls down the pessimistic concurrency path.

## Extension For Symmetric Deletion

A symmetric delete algorithm will proceed with a tree rebalancing operation after an underflow by attempting to merge with either the left sibling, and if that doesn't work with the right sibling at any level in the B+Tree. This violates our strict ordering principle for acquiring latches, because a merge can proceed in either direction.

(`Thread 1`): operating on Node B, needs to merge with its previous sibling, Node A. It holds an exclusive latch on B and tries to acquire an exclusive latch on A. (left to right)

(`Thread 2`): operating on Node A, needs to merge with its next sibling, Node B. It holds an exclusive latch on A and tries to acquire an exclusive latch on B. (right to left)

The creates a deadlock.

The fix is to enforce a strict one-way (say left-to-right) latch acquisition order. This is a delicate operation. Before locking the previous sibling A, the exclusive lock on the current node is released. The locks are reacquired in the correct order. First on the previous sibling, then on the current node.

```cpp

  auto maybe_previous = static_cast<InnerNode *>(parent)->MaybePreviousWithSeparator(key_to_remove);

  if (maybe_previous.has_value()) {

    /* ... */

    // Enforcing a strict left-to-right (one-way) ordering
    right->ReleaseNodeExclusiveLatch();
    left->GetNodeExclusiveLatch();
    right->GetNodeExclusiveLatch();

    /* ... */
  }
```

## Concurrent Scans Can Miss Entries

Earlier in the case of concurrent scans we saw how local reasoning of strict ordering is not sufficient, because of the extension. A global strict ordering has to be enforced to prevent unintended deadlocks from interaction of scans with other concurrent operations.

Now let us look at a scenario where both the extensions can interact in a manner which violates strict ordering and introduces lock order inversion bugs in subtle ways.

(`Thread 1`): A forward scan iterator has completed scanning entries in Node A, and is next going to process it's right sibling Node B.

(`Thread 2`): A delete operation has taken the pessimistic path, because Node B is going to underflow after removing this entry. So it decides to merge with left sibling Node A. It already holds an exclusive latch on Node B and has to acquire an exclusive latch on Node B.

If the forward scan operator releases its shared latch on Node A, before it acquires a shared latch on Node B, the following sequence of events can happen with the right timing of events.

(`Thread 1`): Releases latch on Node A before acquiring a latch on Node B.

(`Thread 2`): Release exclusive latch on Node B. Acquires exclusive latch on Node A. Acquires the exclusive latch back on Node B.

(`Thread 1`): Is blocked by the exclusive latch held on Node B, by the delete operation in `Thread 2`.

(`Thread 2`): Merges Node A and B together, and modifies the right sibling pointer to Node C.

(`Thread 1`): Finally acquires a latch on Node C which is the new right sibling on Node A, not realizing it missed node B completely.

If latches acquisition is not crafted carefully in the iterator code, the following situation creates a race condition which is very hard to even detect, or reproduce.

The fix here is to correctly implement crab latching in the iterator code. A shared lock should not be released before `TrySharedLock()` is attempted on the sibling node. Doing so results in race conditions which are impossible to track down. This requires careful programming discipline.
