---
description: "Reading Snowflake's Query Profile and history: exploding joins (6x rows here), spilling, poor pruning, queuing and UNION, with the fix for each."
---

# 03 · Query Profile & Performance Tuning

When a Snowflake query is slow, guessing is expensive: a bigger warehouse doubles the rate whether or not it helps.
Snowflake records what every query actually did, operator by operator, in the **Query Profile** and the **query
history**. This lesson is a field guide to reading them, organized around the problems Snowflake's documentation
itself lists as the most common: exploding joins, `UNION` without `ALL`, queries too large for memory (spilling), and
inefficient pruning. Queuing is added because it looks like slowness but isn't.

## Where to look

**Snowsight → Monitoring → Query History → a query → Query Profile** shows the execution plan as a tree of operators
(`TableScan`, `Filter`, `Join`, `Aggregate`, `Sort`, `WindowFunction`, `Result`…), the percentage of time spent in
each, rows flowing along each edge, and panes for Most Expensive Nodes, Statistics and Query Insights. The same
per-operator data is available in SQL:

```sql
SELECT operator_id, operator_type, operator_statistics, execution_time_breakdown
FROM TABLE(GET_QUERY_OPERATOR_STATS(LAST_QUERY_ID()));
```

And for many queries at once, `ACCOUNT_USAGE.QUERY_HISTORY` (latency up to 45 minutes, 365 days of history) has the
summary columns you'll use most:

```sql
SELECT query_id, warehouse_name, warehouse_size,
       total_elapsed_time / 1000            AS elapsed_s,
       compilation_time / 1000              AS compile_s,
       queued_overload_time / 1000          AS queued_s,
       partitions_scanned, partitions_total,
       bytes_scanned / POWER(1024, 3)       AS gb_scanned,
       percentage_scanned_from_cache,
       bytes_spilled_to_local_storage  / POWER(1024, 3) AS gb_spilled_local,
       bytes_spilled_to_remote_storage / POWER(1024, 3) AS gb_spilled_remote,
       rows_produced
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND warehouse_name IS NOT NULL
ORDER BY total_elapsed_time DESC
LIMIT 50;
```

Read a slow query's numbers in this order. Each points to a different fix.

## 1. Was it slow, or was it waiting?

`queued_overload_time` is time spent **waiting for the warehouse**, because it was already running as many queries as
it could. A query that ran for 2 seconds after queuing for 40 isn't slow. The warehouse is oversubscribed. The fix is
concurrency (a multi-cluster warehouse, or splitting workloads across warehouses, Level 4 · 06), not a bigger size.
`queued_provisioning_time` is time waiting for the warehouse to start or resize. `compilation_time` that's large
relative to execution points at very complex SQL (huge `IN` lists, deeply nested views) or a lot of metadata, as with
tables with very many micro-partitions and no pruning.

## 2. Exploding joins

A join whose output is far larger than its inputs usually means a **missing or incomplete join condition**. This is
reproduced on the course data: joining orders to a price history on `sku` alone, forgetting that each SKU has six
price rows over time (DuckDB 1.5.6 locally, not Snowflake):

```text
price_rows: 4800   skus: 800

join on sku only (missing the time condition)    -> 1,199,982 rows
ASOF join: price in effect at order time         ->   199,997 rows

revenue_wrong: 287,444,874
revenue_asof:   47,902,749
```

The join multiplied every order by six, and the revenue figure was **six times too high**. Nothing failed, and the
result looks plausible unless someone checks. In the Query Profile, this shows as a **Join** operator producing far
more rows than it consumes, often with most of the query's time spent in it. Fix the condition (here, `ASOF JOIN … MATCH_CONDITION`
from Level 1 · 06, or a `BETWEEN effective_at AND next_effective_at` range join), or deduplicate the right-hand side to
one row per key before joining. A cheap habit that catches this early: compare `COUNT(*)` before and after a join you
expect to be one-to-one.

## 3. UNION without ALL

`UNION` removes duplicates, which means a full aggregation over both inputs. `UNION ALL` just concatenates. In the
profile, `UNION` appears as a `UnionAll` operator with an extra `Aggregate` above it. If the inputs can't overlap (two
different months, two regions) or duplicates don't matter, write `UNION ALL`.

## 4. Spilling: the query didn't fit in memory

Large sorts, aggregations with many groups, window functions over big partitions, and joins with a large build side
need memory. When they run out, Snowflake **spills** intermediate data to the warehouse's local SSD (*bytes spilled to
local storage*) and, if that's not enough, to remote storage (*bytes spilled to remote storage*). Remote spilling is
dramatically slower.

Fixes, in order of preference:

1. **Reduce the data** before the expensive step: filter earlier, select fewer columns, aggregate before joining.
2. **Process in smaller batches** (per day, per region).
3. **Use a larger warehouse** for that query. More nodes means more memory and local disk. This is the case where
   sizing up genuinely pays: a query that spilled remotely on a Medium can run several times faster on a Large and cost
   less.
4. For memory-hungry Snowpark/ML work, a **Snowpark-optimized** warehouse (more memory per node).

## 5. Inefficient pruning

`partitions_scanned` close to `partitions_total` for a query with a selective filter means the filter couldn't skip data
(lessons 01–02). Check that the filter is on the raw column with a range predicate, not hidden in a function or a
subquery. If the filter is fine, the table's layout doesn't match the query, and clustering or sort-on-load fixes it.
`percentage_scanned_from_cache` tells you how much came from the warehouse's SSD cache rather than remote storage
(lesson 04).

## A tuning workflow

1. **Find what matters.** Rank recurring queries by total elapsed time or credits over a week, not the single slowest
   run:

```sql
SELECT query_parameterized_hash,
       ANY_VALUE(query_text)                 AS example,
       COUNT(*)                              AS runs,
       SUM(total_elapsed_time) / 1000 / 3600 AS total_hours,
       AVG(partitions_scanned / NULLIF(partitions_total, 0)) AS avg_scan_ratio,
       SUM(bytes_spilled_to_remote_storage) / POWER(1024, 3) AS gb_spilled_remote
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY query_parameterized_hash
ORDER BY total_hours DESC
LIMIT 20;
```

   `query_parameterized_hash` groups queries that differ only in literal values, so the dashboard query run 5,000
   times with different dates counts as one workload.
2. **Open the Query Profile** of a representative run and identify the most expensive operator.
3. **Match the symptom** to one of the five problems above and apply its fix.
4. **Re-measure** with `USE_CACHED_RESULT = FALSE`, so the result cache doesn't make it look instant (lesson 04).

## How It Actually Works

A query compiles in the cloud services layer into a plan of operators, and pruning is decided there (lesson 01). The
warehouse executes the plan in parallel: every node processes a share of the micro-partitions, and operators like joins
and aggregations **exchange** rows between nodes by hashing the join or grouping key, so that matching rows meet on the
same node. That exchange is why a skewed key (one customer with a huge share of the rows) overloads one node while
others idle. It shows up as one slow operator with a long tail in the profile's per-node statistics.

Each operator has a memory budget on its node. When a hash table or sort buffer outgrows it, the operator writes
partitions of its data to local SSD and processes them in passes, and when local disk is full, to remote object
storage. That's the spilling the profile reports. The fix is either less data per node (filter or batch) or more nodes
and memory (a larger warehouse). The per-operator row counts you see in the profile are the actual rows produced at
runtime, which is what makes exploding joins easy to spot after the fact.

## Common mistakes

- **Sizing up without reading the profile.** It doubles the cost of a queued, exploding or badly pruned query and
  fixes none of them.
- **Tuning the slowest single query** instead of the workload that consumes the most time in aggregate.
- **Joins that aren't one-to-one when you assume they are.** Check row counts.
- **`UNION` by habit.**
- **Benchmarking with the result cache on.** The second run looks instant.
- **Ignoring remote spilling.** It's the clearest signal that a larger warehouse (or smaller batches) will pay off.

## Exercise

1. In a trial account, write the exploding join against `SNOWFLAKE_SAMPLE_DATA.TPCH_SF1` deliberately (join `LINEITEM`
   to `PARTSUPP` on `partkey` only), open its Query Profile and find the Join operator's input and output row counts.
   Then fix it by adding `suppkey`.
2. Force spilling: run a `COUNT(DISTINCT …)` over a large sample table grouped by a high-cardinality key on an X-Small
   warehouse, then on a Medium. Compare bytes spilled, elapsed time and credits.
3. Run the workload query above in your account and pick the top workload. Which of the five problems does its profile
   show?
4. Rewrite a `UNION` query from your own work as `UNION ALL` where it's safe, and compare the two profiles.
