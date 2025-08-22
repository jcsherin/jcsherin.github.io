---
title: 'Cache-Friendly B+Tree Nodes With Dynamic Fanout'
date: 2025-08-18
summary: >
  The B+Tree node contains an array of key-value entries whose size must be known at compile time. But if you hard-code the sizes, it prevents the user from configuring the node fanout when using the B+Tree data structure implementation. Furthermore, the node layout should be contiguous in memory, and should not contain an indirection to heap allocated memory for the key-value entries array field. This post demonstrates how the struct hack combined with dynamic memory allocation solves both the challenges elegantly.
layout: layouts/post.njk
draft: false
---

If a B+Tree node has to fit into the cpu cache, the memory layout has to be a single contiguous block. This memory layout requires low-level control which increases the complexity of the implementation, but is a necessary trade-off for higher performance.

**Memory Layout of a B+Tree Node**

```text
  +----------------------+
  | Node Header Metadata |
  +----------------------+
  | node_type            |  <-- Inner or Leaf Node
  | max_size             |  <-- Maximum Entries Per Node (fanout)
  | node_latch           |  <-- RW-Lock
  | end_ptr              |  <-- Past-the-end pointer
  +----------------------+
  | Node Data            |
  +----------------------+  <-- Data Elements (key-value)
  | data[0]              |
  | data[1]              |
  | data[2]              |  <-- end_ptr: if current size = 2
  | ...                  |
  | data[N]              |  <-- N = max_size - 1
  +----------------------+
```

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

The `std::vector` can be ruled out for storing the key-value elements of a B+Tree node. It is a pointer to the data, which is then stored in another block of memory on the heap. The memory layout is not contiguous. Therefore, we have to fallback to using C-style arrays for the data section of the B+Tree node.

The size of the array `data[N]` must be known at compilation time. The typical B+Tree implementation makes the fanout a user-configurable parameter. There is also no restriction that the fanout value should be similar for the inner nodes and leaf nodes.

This problem is not isolated to B+Tree node implementations. The pattern here is that you need low-level control over memory layout. The concerned object contains at a variable-length payload whose size is determined only at runtime.

It is not obvious how to escape this conundrum. Systems programmers for ages have run into this problem, and applied the same workaround over and over again. So much that, it has been standardized in C99, and has support in both C/C++ compiler implementations.

## The Struct Hack

The solution to the this problem is a technique which originates in C programming known as the struct hack. The variable-length member (implemented as an array) is placed at the last position in a struct. To satisfy the compiler a size of `1` is specified, so the array size is known at compilation time.

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

Then during runtime when you know `N` you allocate a single block of memory for both the struct and the `N` elements. To the compiler this is an opaque block, and it cannot provide any guarantees. But writing past the struct is safe because the variable-length member is in the last position.

```c
// The (N - 1) adjusts for the 1-element array in Payload struct
Payload *item = malloc(sizeof(Payload) + (N - 1) * sizeof(char))
```

This pattern is officially supported in the language since C99, called a [flexible array member].

[flexible array member]: https://en.wikipedia.org/wiki/Flexible_array_member

The C++11 standard includes the flexible array member.

> **Arrays of unknown bound**
>
> If `expr` is omitted in the declaration of an array, the type declared is "array of unknown bound of T", which is a kind of incomplete type, ...
>
> ```cpp
> extern int x[];      // the type of x is "array of unknown bound of int"
> int a[] = {1, 2, 3}; // the type of a is "array of 3 int"
> ```

The size can be omitted from the array declaration `element[]`. The code will compile.

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
  // entry in the `start_` array.
  KeyValuePair* end_;

  // Array containing key-value entries of unknown bound.
  KeyValuePair start_[];
};
```

If we had used `std::vector<KeyValuePair>` for the B+Tree node data it will result in indirection. The `std::vector` is a pointer to data which is stored in separate block of memory in another part of the heap memory. The memory layout of the B+Tree node is not going to be contiguous.

Trying to access the node data will be significantly slower because of the pointer indirection.

Following the pointer will increase cache misses, forcing the CPU to stall and wait for data to be fetched from main memory. A cache miss may cost hundreds of CPU cycles compared to just a few cycles for a cache hit. This latency cost adds up and is unacceptable for performance sensitive data structure implementations such as the B+Tree.

The struct hack or flexible array member prevents the pointer indirection. The node header and data are co-located in one continuous memory block. This layout is cache-friendly and will result in fewer cache misses.

## Raw Memory Buffer

This is the key step. The construction of the object has to be separate from its memory allocation. We cannot therefor use the standard `new` syntax which will attempt to allocate storage, and then initialize the object in the same storage.

Instead we will use the [placement new] syntax which will construct objects in the memory buffer we created. We know exactly how much space to allocate, which is information the standard `new` operator does not have in this scenario because of the flexible array member.

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

We have now successfully created a cache-friendly B+Tree node with a user-defined fanout configuration which is known only at runtime.

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
    for (KeyValuePair* element = start_; element < end_; ++element) {
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
+-----------------------+
| BPlusTreeNode Members |
| (Header)              |
+-----------------------+ <-- offset where the data buffer starts
| start_[0]             | <-- where the compiler thinks derived class
| start_[1]             |     member are written to
| ...                   |
| start_[N]             |
+-----------------------+ <-- end_
```

The raw memory we manually allocated is opaque to the compiler and it cannot safely reason about where the newly added members to the derived class are physically located. The end result is it will overwrite the data buffer and cause data corruption.

The workaround is to break encapsulation and add derived members to the base class so that the flexible array member is always in the last position. This is a significant drawback when we begin using flexible array members.

### Reinventing The Wheel

For the data array we now effectively end up reinventing `std::vector` utilities for insertion, deletion and iteration. This includes bound-checking to prevent buffer overflows.

This significantly raises the complexity, and maintenance burden of the implementation. We also have to make sure that our custom implementation is performant at par, with the standard library implementation.

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
  std::advance(end_, 1);

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
