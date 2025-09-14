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
