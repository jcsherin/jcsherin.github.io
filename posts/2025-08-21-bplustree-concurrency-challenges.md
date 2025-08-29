---
title: 'Practical Hurdles In Crab Latching Concurrency'
date: 2025-08-21
summary: >
  Concurrency models (not limited to crab latching) prove that concurrent operations on a B+Tree index are possible with fine-grained latching. This contributes significantly to performance at the same time guaranteeing correctness and safety. However real-world implementations have to bridge a gap between the model and implementing useful features which directly conflicts with its safety properties. For instance, a naive bi-directional range scan implementation will introduce data races and create deadlocks.
layout: layouts/post.njk
draft: false
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
        <li><a href="#how-do-deadlocks-happen">How Do Deadlocks Happen?</a></li>
        <li><a href="#how-are-deadlocks-prevented">How Are Deadlocks Prevented?</a></li>
        <li><a href="#efficient-fine-grained-crab-latching">Efficient Fine-Grained Crab Latching</a></li>
      </ul>
    </li>
    <li>
      <a href="#concurrent-index-scans">Concurrent Index Scans</a>
      <ul>
        <li><a href="#extension-for-concurrent-bi-directional-scans">Extension For Concurrent Bi-directional Scans</a></li>
        <li><a href="#deadlock:-lock-order-inversion">Deadlock: Lock Order Inversion</a></li>
      </ul>
    </li>
  </ol>
</nav>

## An Overview Of Crab Latching

> Latches are held only during a critical section, that is, while a data structure is read or updated. Deadlocks are avoided by appropriate coding disciplines, for example, requesting multiple latches in carefully designed sequences. Deadlock resolution requires a facility to rollback prior actions, whereas deadlock avoidance does not. Thus, deadlock avoidance is more appropriate for latches, which are designed for minimal overhead
> and maximal performance and scalability. Latch acquisition and release may
> require tens of instructions only, usually with no additional cache faults since a latch can be embedded in the data structure it protects.
>
> Goetz Graefe, "A Survey of B-Tree Locking Techniques" (2010)

[Graefe (2010)]: https://15721.courses.cs.cmu.edu/spring2016/papers/a16-graefe.pdf

### How Do Deadlocks Happen?

(`Thread 1`) Holds an exclusive (write) latch on `Node P`. It now wants to acquire an exclusive latch on it's child `Node C` for inserting an element.

(`Thread 2`) Already holds an exclusive latch on `Node C`. It is waiting to acquire an exclusive latch on it's parent `Node P`, so that the pivot key can be updated.

This creates a deadlock, where neither threads can make any progress. This could have been prevented if a strict ordering of the direction in which latches are acquired existed. A top-down ordering is better because all traversals begin at the root node to reach a leaf node.

### How Are Deadlocks Prevented?

The ordering requirement implies that only a parent node can acquire an exclusive latch on a child node. The implementation of `Thread 2` becomes an invalid state and should not be possible. So instead the exclusive latch on `Node P` is never released when traversing down to the child `Node C`. Since only one writer can hold the exclusive (write) latch at a time, this will not create a deadlock with `Thread 1`. The first thread to acquire the exclusive latch on `Node P`, will block the second thread.

Even though latches are light-weight and held only for a short duration of time, it is not a good idea to hold latches which are strictly not necessary. It will create contention in hot paths which is bad news for throughput. This is where the crab latching protocol shines with its efficiency.

### Efficient Fine-Grained Crab Latching

In crab latching, a child node is considered "unsafe" if the current operation will cause it to either split (overflow) or merge (underflow). An "unsafe" node may also end up modifying it's parent like `Thread 2`. So exclusive latches are held for the entire path segment, which contains "unsafe" nodes.

But we know that a node split or merge is going to be rare. More often an insert or delete will be local to a leaf node and does not have cascading changes which recurse back to the root node. Such nodes are considered to be "safe"

In the common scenario, when crab latching sees that a child node is "safe", it first acquires a shared (read-only) latch on the child node and then release it's shared (read-only) latch on the parent. Hence the "crab latching" terminology. A shared latch does not block other threads. An exclusive latch is only acquired on the leaf node at the time of an insertion or deletion.

This fine-grained latching is efficient and allows other concurrent readers and only makes writers to wait, improving overall throughput. This is the optimistic approach which assumes will happen in the general case. We fallback to the pessimistic approach only if leaf node will underflow or overflow after an operation.

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

<!-- ### Limitations, Safety & Correctness

- Livelock
- Exponential backoff, bounded retries
- Problems with restarting iteration at the current node
- Restarting from the root node is always going to be correct -->

<!-- ---

The concurrency papers surveyed in [Graefe (2010)] demonstrated that concurrent operations on a B+Tree index are possible with fine-grained latching. Before that, the safest way to modify a B+Tree involved protecting the entire data structure with a single, exclusive latch of the root node which effectively forced concurrent operations into a serial execution order.

A correct, thread-safe B+Tree implementation is based on deadlock prevention, which makes deadlocks impossible by design. This is achieved through careful ordering protocol of how latches are acquired, held and released on B+Tree nodes. This is distinct from deadlock detection, which is a runtime mechanism found within the transaction manager to resolve deadlocks between transactions. Simply put, prevention for low-level data structures and detection for high-level transactions.

The strict ordering in an optimistic concurrency control (OCC), prevents deadlocks by design. In a B+Tree a new shared (read-only) or an exclusive (write) latch is acquired only ever going in one direction: from top to bottom. Since latches are never acquired from bottom to top direction, it is guaranteed that a deadlock can never happen during traversal.

A correct thread-safe B+Tree implementation is based on dead avoidance which is a compile-time guarantee of the implementation. This is distinct from deadlock detection, which is a found within the transaction manager. Deadlock avoidance for data structures, and deadlock detection for transactions.

The optimistic crab latching (or locking) is a concurrency protocol for making a B+Tree index implementation thread-safe. At the data structure level,

The challenge of implementing a correct, concurrent B+Tree is that it becomes immediately apparent that the debugger will not help. If you try to step through the code, subtle timing issues can make the problem disappear and make it impossible to reproduce the concurrency bug. But the program will continue to crash, but not when you fire up the debugger.

_If you are in a hurry ship with a debugger!_

Deadlocks should be avoided. This is done through careful programming. The technique involves ensuring that latches (locks) are always acquired only in a single direction. A correct implementation will never deadlock.

> **Latches vs Locks Distinction** - In database management systems latches are used for protecting in-memory data structures. Locks are used for transactions, and protecting database pages, rows etc.
>
> -[ ] Include a summary table from the Goetze Grafe paper

In practice, this is harder for B+Tree implementations where the leaf nodes are linked together in both directions. And it is common to provide both forward and reverse iterators over the key space. This immediately conflicts with our deadlock prevention in code technique of only acquiring latches in one direction. The problem here is that if both a forward and reverse iterator are active at the same time, they can meet each other in the middle and deadlock. So the implementation becomes a little more challenging and we'll see how later in this post.

## Optimistic Crab Latching Protocol

The B+Tree is a self-balancing data structure which uses node splits, merging and redistribution of key-value entries to maintain its height property, and a minimum threshold of values per node.

The optimistic approach is the key to performance where an exclusive latch which locks out other readers and writers from a node is almost never acquired under "safe" conditions. A node is deemed safe if the write operation we are about to perform will not cause it to underflow. A condition where a node contains less key-value entries than the minimum defined threshold. So when inserting or deleting a key-value from a node, an exclusive write latch is required only for a short period of time.

But if the operation is "unsafe", if it will cause an underflow or overflow which triggers a tree rebalancing operation. Then the optimistic approach is abandoned, and traversal is restarted by acquiring latches starting at the root using a pessimistic approach.

Since multiple writers may not be modifying the same parts of the tree, this cost of the pessimistic latching is amortized away over the longer term.

## Iterator Implementation With Retry

Acquiring a latch is not guaranteed unlike for insert, delete or get operations. This is to prevent two iterators crossing each other, or an iterator blocking a tree balancing operation.

```cpp
void operator++() {
  // ...

  if (!(current_node_->TrySharedLock())) {
    previous_node->ReleaseNodeSharedLatch();
    SetRetryIterator();
    return;
  }

  // ...

}
```

Requires a non-blocking latching method which is not guaranteed to always succeed. The normal latching routines are blocking operations which wait for the mutex to be available. But we use them only for traversals where we can guarantee a top-bottom order of latching. It cannot be used for iterators which have both left-right (forward) and right-left (reverse) directions.

```cpp
bool TryLockShared() {
  return latch_.try_lock_shared();
}
```

The iterator internally has a `RETRY` state which is set if the iteration is blocked. This puts the operator responsible for handling the situation. The data structure typically will not implement any code to handle the `RETRY` situation internally.

```cpp
        void SetRetryIterator() {
            ResetIterator();
            state_ = RETRY;
        }

```

## Testing Strategy

The serial tests exercises every branch of code for node splits, merges and redistribution.

See [Insert Tests] and [Delete Tests].

[Insert Tests]: https://github.com/jcsherin/btree/blob/main/test/btree_insert_test.cpp
[Delete Tests]: https://github.com/jcsherin/btree/blob/main/test/btree_delete_test.cpp

The concurrency tests simulate real-world conditions and also uses key space randomization to induce failure in the crab latching protocol implementation.

See [Concurrent Tests].

[Concurrent Tests]: https://github.com/jcsherin/btree/blob/main/test/btree_concurrent_test.cpp

### Instrumenting Lock And Unlocks

The [latch implementation] adds compile-time instrumentation which is useful in development to find out mistakes as soon as they appear in a crab latching implementation.

[latch implementation]: https://github.com/jcsherin/btree/blob/main/src/shared_latch.h

The lock and unlock calls are instrumented using an atomic counter. When the lifetime of a node ends, it will release the latch in the destructor phase. In the latch destructor we have static assertions enabled through the `ENABLE_LATCH_DEBUGGING` compile-time flag, which will assert that the latch counts are zero at the time the latch is destroyed.

```cpp
#ifdef ENABLE_LATCH_DEBUGGING
~SharedLatch() {
  if (exclusive_lock_count_ != 0) {
      fprintf(stderr, "exclusive_lock_count_ is %d, expected 0\n", exclusive_lock_count_.load());
  }
  assert(exclusive_lock_count_ == 0);
  if (shared_lock_count_ != 0) {
      fprintf(stderr, "shared_lock_count_ is %d, expected 0\n", shared_lock_count_.load());
  }
  assert(shared_lock_count_ == 0);
}
#endif
```

This guarantees that for every latch which was acquired, it was also released. Simple, yet catches mistakes in implementation as soon as the tests are run.

### TSAN: Detecting Data Races

The earlier defensive programming techniques works for preventing deadlocks from every happening. But data races are different. It is a synchronization bug which happens if two threads access the same memory location concurrently and one of them modifies it. This is practically impossible to reproduce even through diligent testing, because they may happen only one in a million times.

This is where the ThreadSanitizer (TSAN) tool becomes invaluable in detecting and fixing data race conditions in code. It is capable of finding a race condition even if it doesn't fail in our test suite. This is such a useful tool for fixing bugs which are otherwise impossible to detect or even reproduce.

Here are a few bugs made possible by TSAN,

1. [Insertion deadlock](https://github.com/jcsherin/btree/blob/main/src/bplustree.h#L993-L1006)
2. [Split internal node with insufficient node pointers](https://github.com/jcsherin/btree/blob/aa2c6c22f1cc47cd4ea1aa3f98443e5140f6cc05/src/bplustree.h#L1023-L1048)
3. [Recheck conditions after entering critical section](https://github.com/jcsherin/btree/blame/aa2c6c22f1cc47cd4ea1aa3f98443e5140f6cc05/src/bplustree.h#L1191-L1203)
4. [Prevent data-race when updating B+Tree root](https://github.com/jcsherin/btree/blame/aa2c6c22f1cc47cd4ea1aa3f98443e5140f6cc05/src/bplustree.h#L1501-L1517)

## Limitations and Future Work

The above multi-layered approach is not perfect. There is still room for improvement.

The randomized tests when they fail, do not automatically shrink the failing test like a property testing framework. So it requires manual effort to narrow down the cause of failure through trial-and-error.

The current test workloads operates on data which is sequential, or randomized order. But real-world workloads mostly follow a Zipfian distribution. This will improve the quality of the test suite and prove the robustness of the implementation if such data distributions could be generated and tested.

Despite all this it is not possible to guarantee that all data races have been eliminated. But if we detect a new kind of failure mode in the future, the foundation is solid to add a regression test and prevent known modes of failure in the future.
 -->
