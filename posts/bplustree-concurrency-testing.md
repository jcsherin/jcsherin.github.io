---
title: 'Practical Hurdles In Crab Latching Concurrency'
date: 2025-08-21
summary: >
  Concurrency models (not limited to crab latching) prove that concurrent operations on a B+Tree index are possible with fine-grained latching. This contributes significantly to performance at the same time guaranteeing correctness and safety. However real-world implementations have to bridge a gap between the model and implementing useful features which directly conflicts with its safety properties. For instance, a naive bi-directional range scan implementation will introduce data races and create deadlocks.
layout: layouts/post.njk
draft: true
---

Contrary to popular belief, the concurrency models described in B+Tree papers like Yao-Lehman(1981), Bayer-Schkolnick(1977) etc. cover only a limited set of operations. It seems like at the time these papers where written, at the time when these papers were written was to prove that concurrent operations were possible on a B+Tree index.

> Latches are held only during a critical section, that is, while a data structure is read or updated. Deadlocks are avoided by appropriate coding disciplines, for example, requesting multiple latches in carefully designed sequences. Deadlock resolution requires a facility to rollback prior actions, whereas deadlock avoidance does not. Thus, deadlock avoidance is more appropriate for latches, which are designed for minimal overhead
> and maximal performance and scalability. Latch acquisition and release may
> require tens of instructions only, usually with no additional cache faults since a latch can be embedded in the data structure it protects.
>
> Goetz Graefe, "A Survey of B-Tree Locking Techniques" (2010)

The concurrency papers surveyed in [Graefe (2010)] demonstrated that concurrent operations on a B+Tree index are possible with fine-grained latching. Before that, the safest way to modify a B+Tree involved protecting the entire data structure with a single, exclusive latch of the root node which effectively forced concurrent operations into a serial execution order.

[Graefe (2010)]: https://15721.courses.cs.cmu.edu/spring2016/papers/a16-graefe.pdf

---

A correct, thread-safe B+Tree implementation is based on deadlock prevention, which makes deadlocks impossible by design. This is achieved through careful ordering protocol of how latches are acquired, held and released on B+Tree nodes. This is distinct from deadlock detection, which is a runtime mechanism found within the transaction manager to resolve deadlocks between transactions. Simply put, prevention for low-level data structures and detection for high-level transactions.

The strict ordering in an optimistic concurrency control (OCC), prevents deadlocks by design. In a B+Tree a new shared (read-only) or an exclusive (write) latch is acquired only ever going in one direction: from top to bottom. Since latches are never acquired from bottom to top direction, it is guaranteed that a deadlock can never happen during traversal.

<!-- A correct thread-safe B+Tree implementation is based on dead avoidance which is a compile-time guarantee of the implementation. This is distinct from deadlock detection, which is a found within the transaction manager. Deadlock avoidance for data structures, and deadlock detection for transactions. -->

<!-- The optimistic crab latching (or locking) is a concurrency protocol for making a B+Tree index implementation thread-safe. At the data structure level,  -->

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
