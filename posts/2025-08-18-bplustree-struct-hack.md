---
title: 'Cache-Friendly B+Tree Nodes With Dynamic Fanout'
date: 2025-08-18
summary: >
  The B+Tree node contains an array of key-value entries whose size must be known at compile time. But if you hard-code the sizes, it prevents the user from configuring the node fanout when using the B+Tree data structure implementation. Furthermore, the node layout should be contiguous in memory, and should not contain an indirection to heap allocated memory for the key-value entries array field. This post demonstrates how the struct hack combined with dynamic memory allocation solves both the challenges elegantly.
layout: layouts/post.njk
draft: false
---

For a high-performance B+Tree, the memory layout of each node must be a single contiguous block. This improves locality of reference, increasing the likelihood that the node's contents reside in the CPU cache.

In C++, achieving this means forgoing the use of `std::vector`, as it introduces a layer of indirection through a separate memory allocation. The technique involved inevitably increases the implementation complexity and introduces several non-obvious drawbacks. Nevertheless, this is a necessary trade-off for unlocking high performance.

```text
  +----------------------+
  | Node Metadata Header |
  +----------------------+
  | node_type            |  <-- Identify as Inner or Leaf Node
  | max_size             |  <-- Maximum Capacity (aka Node Fanout)
  | node_latch           |  <-- RW-Lock Mutex
  | iter_end_            |  <-- Past-the-end Pointer
  +----------------------+
  | Node Data            |
  +----------------------+  <-- Key-Value Elements Stored From Here
  | entries_[0]          |
  | entries_[1]          |
  | entries_[2]          |  <-- iter_end_ (when current size = 2)
  | ...                  |
  | entries_[N]          |  <-- N = max_size - 1
  +----------------------+  <-- iter_end_ (when node is full)
```

<figcaption>Fig 1. Memory Layout of a B+Tree Node</figcaption>

<nav class="toc" aria-labelledby="toc-heading">
  <h2 id="toc-heading">Table of Contents</h2>
  <ol>
    <li><a href="#challenges">Challenges</a></li>
    <li><a href="#the-struct-hack">The Struct Hack</a></li>
    <li><a href="#b+tree-node-declaration">B+Tree Node Declaration</a></li>
    <li><a href="#raw-memory-buffer">Raw Memory Buffer</a></li>
    <li>
      <a href="#the-price-of-fine-grained-control">The Price Of Fine-Grained Control</a>
      <ul>
        <li><a href="#manual-handling-of-deallocation">Manual Handling Of Deallocation</a></li>
        <li><a href="#adding-new-members-in-a-derived-class">Adding New Members In A Derived Class</a></li>
        <li><a href="#reinventing-the-wheel">Reinventing The Wheel</a></li>
        <li><a href="#hidden-data-type-assumptions">Hidden Data Type Assumptions</a></li>
      </ul>
    </li>
  </ol>
</nav>

## Challenges

Using `std::vector` for a B+Tree node's entries is a non-starter. A `std::vector` object holds a pointer to its entries which are stored in a separate block of memory on the heap. This indirection breaks up the memory layout, forcing us to fall back on C-style arrays for storing the node's entries.

This leads to a dilemma. The size of the array must be known at compilation time, yet we need to allow users to configure the fanout (the array's size) at runtime. Furthermore, the implementation should allow inner nodes and leaf nodes to have different fanouts.

This isn't just a B+Tree problem. It is a common challenge in systems programming whenever an object needs to contain a variable-length payload whose size is only known at runtime. How can you define a class that occupies a single block of memory when a part of the block has a dynamic size?

The solution isn't obvious, but it's a well-known trick that systems programmers have used for decades--a technique so common it was eventually standardized in C99.

## The Struct Hack

The solution to this problem is a technique originating in C programming known as the struct hack. The variable-length member (implemented as an array) is placed at the last position in the struct. To satisfy the compiler a size of `1` is specified, ensuring the array's size is known at compilation time.

```c
struct Payload {
  /* Header Section */
  // ...

  /* Data Section */

  // The variable-length member is in last position.
  // The size `1` satisfies the compiler.
  char elements[1];
}
```

At runtime, when the required size `N` is known, you allocate a single block of memory for the struct and the `N` elements combined. The compiler treats this as an opaque block, and provides no safety guarantees. However, accessing the extra allocated space is safe because the variable-length member is the final field in the struct.

```c
// The (N - 1) adjusts for the 1-element array in Payload struct
Payload *item = malloc(sizeof(Payload) + (N - 1) * sizeof(char))
```

This pattern was officially standardized in C99, where it is known as a [flexible array member].

[flexible array member]: https://en.wikipedia.org/wiki/Flexible_array_member

The C++11 standard formally incorporates the flexible array member, referring to it as an **array of unknown bound** when it is the last member of a struct.

> **Arrays of unknown bound**
>
> If `expr` is omitted in the declaration of an array, the type declared is "array of unknown bound of T", which is a kind of incomplete type, ...
>
> ```cpp
> extern int x[];      // the type of x is "array of unknown bound of int"
> int a[] = {1, 2, 3}; // the type of a is "array of 3 int"
> ```

This means that in C++, the size can be omitted from the final array declaration (e.g. `entries_[]`), and the code will compile, enabling the same pattern.

## B+Tree Node Declaration

Using the flexible array member syntax, we can now declare a B+Tree node with a memory layout which is a continuous single block in the heap.

```cpp
template <typename KeyType, typename ValueType>
class BPlusTreeNode {
public:
  using KeyValuePair = std::pair<KeyType, ValueType>;

private:
  // Node Header Members ... (elided)

  // Points to the memory location beyond the last key-value
  // entry in the `entries_` array.
  KeyValuePair* iter_end_;

  // Array containing key-value entries of unknown bound.
  KeyValuePair entries_[];
};
```

Using a `std::vector<KeyValuePair>` for the node's entries would result in an indirection. This immediately fragments the memory layout. Attempting to access a node entry will be noticeably slower. The pointer indirection is costly: following the pointer increases the risk of a cache miss, forcing the CPU to stall and wait for the cache line to be fetched from main memory.

A cache miss may cost hundreds of CPU cycles compared to just a few cycles for a cache hit. This cumulative latency is unacceptable for any high-performance data structure.

This technique avoids the pointer indirection and provides fine-grained control over memory layout. The node header and data are co-located in one continuous memory block. This layout is cache-friendly and will result in fewer cache misses.

## Raw Memory Buffer

This is the key step. The construction of the object has to be separate from its memory allocation. We cannot therefore use the standard `new` syntax which will attempt to allocate storage, and then initialize the object in the same storage.

Instead, we use the [placement new] syntax which only constructs an object in a preallocated memory buffer. We know exactly how much space to allocate, which is information the standard `new` operator does not have in this scenario because of the flexible array member.

[placement new]: https://en.cppreference.com/w/cpp/language/new.html#Placement_new

```cpp
// A static helper to allocate storage for a B+Tree node.
static BPlusTreeNode *Get(int p_fanout) {
  // calculate total buffer size
  size_t buf_size = sizeof(BPlusTreeNode) + p_fanout * sizeof(KeyValuePair);

  // allocate raw memory buffer
  void *buf = ::operator new(buf_size);

  // construct B+Tree node object in the preallocated buffer
  auto node = new(buf) BPlusTreeNode(p_fanout);

  return node;
}
```

The result is a cache-friendly B+Tree node with a fanout that can be configured at runtime.

## The Price Of Fine-Grained Control

To create an instance of a B+Tree node with a fanout of `256`, it is not possible to write simple idiomatic code like this: `new BPlusTreeNode(256)`.

Instead we use the custom `BPlusTreeNode::Get` helper which knows how much raw memory to allocate for the object including the data section.

```cpp
BPlusTreeNode *root = BPlusTreeNode<KeyValuePair>::Get(256);
```

### Manual Handling Of Deallocation

The destructor code is also not idiomatic anymore. When the lifetime of the B+Tree node ends, the deallocation code has to be carefully crafted to avoid resource or memory leaks.

```cpp
class BPlusTreeNode {

  void FreeNode() {
    // Call the destructor for each key-value entry.
    for (KeyValuePair* element = entries_; element < iter_end_; ++element) {
      element->~KeyValuePair();
    }

    // Call the node destructor
    this->~BPlusTreeNode();

    // Deallocate the raw memory
    ::operator delete(this);
  }
}
```

### Adding New Members In A Derived Class

Adding a new member to a derived class will result in data corruption. It is not possible to add new fields to a specialized `InnerNode` or `LeafNode` class.

```text
+----------------------+
| Node Metadata Header |
+----------------------+
| ...                  |
+----------------------+
| Node Data            |
+----------------------+ <-- offset where the data buffer starts
| entries_[0]          | <-- offset where the derived class members
| entries_[1]          |     will be written to, overwriting the
| ...                  |     entries
| entries_[N]          |
+----------------------+
```

<figcaption>Fig 2. Adding new members in a derived class will overwrite the <code>entries_</code> array.</figcaption>

The raw memory we manually allocated is opaque to the compiler and it cannot safely reason about where the newly added members to the derived class are physically located. The end result is it will overwrite the data buffer and cause data corruption.

The workaround is to break encapsulation and add derived members to the base class so that the flexible array member is always in the last position. This is a significant drawback when we begin using flexible array members.

```text
+----------------------+
| Node Metadata Header |
+----------------------+
| ...                  |
| low_key_             | <-- Inner Node: left-most node pointer
| left_sibling_        | <-- Leaf Node: link to left sibling
| right_sibling_       | <-- Leaf Node link to right sibling
+----------------------+
| Node Data            |
+----------------------+ <-- Flexible array member guaranteed to
| entries_[0]          |     be in the last position
| entries_[1]          |
| ...                  |
| entries_[N]          |
+----------------------+
```

<figcaption>Fig 3. Memory layout of base class with members necessary for the derived <code>Inner</code> and <code>Leaf</code> node implementations.</figcaption>

### Reinventing The Wheel

By using a raw C-style array, we effectively reinvent parts of `std::vector`, implementing our own utilities for insertion, deletion and iteration. This not only raises the complexity and maintenance burden but also means we are responsible for ensuring our custom implementation is as performant as the highly-optimized standard library version.

The engineering cost to make this implementation production-grade is significant.

### Hidden Data Type Assumptions

The following is an implementation for inserting a `KeyValuePair` element in the data array.

```cpp
bool Insert(const KeyValuePair &element, KeyValuePair *pos) {
  // The node is currently full so we cannot insert this element.
  if (GetCurrentSize() >= GetMaxSize()) { return false; }

  // Shift elements from `pos` to the right by one to make
  // place for inserting new `element`.
  if (std::distance(pos, End()) > 0) {
      std::memmove(
        reinterpret_cast<void *>(std::next(pos)), // Destination
        reinterpret_cast<void *>(pos), // Source
        std::distance(pos, End()) * sizeof(KeyValuePair) // Count
      );
  }

  // Insert element at `pos`.
  new(pos) KeyValuePair{element};

  // Bookkeeping
  std::advance(iter_end_, 1);

  return true;
}
```

The code above uses `std::memmove` to shift entries within the node. This introduces a hidden constraint, that the data types implementing the `BPlusTreeNode` should be POD (plain old data) types or trivially copyable.

The class interface is generic over the key and value types. But it doesn't convey this constraint to the user of the `BPlusTreeNode`.

```cpp
template <typename KeyType, typename ValueType>
class BPlusTreeNode {
public:
  using KeyValuePair = std::pair<KeyType, ValueType>;

  // ...
}
```

If we used `std::string` as either the `KeyType` or `ValueType` then calling the `Insert` method introduces undefined behavior. Copying a `std::string` object with `memmove` creates a shallow copy of its internal pointers. When the original string is destroyed, it will deallocate the memory, leaving the copied string with a dangling pointer.
