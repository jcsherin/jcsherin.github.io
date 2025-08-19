---
title: 'Cache-Friendly B+Tree Nodes With Dynamic Fanout'
date: 2025-08-18
summary: >
  The B+Tree node contains an array of key-value entries whose size must be known at compile time. But if you hard-code the sizes, it prevents the user from configuring the node fanout when using the B+Tree data structure implementation. Furthermore, the node layout should be contiguous in memory, and should not contain an indirection to heap allocated memory for the key-value entries array field. This post demonstrates how the struct hack combined with dynamic memory allocation solves both the challenges elegantly.
layout: layouts/post.njk
draft: true
---

The ideal memory layout for a B+Tree node is one in which both the header and data is stored contiguously in a block of memory. This rules out using an `std::vector` for storing the node data because of the pointer indirection. The `std::vector` is a pointer to the data, which is stored in a separate block of memory on the heap. We need fine-grained control over how memory is allocated for a B+Tree node.

Using an array instead presents another immediate challenge. A standard member array like `entries[N]` requires `N` to be a compile-time constant. Hard-coding commonly used combinations though plausible, is not an ideal workaround. B+Tree are often initialized with different configuration for leaf nodes and inner nodes. This prevents user-configurable node fanout.

The general problem is being able to have fine-grained control over memory layout when a member is a variable-length sequence. This pattern appears often in a format with a fixed header section followed by a variable-length data section. At first when you encounter this problem, it is non-obvious how to write a program which compiles without memory indirection.

<nav class="toc" aria-labelledby="toc-heading">
  <h2 id="toc-heading">Table of Contents</h2>
  <ol>
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

Instead if we use a `std::vector<KeyValuePair>` for the B+Tree node data. This stores a pointer in the struct, with the node data resident in a separate block of memory in another part of the heap.

Now accessing the node data can be significantly slower because of the pointer indirection. Following the pointer will increase cache misses, forcing the CPU to stall and wait for data to be fetched from main memory. A cache miss may cost hundreds of CPU cycles compared to just a cycles for a cache hit. This performance hit though is unacceptable if you need high-performance from your B+Tree implementation.

So we go through all this trouble to avoid pointer indirection and co-locate both the header and data of a B+Tree node in the same memory block. This layout is cache-friendly and improves B+Tree performance.

## Raw Memory Buffer

This is the key step. The construction of the object has to be separate from its memory allocation. For this we cannot use the standard `new` syntax which will attempt to allocate storage, and then initialize the object in this storage.

Instead we will use the [placement new] syntax which will construct objects in the memory buffer specified by us.

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

To create an instance, we can no longer simply write `new BPlusTreeNode(256)`. Instead we have to use our custom helper which knows how much raw memory to allocate for the object including the data section.

```cpp
BPlusTreeNode *root = BPlusTreeNode<KeyValuePair>::Get(256);
```

### Manual Handling Of Deallocation

We also need to handle object deallocation, so that when the lifetime of the object ends, the memory is freed to avoid memory leaks.

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

Adding a new member to a derived class will result in data corruption.

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

We lose RAII guarantees provided by the compiler and runtime bounds checking in `std::vector`.

We now bear the full responsibility of manually implementing, testing and maintaining the code for utility functions like insertion, deletion and iteration. The bounds-checking code to prevent buffer overflows requires verification.

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
