---
title: 'Speeding Up a Parquet Shredding Pipeline by ~8x'
date: 2025-09-02
summary: >
  placeholder.
layout: layouts/post.njk
draft: true
---

Lately, I've been poking around record shredding and needed a dataset of nested data structures for tracing query execution of shredded data. For this, I implemented a data generator which follows a [Zipfian-like] distribution. The generated data is staged in-memory as [Arrow RecordBatches], and then written to disk as Parquet files.

[Zipfian-like]: https://en.wikipedia.org/wiki/Zipf%27s_law
[Arrow RecordBatches]: https://arrow.apache.org/rust/parquet/arrow/index.html

The baseline version I wrote is a simple pipeline using Rust MPSC which connects multiple data generation (producer) threads to a single Parquet writer (consumer) thread. For a nested dataset of 10 million rows, it ~3.7s to complete. In this post, we'll see how a sequence of performance optimizations, reduced the total runtime to ~440ms (8x speedup).

![Performance Trend Across Runs](img/hyperfine_trend_plot.png)

<figcaption>Fig 1. The performance trend across a sequence of code optimizations, measured using <code>hyperfine</code>. The black line indicates the median runtime in seconds, while the shaded area indicates the range between min and max runtime.</figcaption>

![IPC Trend Across Runs](img/ipc_trend_plot.png)

This chart displays only improvements in total runtime, which does not tell the whole story. While some optimizations here show no difference in the total runtime, the improvements came from higher IPC (instructions per cycle), fewer cache misses and fewer branch mispredictions.

A string interning optimization (no. 9) looked like a guaranteed win. It was introduced to eliminate a lot of small string allocations in the data generation (producer) threads. The performance got worse (more on this later in this post) and the change had to be reverted. This strongly reinforces, the importance of measurements and profiling data for knowing unambiguously if a code optimization made an improvement or did the opposite.

All benchmarks were run on a Linux machine with the following configuration:

- Ubuntu 24.04.2 LTS (Kernel 6.8)
- AMD Ryzen 7 PRO 8700GE (8 Cores, 16 Threads)
- 64 GB of DDR5-5600 ECC RAM
- 512 GB NVMe SSDs.

<nav class="toc" aria-labelledby="toc-heading">
  <h2 id="toc-heading">Table of Contents</h2>
  <ol>
    <li><a href="#background">Background</a></li>
    <li>
        <a href="#phase-1:-minor-improvements">Phase 1: Minor Improvements</a>
        <ul>
            <li><a href="#01:-use-a-dictionary-data-type">01: Use a Dictionary Data Type</a></li>
        </ul>
    </li>
    <li>
        <a href="#phase-2:-architectural-changes">Phase 2: Architectural Changes</a>
    </li>
    <li>
        <a href="#phase-3:-a-performance-regression">Phase 3: A Performance Regression</a>
    </li>
    <li>
        <a href="#phase-4:-micro-optimizations">Phase 4: Micro-Optimizations</a>
    </li>
</ol>
</nav>

## Background

The program is a CLI tool for generating a target number of rows of nested data structures and then written to disk in Parquet format.

Nested data structures do not naturally fit into a flat columnar format. Record shredding is a process which converts the nested data into a flat, columnar format while preserving the original structural hierarchy of the raw data.

The generated data follows a [Zipfian-like] distribution. It is staged in memory as [Arrow RecordBatches], before being written to disk as Parquet files.

The data is generated in parallel using a [Rayon] thread pool. Then data generator threads (producers) sends the data to a Parquet writer thread (consumer). The number of writers are configurable from the CLI.

[Rayon]: https://github.com/rayon-rs/rayon

## Phase 1: Minor Improvements

### 01: Use a Dictionary Data Type

In the baseline version, the `PhoneType` Rust enum is mapped to a string data type (`DataType::Utf8`) in the Arrow schema.

```rust
pub enum PhoneType {
    Mobile,
    Home,
    Work,
}
```

Instead, by changing the Arrow field data type to `DataType::Dictionary`, the expectation is that the total memory footprint of the program, and storage size of the Parquet file will improve.

```diff
 pub fn get_contact_phone_fields() -> Vec<Arc<Field>> {
     vec![
         Arc::from(Field::new("number", DataType::Utf8, true)),
-        Arc::from(Field::new("phone_type", DataType::Utf8, true)),
+        Arc::from(Field::new(
+            "phone_type",
+            DataType::Dictionary(Box::new(DataType::UInt8), Box::new(DataType::Utf8)),
+            true,
+        )),
     ]
 }
```

After the change, the maximum RSS (resident set size) is reduced by ~1MB in a run for generating 10 million rows. The Parquet storage size improvement is negligible. There is a minor regression in runtime.

Even though, there are no dramatic gains here like we expected, we will maintain this change because it removes the mismatch between the underlying Rust and Arrow data types. That is definitely a readability improvement.

### 02: Eliminate Vec Allocation

The generate data with a predefined data skew (Zipfian-like), a data template value is first generated. The holes in the templates are filled in to generate the final `Contact` struct value, which is then converted to an Arrow `RecordBatch`. The series of value transformations looks like this:

`Vec<PartialContact>` → `Vec<Contact>` → `RecordBatch`.

Instead of creating the intermediate `Vec<Contact>`, we can do a late materialization of the final `Contact` value when building a `RecordBatch` by directly passing it the instructions within `Vec<PartialContact>`. After eliminating the intermediate step, the value transformation will look like this:

`Vec<PartialContact>` → `RecordBatch`.

```diff
-  // Assemble the Vec<Contact> for this small chunk
-  let contacts_chunk: Vec<Contact> = partial_contacts
-      .into_iter()
-      .map(|partial_contact| { ... })
-      .collect();
-
-  // Convert the chunk to a RecordBatch and send it to the writer
-  let record_batch = create_record_batch(parquet_schema.clone(), &contacts_chunk)
-      .expect("Failed to create RecordBatch");
+  let record_batch =
+      to_record_batch(
            parquet_schema.clone(), &phone_id_counter, partial_contacts)
+       .expect("Failed to create RecordBatch");
+
```

After the change, there is no noticeable change in total runtime. On the other hand, there is a noticeable improvement across the board in CPU utilization metrics. Even though the pipeline did not execute any faster, it ran more efficiently.

![Flamegraph montage for baseline version up to run 04](img/flamegraph_montage_phase1.png)
![Hyperfine box plots for baseline version up to run 04](img/hyperfine_boxplot_grid_phase1.png)
![Perf stats for baseline version up to run 04 ](img/perf_stats_phase1.png)
![IPC trend for baseline version up to run 04](img/ipc_trend_phase1.png)

## Phase 2: Architectural Changes

![Flamegraph montage from run 04 to run 08](img/flamegraph_montage_phase2.png)
![Hyperfine box plots from run 04 to run 08](img/hyperfine_boxplot_grid_phase2.png)
![Perf stats from run 04 to run 08](img/perf_stats_phase2.png)
![IPC trend from run 04 to run 08](img/ipc_trend_phase2.png)

## Phase 3: A Performance Regression

![Flamegraph montage from run 08 to run 10](img/flamegraph_montage_phase3.png)
![Hyperfine box plots from run 08 to run 10](img/hyperfine_boxplot_grid_phase3.png)
![Perf stats from run 08 to run 10](img/perf_stats_phase3.png)
![IPC trend from run 08 to run 10](img/ipc_trend_phase3.png)

## Phase 4: Micro-optimizations

![Flamegraph montage from run 10 to run 12](img/flamegraph_montage_phase4.png)
![Hyperfine box plots from run 10 to run 12](img/hyperfine_boxplot_grid_phase4.png)
![Perf stats from run 10 to run 12](img/perf_stats_phase4.png)
![IPC trend from run 10 to run 12](img/ipc_trend_phase4.png)
