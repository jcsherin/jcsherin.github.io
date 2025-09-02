---
title: 'Speeding Up a Parquet Shredding Pipeline by ~8x'
date: 2025-09-02
summary: >
  placeholder.
layout: layouts/post.njk
draft: true
---

Lately, I've been poking around record shredding and needed a dataset of nested data structures for tracing query execution. For this, I implemented a data generator which follows a [Zipfian-like] distribution. The generated data is staged in-memory as [Arrow RecordBatches], and then written to disk as Parquet files.

[Zipfian-like]: https://en.wikipedia.org/wiki/Zipf%27s_law
[Arrow RecordBatches]: https://arrow.apache.org/rust/parquet/arrow/index.html

```rust
    let config = PipelineConfigBuilder::new()
        .with_target_records(cli.target_records)
        .with_num_writers(cli.num_writers)
        .with_record_batch_size(cli.record_batch_size)
        .with_output_dir(output_dir)
        .with_output_filename(cli.output_filename)
        .with_arrow_schema(get_contact_schema())
        .try_build()?;

    let factory = ContactGeneratorFactory::from_config(&config);
    let metrics = run_pipeline(&config, &factory)?;
```

The first version was a simple pipeline using Rust MPSC which connects multiple data generation (producer) threads to a single Parquet writer (consumer) thread. It took around ~3.7s to generate a dataset containing 10 million rows. 

The measurements plotted below represents the sequence of 12 optimizations which contributed to reducing the final runtime to ~440ms (8x speedup). The chart shows that not every optimization resulted in lowering the runtime, but it contributed to improving the efficiency of the program as measured through other performance metrics. 

A string interning optimization (no. `09`) which looked good on paper, turned out to be not so good in practice. Later in this post, we'll see why. The code changes had to be reverted (no. `10`) to get back what was lost.

![Performance Trend Across Runs](img/hyperfine_trend_plot.png)
<figcaption>Fig 1. Total runtime measured after each optimization step (Lower is better). The first runtime <code>00 (x-axis)</code> represents the baseline version before of the program before any optimizations.</figcaption>

All benchmarks were run on a Linux machine with the following configuration: 
- Ubuntu 24.04.2 LTS (Kernel 6.8)
- AMD Ryzen 7 PRO 8700GE (8 Cores, 16 Threads)
- 64 GB of DDR5-5600 ECC RAM
- 512 GB NVMe SSDs.

