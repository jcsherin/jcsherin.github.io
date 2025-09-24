---
title: 'Test Page: All Markdown Patterns'
date: 2025-09-24
summary: >
  A kitchen sink page containing all markdown and HTML patterns used across the blog for easy styling and testing.
layout: layouts/post.njk
draft: true
---

This page is a test bed for all the markdown and HTML elements used in this blog. Use it to test CSS changes.

<nav class="toc" aria-labelledby="toc-heading">
  <h2 id="toc-heading">Table of Contents</h2>
  <ol>
    <li>
      <a href="#headings">Headings</a>
      <ul>
        <li><a href="#heading-3">Heading 3</a></li>
        <li><a href="#heading-4">Heading 4</a></li>
        <li><a href="#heading-5">Heading 5</a></li>
        <li><a href="#heading-6">Heading 6</a></li>
      </ul>
    </li>
    <li><a href="#paragraphs-and-text-formatting">Paragraphs and Text Formatting</a></li>
    <li><a href="#links">Links</a></li>
    <li><a href="#blockquotes">Blockquotes</a></li>
    <li>
      <a href="#lists">Lists</a>
      <ul>
        <li><a href="#unordered-list">Unordered List</a></li>
        <li><a href="#ordered-list">Ordered List</a></li>
      </ul>
    </li>
    <li><a href="#images">Images</a></li>
    <li><a href="#code-blocks">Code Blocks</a></li>
    <li>
      <a href="#tables">Tables</a>
      <ul>
        <li><a href="#markdown-table">Markdown Table</a></li>
        <li><a href="#html-table">HTML Table</a></li>
      </ul>
    </li>
  </ol>
</nav>

## Headings

This is a Heading 2 (the post title is H1).

### Heading 3

This is a Heading 3. It's used for sub-sections.

#### Heading 4

This is a Heading 4.

##### Heading 5

This is a Heading 5.

###### Heading 6

This is a Heading 6.

## Paragraphs and Text Formatting

This is a standard paragraph of text. It contains **bold text**, _italic text using underscores_, and _italic text using asterisks_. You can also have `inline code` which is useful for mentioning variables or filenames.

Another paragraph follows, demonstrating the flow of text. Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

## Links

Here is an [inline link to Google](https://www.google.com).
Here is a [reference-style link to the Eleventy docs][11ty].

[11ty]: https://www.11ty.dev/

## Blockquotes

> This is a blockquote. It's useful for highlighting a quote or an important note from another source. It can span multiple lines.

> An efficient implementation will avoid allocation for a new node, and reuse the memory of the sibling node.

## Lists

### Unordered List

- Item 1
- Item 2
  - Nested Item 2a
  - Nested Item 2b
- Item 3

### Ordered List

1.  First item
2.  Second item
    1.  Nested item 2.1
    2.  Nested item 2.2
3.  Third item

## Images

This is an SVG image using the new `figure` shortcode:
{% figure "/posts/2025-07-23-shredding-nested-data-in-parquet/img/figure-1.svg", "An SVG image" %}
Fig 1. An SVG image that should scale correctly.
{% endfigure %}

This is a PNG image using our new `figure` shortcode:
{% figure "/posts/2025-09-01-arrow-shredding-pipeline-perf/img/flamegraph_grid_numbered.png", "A flamegraph grid" %}
Fig 2. A PNG image that may need styling to prevent overflow on mobile.
{% endfigure %}

## Code Blocks

Here is a `text` code block:

```text
Some plain text inside a code block.
No syntax highlighting is applied here.
+-----------------------------+----------------------------------------+
| NAME                        | COMMENT                                |
+-----------------------------+----------------------------------------+
```

<figcaption>A caption for a text code block.</figcaption>

A `rust` code block with syntax highlighting:

```rust
fn main() -> Result<(), Box<dyn Error + Send + Sync>> {
    println!("Hello, World!");
}
```

## Tables

### Markdown Table

| Header 1 | Header 2 | Header 3 |
| -------- | -------- | -------- |
| Cell 1-1 | Cell 1-2 | Cell 1-3 |
| Cell 2-1 | Cell 2-2 | Cell 2-3 |

### HTML Table

<table>
  <thead>
    <tr>
      <th>HTML Header 1</th>
      <th>HTML Header 2</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>HTML Cell 1-1</td>
      <td>HTML Cell 1-2</td>
    </tr>
    <tr>
      <td>HTML Cell 2-1</td>
      <td>HTML Cell 2-2</td>
    </tr>
  </tbody>
</table>
