---
description: "How micro-partition min/max metadata lets Snowflake skip data, shown with Parquet statistics: 1 of 98 partitions read or all 98, by row order alone."
---

# 01 · Micro-Partitions & Pruning

Snowflake has no indexes. The single biggest factor in how much data a query reads, and therefore how long it runs and
what it costs, is **pruning**: skipping micro-partitions that can't contain matching rows, using the minimum and maximum
values Snowflake records for every column of every micro-partition. Pruning is automatic, but whether it *works*
depends on something you control: **how rows are grouped into micro-partitions**, which mostly means the order the data
arrived in.

## What Snowflake records per micro-partition

From Snowflake's documentation, each micro-partition holds 50–500 MB of uncompressed data, stored by column, and its
metadata includes:

- the **range of values** (min and max) of each column,
- the number of distinct values,
- other properties used for optimization.

When a query has a filter like `WHERE ordered_at BETWEEN '2026-03-15' AND '2026-03-16'`, the cloud services layer compares
the filter with every micro-partition's `ordered_at` range **before** a warehouse reads anything, and drops those whose
range can't overlap. Only the survivors are scanned, and of those, only the referenced columns.

## A real demonstration with Parquet

Parquet files work the same way: rows are grouped into **row groups**, and each row group stores min/max statistics
per column. That makes them a faithful, runnable stand-in for micro-partitions. This experiment (DuckDB 1.5.6 and
Python, locally, not Snowflake) writes the 200,000 orders from Level 1 twice: once in time order, as they'd arrive from
an application, and once shuffled, as they might after an unordered bulk load or many scattered updates:

```python
import duckdb
con = duckdb.connect("../data/sales.duckdb")
ROWS_PER_GROUP = 2000            # 100 "micro-partitions"
con.sql(f"COPY (SELECT * FROM orders ORDER BY ordered_at) TO 'orders_by_time.parquet' (ROW_GROUP_SIZE {ROWS_PER_GROUP})")
con.sql(f"COPY (SELECT * FROM orders ORDER BY hash(order_id)) TO 'orders_shuffled.parquet' (ROW_GROUP_SIZE {ROWS_PER_GROUP})")
```

DuckDB rounded the row-group size to its internal vector size, giving **98 row groups** of up to 2,048 rows. Their
`ordered_at` ranges look like this:

```text
orders_by_time.parquet                                  orders_shuffled.parquet
row_group │ stats_min           │ stats_max            row_group │ stats_min           │ stats_max
        0 │ 2026-01-01 00:02:42 │ 2026-01-03 20:07:00          0 │ 2026-01-01 00:43:58 │ 2026-09-27 21:06:39
        1 │ 2026-01-03 20:08:02 │ 2026-01-06 16:31:03          1 │ 2026-01-01 07:26:32 │ 2026-09-27 22:42:14
        2 │ 2026-01-06 16:33:40 │ 2026-01-09 12:51:36          2 │ 2026-01-01 01:30:33 │ 2026-09-27 19:16:20
```

In time order, each row group covers about three days. Shuffled, **every** row group covers the entire nine months. Now
apply the pruning rule (skip a group if its `[min, max]` can't overlap the filter) to three filters:

```python
def would_scan(path, col, lo, hi):
    conv = int if col == "customer_id" else str      # compare numbers as numbers, ISO timestamps as text
    groups = [(g, conv(mn), conv(mx)) for g, mn, mx in stats(path, col)]
    lo, hi = conv(lo), conv(hi)
    keep = [g for g in groups if not (g[2] < lo or g[1] > hi)]       # skip if [min,max] misses [lo,hi]
    return len(keep), len(groups)
```

```text
orders_by_time.parquet
   one day   (ordered_at on 2026-03-15)   scans   1 of 98 row groups
   one month (ordered_at in 2026-06)      scans  12 of 98 row groups
   one customer (customer_id = 4242)      scans  98 of 98 row groups
orders_shuffled.parquet
   one day   (ordered_at on 2026-03-15)   scans  98 of 98 row groups
   one month (ordered_at in 2026-06)      scans  98 of 98 row groups
   one customer (customer_id = 4242)      scans  98 of 98 row groups
```

The **same data**, the **same query**: one day reads **1%** of the table in time order and **100%** shuffled. And no
layout helps every column. Customer 4242's orders are spread across the whole year, so a customer filter scans
everything in both layouts.

(An earlier version of this script compared `customer_id` statistics as **text**, and "pruned" groups where `'4242'`
sorted outside a string range like `'1003'…'4999'`. It reported 84 of 98 for one layout, which was wrong. Pruning on
numbers has to compare numbers. The engine does this correctly. Hand-written analyses have to as well.)

## Measuring it: overlap and depth

Snowflake describes how well a table is laid out for a column with two numbers, both computed from micro-partition
ranges:

- **overlaps**: how many other micro-partitions a micro-partition's range overlaps;
- **depth**: how many micro-partitions contain a given value. A depth of 1 means a value lives in exactly one
  micro-partition.

Computing these from our row groups' `ordered_at` ranges (averaged over every partition, and over every boundary value
for depth, our approximation of Snowflake's metric):

```text
orders_by_time.parquet     partitions 98  average_overlaps   0.00  average_depth   1.00
orders_shuffled.parquet    partitions 98  average_overlaps  97.00  average_depth  49.50
```

In Snowflake you get these numbers, for real micro-partitions, from:

```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('learn_db.raw.orders', '(ordered_at)');
SELECT SYSTEM$CLUSTERING_DEPTH('learn_db.raw.orders', '(ordered_at)');
```

The first returns JSON with the total partition count, `average_overlaps`, `average_depth` and a histogram of
depths. Snowflake's docs note that depth isn't an absolute score: query performance is the real test. The direction
is clear, though: closer to 1 is better for filters on that column.

## Seeing pruning in Snowflake

For any query, the **Query Profile** in Snowsight (Level 2 · 03) shows, on the `TableScan` operator, **Partitions
scanned** and **Partitions total**. The same numbers are in the query history:

```sql
SELECT query_id, query_text, partitions_scanned, partitions_total,
       ROUND(100 * partitions_scanned / NULLIF(partitions_total, 0), 1) AS pct_scanned,
       bytes_scanned, total_elapsed_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE warehouse_name = 'LEARN_WH'
  AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY bytes_scanned DESC
LIMIT 20;
```

A frequently run query with a selective filter and `pct_scanned` near 100 is the clearest sign that the table's layout
doesn't match how it's queried.

## Writing filters that can prune

Pruning needs a predicate that can be compared with the column's min/max:

```sql
-- prunes: plain range on the column
SELECT SUM(quantity * unit_price) FROM learn_db.raw.orders
WHERE ordered_at >= '2026-03-15' AND ordered_at < '2026-03-16';

-- may not prune: the column is hidden inside an arbitrary expression
SELECT SUM(quantity * unit_price) FROM learn_db.raw.orders
WHERE TO_CHAR(ordered_at, 'YYYY-MM-DD') = '2026-03-15';

-- doesn't prune (per Snowflake's docs): the bound comes from a subquery, even if it returns a constant
SELECT SUM(quantity * unit_price) FROM learn_db.raw.orders
WHERE ordered_at >= (SELECT MAX(report_date) FROM learn_db.analytics.report_runs);
```

Snowflake can prune through some simple expressions (casts, date truncation), but not arbitrary ones, and it can't use
a bound it doesn't know at compile time. The reliable habits are to filter **the raw column** with **range predicates**,
and when the bound comes from data, look it up first and pass it in as a literal or a session variable.

## How It Actually Works

Pruning happens at **compile time**, in the cloud services layer, using metadata only. For each table in the query, the
optimizer walks the micro-partition list (which can hold millions of entries for large tables) and evaluates each
predicate against each partition's min/max statistics. A partition is kept if the predicate *might* be true for some
value in its range, and dropped if it's provably false for all of them. The warehouse receives only the surviving
file list. That's why pruning reduces both I/O and the number of files to fetch, and why it works the same on a cold
or warm warehouse.

Snowflake can also prune at **runtime** in joins: after building a hash table from the smaller side of a join, it can
derive a range of join-key values and skip probe-side partitions outside it. But compile-time pruning on filters is
the part you can design for.

Micro-partitions are formed **in the order rows are written**. Rows loaded together land together, so a table loaded
daily, in time order, is naturally well clustered by time, exactly like `orders_by_time.parquet`. What destroys that
layout is loading data in random order, or rewriting it with operations that reshuffle rows. The next lesson covers how
to restore or enforce a layout with clustering keys.

## Common mistakes

- **Assuming an index exists.** There are none. Layout plus pruning is the mechanism.
- **Wrapping the filter column in functions** (`TO_CHAR(ts) = …`, `YEAR(ts) = 2026` on hot paths). Filter on the
  column with ranges.
- **Filter bounds from subqueries**, which don't prune.
- **Judging a table by row count.** Look at `partitions_scanned / partitions_total` for the real queries.
- **Expecting good pruning on every column.** Only the column(s) the data is ordered by (or naturally correlated with)
  prune well.
- **Loading historical backfills in random order**, then wondering why recent-day queries scan everything.

## Exercise

1. Reproduce the Parquet experiment with row groups of 500 and 8,000 rows. How do the "one day" and "one month" scan
   counts change, and why?
2. Write a third layout ordered by `customer_id` and rerun the three filters. Which query improves, and which gets worse?
3. In Snowflake, load the orders once in file order and once shuffled (`INSERT INTO … SELECT … ORDER BY RANDOM()`), at
   a larger scale (`SNOWFLAKE_SAMPLE_DATA.TPCH_SF10.ORDERS` works). Compare `SYSTEM$CLUSTERING_INFORMATION` and
   `partitions_scanned` for a one-day filter.
4. Rewrite the subquery-bound query so it prunes, using `SET max_date = (SELECT …);` followed by
   `WHERE ordered_at >= $max_date`.
