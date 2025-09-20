---
title: 'Faster LIKE Queries In Parquet Without an External Index'
date: 2025-09-02
summary: >
  placeholder.
layout: layouts/post.njk
draft: true
---

The query:

```sql
SELECT *
  FROM 'foo.parquet'
 WHERE title LIKE '%Diary%';
```

The default query plan for `LIKE` queries in [Apache DataFusion].

[Apache DataFusion]: https://datafusion.apache.org/

```text
+---------------+-------------------------------+
| plan_type     | plan                          |
+---------------+-------------------------------+
| physical_plan | ┌───────────────────────────┐ |
|               | │    CoalesceBatchesExec    │ |
|               | │    --------------------   │ |
|               | │     target_batch_size:    │ |
|               | │            8192           │ |
|               | └─────────────┬─────────────┘ |
|               | ┌─────────────┴─────────────┐ |
|               | │         FilterExec        │ |
|               | │    --------------------   │ |
|               | │         predicate:        │ |
|               | │     title LIKE %Diary%    │ |
|               | └─────────────┬─────────────┘ |
|               | ┌─────────────┴─────────────┐ |
|               | │      RepartitionExec      │ |
|               | │    --------------------   │ |
|               | │ partition_count(in->out): │ |
|               | │          1 -> 12          │ |
|               | │                           │ |
|               | │    partitioning_scheme:   │ |
|               | │    RoundRobinBatch(12)    │ |
|               | └─────────────┬─────────────┘ |
|               | ┌─────────────┴─────────────┐ |
|               | │       DataSourceExec      │ |
|               | │    --------------------   │ |
|               | │          files: 1         │ |
|               | │      format: parquet      │ |
|               | │                           │ |
|               | │         predicate:        │ |
|               | │     title LIKE %Diary%    │ |
|               | └───────────────────────────┘ |
|               |                               |
+---------------+-------------------------------+
1 row(s) fetched.
Elapsed 0.016 seconds.

```

The optimized query plan where an embedded Tantivy full-text index embedded within the Parquet file is used.

```text
+---------------+-------------------------------+
| plan_type     | plan                          |
+---------------+-------------------------------+
| physical_plan | ┌───────────────────────────┐ |
|               | │       DataSourceExec      │ |
|               | │    --------------------   │ |
|               | │          files: 1         │ |
|               | │      format: parquet      │ |
|               | │                           │ |
|               | │         predicate:        │ |
|               | │        id IN (3, 2)       │ |
|               | └───────────────────────────┘ |
|               |                               |
+---------------+-------------------------------+
```

The Tantivy full-text index is embedded within Parquet as a sequence of bytes.

File format specification:

```text
+=================================================================+
|                              Header                             |
|-----------------------------------------------------------------|
| MAGIC_BYTES ('FTEP')                            |   (4 bytes)   |
|-------------------------------------------------+---------------|
| version                                         |   (1 byte)    |
|-------------------------------------------------+---------------|
| file_count                                      |   (4 bytes)   |
|-------------------------------------------------+---------------|
| total_data_block_size                           |   (8 bytes)   |
|-------------------------------------------------+---------------|
| file_metadata_size                              |   (4 bytes)   |
|-------------------------------------------------+---------------|
| file_metadata_crc32                             |   (4 bytes)   |
|-----------------------------------------------------------------|
|                  file_metadata_list (variable size)             |
|                  (A series of FileMetadata entries)             |
|                                                                 |
| +-------------------------------------------------------------+ |
| | FileMetadata Entry 1 (e.g., for "meta.json")                | |
| |-------------------------------------------------------------| |
| | data_offset (points to start of file in Data Block) | (8 B) | | ------+
| | data_content_len                                    | (8 B) | |       |
| | data_footer_len (0 for meta.json)                   | (1 B) | |       |
| | path_len                                            | (1 B) | |       |
| | path ("meta.json")                                  |(~9 B) | |       |
| +-------------------------------------------------------------+ |       |
| | FileMetadata Entry 2 (e.g., for ".fast")                    | |       |
| |-------------------------------------------------------------| |       |
| | data_offset (points to start of file in Data Block) | (8 B) | | ----+ |
| | ... (other fields) ...                                      | |     | |
| +-------------------------------------------------------------+ |     | |
| |                             ...                             | |     | |
+=================================================================+     | |
|                            DataBlock                            |     | |
|                 (Size = total_data_block_size)                  |     | |
|-----------------------------------------------------------------| <---+ |
|                                                                 |       |
|             Contents of "meta.json" (raw bytes)                 |       |
|                                                                 |       |
|-----------------------------------------------------------------| <-----+
|                                                                 |
|               Contents of ".fast" file (raw bytes)              |
|                                                                 |
|-----------------------------------------------------------------|
|                                                                 |
|        Reconstructed `TantivyFooterHack` for ".fast" file       |
|                                                                 |
|-----------------------------------------------------------------|
|                                                                 |
|              Contents of ".fieldnorm" file (raw bytes)          |
|                                                                 |
|-----------------------------------------------------------------|
|                                                                 |
|      Reconstructed `TantivyFooterHack` for ".fieldnorm" file    |
|                                                                 |
|-----------------------------------------------------------------|
|                                                                 |
|      ... and so on for all other Tantivy index files            |
|              (.idx, .pos, .term, .store, etc.)                  |
|                                                                 |
+-----------------------------------------------------------------+
```

Histogram of control phrases in generated Parquet file.

```
DataFusion CLI v49.0.0
> SELECT
    COUNT(CASE WHEN title LIKE '%concurrency%' THEN 1 END) AS concurrency_count,
    COUNT(CASE WHEN title LIKE '%runtime%' THEN 1 END) AS runtime_count,
    COUNT(CASE WHEN title LIKE '%indexing%' THEN 1 END) AS indexing_count,
    COUNT(CASE WHEN title LIKE '%normalization%' THEN 1 END) AS normalization_count,
    COUNT(CASE WHEN title LIKE '%atomics%' THEN 1 END) AS atomics_count,
    COUNT(CASE WHEN title LIKE '%idempotency%' THEN 1 END) AS idempotency_count
FROM '/Users/jcsherin/Projects/rusty-doodles/output/titles.parquet';
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
| concurrency_count | runtime_count | indexing_count | normalization_count | atomics_count | idempotency_count |
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
| 12905516          | 4999816       | 2500033        | 498659              | 250832        | 49910             |
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
1 row(s) fetched.
Elapsed 11.176 seconds.

```

Target size = 50 million rows. Command used:

```text
RUST_LOG=trace,tantivy=off cargo run --bin parquet_gen -- -t 50000000
```

2.5 GB on disk.

```text
❯ ls -l output/titles.parquet
-rw-r--r--  1 jcsherin  staff  2701649192 14 Sep 12:40 output/titles.parquet
```

## 1 million rows

```text
-rw-r--r--  1 jcsherin  staff    96M 14 Sep 14:22 output/titles_with_fts_index.parquet
-rw-r--r--  1 jcsherin  staff    52M 14 Sep 14:22 output/titles.parquet
```

Histogram for

```
> SELECT
    COUNT(CASE WHEN title LIKE '%concurrency%' THEN 1 END) AS concurrency_count,
    COUNT(CASE WHEN title LIKE '%runtime%' THEN 1 END) AS runtime_count,
    COUNT(CASE WHEN title LIKE '%indexing%' THEN 1 END) AS indexing_count,
    COUNT(CASE WHEN title LIKE '%normalization%' THEN 1 END) AS normalization_count,
    COUNT(CASE WHEN title LIKE '%atomics%' THEN 1 END) AS atomics_count,
    COUNT(CASE WHEN title LIKE '%idempotency%' THEN 1 END) AS idempotency_count
FROM '/Users/jcsherin/Projects/rusty-doodles/output/titles_with_fts_index.parquet';
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
| concurrency_count | runtime_count | indexing_count | normalization_count | atomics_count | idempotency_count |
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
| 258749            | 99804         | 49940          | 9949                | 4903          | 987               |
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
1 row(s) fetched.
Elapsed 1.611 seconds.

> SELECT
    COUNT(CASE WHEN title LIKE '%concurrency%' THEN 1 END) AS concurrency_count,
    COUNT(CASE WHEN title LIKE '%runtime%' THEN 1 END) AS runtime_count,
    COUNT(CASE WHEN title LIKE '%indexing%' THEN 1 END) AS indexing_count,
    COUNT(CASE WHEN title LIKE '%normalization%' THEN 1 END) AS normalization_count,
    COUNT(CASE WHEN title LIKE '%atomics%' THEN 1 END) AS atomics_count,
    COUNT(CASE WHEN title LIKE '%idempotency%' THEN 1 END) AS idempotency_count
FROM '/Users/jcsherin/Projects/rusty-doodles/output/titles.parquet';
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
| concurrency_count | runtime_count | indexing_count | normalization_count | atomics_count | idempotency_count |
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
| 258749            | 99804         | 49940          | 9949                | 4903          | 987               |
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
1 row(s) fetched.
Elapsed 1.652 seconds.
```

## 10 million rows

```
-rw-r--r--@ 1 jcsherin  staff   926M 14 Sep 14:32 output/titles_with_fts_index.parquet
-rw-r--r--@ 1 jcsherin  staff   515M 14 Sep 14:31 output/titles.parquet
```

```
> SELECT
    COUNT(CASE WHEN title LIKE '%concurrency%' THEN 1 END) AS concurrency_count,
    COUNT(CASE WHEN title LIKE '%runtime%' THEN 1 END) AS runtime_count,
    COUNT(CASE WHEN title LIKE '%indexing%' THEN 1 END) AS indexing_count,
    COUNT(CASE WHEN title LIKE '%normalization%' THEN 1 END) AS normalization_count,
    COUNT(CASE WHEN title LIKE '%atomics%' THEN 1 END) AS atomics_count,
    COUNT(CASE WHEN title LIKE '%idempotency%' THEN 1 END) AS idempotency_count
FROM '/Users/jcsherin/Projects/rusty-doodles/output/titles.parquet';
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
| concurrency_count | runtime_count | indexing_count | normalization_count | atomics_count | idempotency_count |
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
| 2580730           | 1000423       | 499792         | 99072               | 49798         | 9865              |
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
1 row(s) fetched.
Elapsed 2.497 seconds.

> SELECT
    COUNT(CASE WHEN title LIKE '%concurrency%' THEN 1 END) AS concurrency_count,
    COUNT(CASE WHEN title LIKE '%runtime%' THEN 1 END) AS runtime_count,
    COUNT(CASE WHEN title LIKE '%indexing%' THEN 1 END) AS indexing_count,
    COUNT(CASE WHEN title LIKE '%normalization%' THEN 1 END) AS normalization_count,
    COUNT(CASE WHEN title LIKE '%atomics%' THEN 1 END) AS atomics_count,
    COUNT(CASE WHEN title LIKE '%idempotency%' THEN 1 END) AS idempotency_count
FROM '/Users/jcsherin/Projects/rusty-doodles/output/titles_with_fts_index.parquet';
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
| concurrency_count | runtime_count | indexing_count | normalization_count | atomics_count | idempotency_count |
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
| 2580730           | 1000423       | 499792         | 99072               | 49798         | 9865              |
+-------------------+---------------+----------------+---------------------+---------------+-------------------+
1 row(s) fetched.
Elapsed 3.867 seconds.
```
