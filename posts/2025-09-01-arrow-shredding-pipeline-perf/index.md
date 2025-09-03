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
        <a href="#phase-1:-getting-started">Phase 1: Getting Started</a>
        <ul>
            <li><a href="#01:-use-a-dictionary-data-type">01: Use a Dictionary Data Type</a></li>
            <li><a href="#02:-eliminate-intermediate-vector-allocation">02: Eliminate Intermediate Vector Allocation</a></li>
            <li><a href="#03:-preallocate-a-string-buffer">03: Preallocate a String Buffer</a></li>
            <li><a href="#04:-preallocate-a-string-buffer-2">04: Preallocate a String Buffer 2</a></li>
            <li><a href="#why-is-the-runtime-unchanged">Why is the Runtime Unchanged?</a></li>
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

## Phase 1: Getting Started

First, we build the CLI program in release mode and use that for end to end benchmarking using [hyperfine].

In `Cargo.toml` the following section is added for release builds:

```toml
[profile.release]
debug = "line-tables-only"
strip = false
```

This will include just enough debug information in the release binary which will help us trace hotspots back to the exact line of code in Rust. This is necessary when recording the call-graphs of the program's execution using `perf`.

When generating flamegraphs, we will use [rustfilt] to demangle the symbols for improved readability.

We will also collect hardware performance counters like - cycles, instructions retired, cache references, cache misses, branch instructions and branch mispredictions.

The following optimizations from 01 through 04, uses the flamegraph to identify hotspots indicated by tall towers and then attempt to squash it.

[hyperfine]: https://github.com/sharkdp/hyperfine
[rustfilt]: https://github.com/luser/rustfilt

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

### 02: Eliminate Intermediate Vector Allocation

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

### 03: Preallocate a String Buffer

In the hot loop, where a `RecordBatch` is being created, a string is allocated in the heap for each generated value. For a run of 10 million rows this is the equivalent of 10 million heap allocations.

We can eliminate 99% of these allocations by reusing a mutable string buffer within the loop where `PartialContact` template values are being materialized and appended into the `RecordBatch`.

Suppose a `RecordBatch` is created from a chunk of 1K row values, it now requires only 10K heap allocations.

```diff
+    let mut phone_number_buf = String::with_capacity(16);
+
     for PartialContact(name, phones) in chunk {
         name_builder.append_option(name);

@@ -155,11 +158,13 @@ fn to_record_batch(

     if has_phone_number {
        let id = phone_id_counter.fetch_add(1, Ordering::Relaxed);
-       let phone_number = Some(format!("+91-99-{id:08}"));
+       write!(phone_number_buf, "+91-99-{id:08}")?;
        struct_builder
            .field_builder::<StringBuilder>(PHONE_NUMBER_FIELD_INDEX)
            .unwrap()
-           .append_option(phone_number);
+           .append_value(&phone_number_buf);
+
+       phone_number_buf.clear();
```

After this change, there is again no noticeable change in the total runtime. But similar to earlier change, all measures point to an overall improvement in the CPU efficiency of the program.

### 04: Preallocate a String Buffer 2

This is a follow up optimization from the previous one. The idea is the same, to eliminate 99% of heap allocations when generating data, by preallocating a mutable string buffer, and reusing it.

```diff
 fn name_strategy() -> BoxedStrategy<Option<String>> {
     prop_oneof![
-        80 => Just(()).prop_map(|_| Some(format!("{} {}", FirstName().fake::<String>(), LastName().fake::<String>()))),
+        80 => Just(()).prop_map(|_| {
+            let mut name_buf = String::with_capacity(32);
+            write!(&mut name_buf, "{} {}", FirstName().fake::<&str>(), LastName().fake::<&str>()).unwrap();
+            Some(name_buf)
+         }),
         20 => Just(None)
     ].boxed()
 }

```

The results are identical to the previous optimization. No change in the total runtime. But there is considerable improvement in the CPU efficiency of the program.

### Why is the Runtime Unchanged?

The CPU efficiency has improved across most metrics from the baseline version.

The same program now executes in less CPU cycles, requires less instructions. Reducing heap allocations is particularly noticeable as reduced cache-references, cache-misses, branch-instructions and branch-misses.

Even though the runtime has not changed, the user time metric shows that we have shaved off ~2s (from 28s to under 26s) with these optimizations.

![Perf stats for baseline version up to run 04 ](img/perf_stats_phase1.png)

Even though the individual performance counter metrics looks good above, the IPC (instructions per cycle) has gone down from 1.20 to 1.18.

This has to be taken into consideration with the above metrics. We are now executing the same workload to produce the exact same result using less CPU instructions. That is definitely a micro-improvement.

![IPC trend for baseline version up to run 04](img/ipc_trend_phase1.png)

The optimizations so far had little to no effect on the total runtime of the program.

![Hyperfine box plots for baseline version up to run 04](img/hyperfine_boxplot_grid_phase1.png)

The flamegraphs look eerily similar.

![Flamegraph montage for baseline version up to run 04](img/flamegraph_montage_phase1.png)

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
