---
title: "Record Shredding: Part 1"
date: 2025-06-06
published: false
---

<style>
th, td {
  text-align: center;
  vertical-align: top;
  padding: 8px 16px;
}

.table-container {
  display: flex;
  justify-content: space-around;
  gap: 24px;
}

.col-view td:nth-child(1) { background-color: #FFFFE0; }
.col-view td:nth-child(2) { background-color: #FFDAB9; }
.col-view td:nth-child(3) { background-color: #E6E6FA; }

.row-view tr:nth-child(1) td { background-color: #FFFFE0; }
.row-view tr:nth-child(2) td { background-color: #FFDAB9; }
.row-view tr:nth-child(3) td { background-color: #E6E6FA; }
.row-view tr:nth-child(4) td { background-color: #DDFADD; }

.col-view {
  border-spacing: 8px 0;
}

.row-view {
  border-spacing: 0 8px;
}

/* to be removed later */
body {
  max-width: 800px;
  margin: 0 auto;
  font-size: 20px;
}
</style>

# Introduction

The data is stored in row-major order in transactional storage engines like PostgreSQL, MySQL or SQLite. This matches
the transactional access patterns which targets to read, modify or delete a specific row of data or a very small set of
rows. The filtering conditions of the query have high-selectivity. When reading a row all the column attributes in
the relation are materialized in main memory. Having the entire row contiguous in memory once fetched minimizes
additional disk I/O.

On the other hand in an analytical database like ClickHouse or DuckDB, the access pattern is different enough that
it makes sense to store the same data in column-major order. Data for each column is stored contiguously. This is
because analytical workloads are primarily read-only, scan-heavy, heavily utilize aggregations with an explicit
goal of summarizing the data. The primary advantage here is that only those columns which are relevant for a query
needs to be materialized in memory. For scan-heavy queries which often have low-selectivity, this has a significant
impact on minimizing disk I/O and reducing memory footprint.

The data locality in columnar storage is crucial for query performance. It enables data parallelism where different
blocks of column values can be processed in parallel on multiple CPU cores. It helps with replacing scalar
operations with vectorized processing. Together data parallelism and vectorized processing has a big impact in
making analytical queries execute faster and efficiently.

<div class="table-container">
  <table class="col-view">
    <thead>
      <tr>
        <th colspan="3">column-major order</th>
      </tr>
      <tr>
        <th>id</th>
        <th>username</th>
        <th>role</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>101</td>
        <td>Alice</td>
        <td>sender</td>
      </tr>
      <tr>
        <td>102</td>
        <td>Bob</td>
        <td>receiver</td>
      </tr>
      <tr>
        <td>103</td>
        <td>Eve</td>
        <td>eavesdropper</td>
      </tr>
      <tr>
        <td>104</td>
        <td>Trudy</td>
        <td>intruder</td>
      </tr>
    </tbody>
  </table>

  <table class="row-view">
    <thead>
      <tr>
        <th colspan="3">row-major order</th>
      </tr>
      <tr>
        <th>id</th>
        <th>username</th>
        <th>role</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>101</td>
        <td>Alice</td>
        <td>sender</td>
      </tr>
      <tr>
        <td>102</td>
        <td>Bob</td>
        <td>receiver</td>
      </tr>
      <tr>
        <td>103</td>
        <td>Eve</td>
        <td>eavesdropper</td>
      </tr>
      <tr>
        <td>104</td>
        <td>Trudy</td>
        <td>intruder</td>
      </tr>
    </tbody>
  </table>
</div>
