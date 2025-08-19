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
    <li>
      <a href="#the-struct-hack">The Struct Hack</a>
      <ul>
        <li><a href="#flexible-array-member">Flexible Array Member</a></li>
        <li><a href="#preallocated-memory-buffer">Preallocated Memory Buffer</a></li>
      </ul>
    </li>
    <li>
      <a href="#implementation-trade-offs">Implementation Trade-offs</a>
      <ul>
        <li><a href="#manual-inserts-with-stdmemmove">Manual Inserts With <code>std::memmove</code></a></li>
        <li><a href="#deallocation-and-the-lack-of-raii">Deallocation And The Lack of RAII</a></li>
        <li><a href="#opaque-layout-breaks-encapsulation">Opaque Layout Breaks Encapsulation</a></li>
        <li><a href="#hidden-issues">Hidden Issues</a></li>
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

### Flexible Array Member

### Preallocated Memory Buffer

## Implementation Trade-offs

### Manual Inserts With `std::memmove`

### Deallocation And The Lack of RAII

### Opaque Layout Breaks Encapsulation

### Hidden Issues

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

    // Array containing key-value entries
    KeyValuePair start_[0];
};
```

A B+Tree is a disk-based data structure, where a node is a contiguous region of memory both on-disk and in-memory. The number of entries stored in a node (fanout) should be a user configurable value. This memory layout improves cache efficiency and improves performance.

The alternative is to store a pointer to a vector of key-value entries which is located somewhere in heap memory. This is slower by an order of magnitude because of the pointer indirection. Following the pointer often results in a cache miss, forcing the CPU to stall and wait for data to be fetched from main memory.

So instead we use an array. But the code will not compile unless the size of the array is known at compile time. But if we hard-code the array size, then the node fanout is not user-configurable value anymore.

So we want a cache-friendly memory layout and at the same time be able to decide the node fanout at runtime, rather than at compile time.

The solution is to use the struct hack, combined with dynamic memory allocation.

Here is a B+Tree node definition. (The node header details are elided for clarity).

```cpp
template <typename KeyType, typename ValueType>
class BPlusTreeNode {
public:
	using KeyValuePair = std::pair<KeyType, ValueType>;

private:
	// .. Header containing metadata fields

    // Points to offset past the last element in `start_`
    KeyValuePair* end_;

    // Array storage for key-value entries
    KeyValuePair start_[0];
};
```

You may have noticed that the key-value entries are declared as an array with zero elements, and it is the final member. This may look weird, but this code will compile without errors.

```cpp
KeyValuePair start_[0];
```

This is just a definition of the struct and does not allocate any memory. We cannot use the standard `new` operator with this definition because it will construct an object without any space for the key-value entries.

So we use the placement `new` variation which allows us to separate construction of the object from the memory allocated for it. First, we will pre-allocate memory based on user-defined value for node fanout, and then use the placement `new` constructor in our pre-allocated memory buffer.

The memory allocation for the object is separate from the object construction phase. This fine-grained control over memory management allows us to create a contiguous B+Tree node in memory which is cache friendly, and whose node fanout is user-configurable.

```cpp
static BPlusTreeNode* Get(int max_size) {
    // 1. Calculate the total memory needed
    size_t total_size = sizeof(BPlusTreeNode) + (max_size * sizeof(KeyValuePair));

    // 2. Allocate a single raw memory block
    char* raw_memory = new char[total_size];

    // 3. Use placement new to construct the BPlusTreeNode object in that memory
    BPlusTreeNode* node = new(raw_memory) BPlusTreeNode();

    // (The constructor would typically initialize end_, etc.)
    return node;
}
```
