---
description: "Snowflake's result, warehouse and metadata caches: the exact reuse rules, 24-hour to 31-day result reuse, and auto-suspend vs cache warmth modelled."
---

# 04 · The Three Caches

The second run of a Snowflake query is often dramatically faster than the first, sometimes instant, and sometimes it
uses no warehouse at all. Three different caches are responsible, each with its own rules. Knowing them explains
confusing benchmarks ("my tuning made it 100× faster", when it really just hit the result cache), and it informs one
real cost decision: how long to keep a warehouse running.

## 1. The result cache (cloud services)

When a query finishes, its result is persisted. An identical query can be answered from that result **without running
anything**, often without a warehouse at all. From Snowflake's documentation:

- Results are kept for **24 hours**. Each reuse resets the 24 hours, up to a maximum of **31 days** from the first run.
- Reuse requires **the same query text** (case and whitespace included), and that:
  - the underlying table data hasn't changed (including micro-partition changes from reclustering);
  - the query uses no non-reusable functions (`UUID_STRING`, `RANDOM`, `RANDSTR`, external functions) and reads no
    hybrid tables;
  - the role has the required privileges on everything the query reads;
  - result-affecting configuration hasn't changed.
- Meeting these conditions doesn't *guarantee* reuse.
- The `USE_CACHED_RESULT` parameter (default `TRUE`) turns it off per session, user or account.

```sql
SELECT channel, SUM(quantity * unit_price) FROM learn_db.raw.orders GROUP BY channel;   -- runs on the warehouse
SELECT channel, SUM(quantity * unit_price) FROM learn_db.raw.orders GROUP BY channel;   -- answered from the result cache
select channel, sum(quantity * unit_price) from learn_db.raw.orders group by channel;   -- different text: runs again

ALTER SESSION SET USE_CACHED_RESULT = FALSE;     -- always do this before benchmarking
```

In the Query Profile, a cache hit shows a single `QUERY RESULT REUSE` node. Two practical uses:

- **Dashboards.** Many users opening the same dashboard run identical SQL, and after the first, it's free until the data
  changes. That's a strong reason to generate dashboard SQL consistently.
- **Post-processing.** `RESULT_SCAN` turns any recent result, including the output of `SHOW`, `DESCRIBE` or `CALL`, into a
  table you can query:

```sql
SHOW WAREHOUSES;
SELECT "name", "size", "auto_suspend", "state"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "auto_suspend" IS NULL OR "auto_suspend" > 300;
```

(Columns from `SHOW` output are lower-case and must be quoted.) Snowflake also documents a pipe operator `->>` that
chains a command into a follow-up query without displaying the first result.

## 2. The warehouse cache (local SSD)

Each warehouse node keeps micro-partition data it has read (only the columns read) on local SSD. A later query on the
same warehouse that needs the same data reads it locally instead of from remote object storage. The Query Profile and
`QUERY_HISTORY.PERCENTAGE_SCANNED_FROM_CACHE` report how much came from cache. Its rules are simpler and harsher:

- It belongs to **one warehouse**. Another warehouse reading the same table starts cold.
- It's **lost when the warehouse suspends**. Resizing also changes the set of nodes, and newly added nodes start cold.
- It holds data, not results, so it helps **different** queries that touch the same micro-partitions.

## 3. The metadata cache

Snowflake knows each micro-partition's row count and per-column min/max (lesson 01). Queries that can be answered from
that alone, such as `COUNT(*)` of a whole table or `MIN`/`MAX` of a column without filters, plus `SHOW` and `DESCRIBE`
commands, may be served by cloud services without scanning data and without a running warehouse.

```sql
SELECT COUNT(*), MIN(ordered_at), MAX(ordered_at) FROM learn_db.raw.orders;   -- often metadata-only
```

## The real decision: auto-suspend vs a warm cache

Lesson 04 of Level 1 showed that short auto-suspend saves credits. The warehouse cache adds the other side: every
resume starts cold. This model (Python locally) makes the trade-off concrete with **stated assumptions, not
measurements**. A dashboard query takes **3 s warm** and **12 s cold**, queries arrive at a fixed interval for one hour,
and billing follows the documented rules:

```python
WARM, COLD = 3, 12
def simulate(gap, auto_suspend, hours=1):
    t, billed, run_start, run_end, resumes, latency = 0, 0, None, None, 0, []
    while t < hours * 3600:
        warm = run_end is not None and t <= run_end + auto_suspend
        dur = WARM if warm else COLD
        if warm:
            run_end = max(run_end, t + dur)
        else:
            if run_start is not None:
                billed += max(60, run_end + auto_suspend - run_start)
            run_start, run_end, resumes = t, t + dur, resumes + 1
        latency.append(dur); t += gap
    billed += max(60, run_end + auto_suspend - run_start)
    return billed, resumes, sum(latency) / len(latency)
```

```text
 gap between queries AUTO_SUSPEND  billed s  resumes  avg latency s
                  60           60      3603        1            3.1
                  60          300      3843        1            3.1
                 120           60      2160       30           12.0
                 120          300      3783        1            3.3
                 300           60       864       12           12.0
                 300          300      3603        1            3.8
```

- **Queries every minute:** the warehouse never gets a chance to suspend either way, so a short auto-suspend costs
  nothing in latency.
- **Every 2 minutes:** a 60-second suspend saves 43% of the billed time (2,160 vs 3,783 s), but every query runs cold, at
  **12 s instead of 3.3 s average**.
- **Every 5 minutes:** 60 s suspend bills 864 s against 3,603 s, about a quarter of the cost, with all queries cold.

Neither setting is right in general. For an interactive dashboard where users notice 12 seconds, the longer suspend buys
4× faster responses at roughly double the cost. For a scheduled job nobody watches, take the cheap setting. And if the
dashboard SQL is consistent, the **result cache** may answer most opens for free anyway, which can make the question
moot.

## How It Actually Works

The **result cache** lives in the cloud services layer, not on a warehouse, which is why it survives warehouse suspends
and works across warehouses for users with the right privileges. A result is keyed on the exact query text plus the
context that affects results. Before reuse, cloud services checks that the tables' current versions are the ones the
result was computed from. Any committed change, even one that doesn't touch the rows in the result, creates a new table
version and invalidates it. Results are stored like query output for large results: as files in Snowflake-managed
storage that clients download directly.

The **warehouse cache** is ordinary local SSD on the compute nodes, organized by micro-partition file and column. Since
micro-partitions are immutable, a cached file can never be stale: either the query still needs that exact file, or the
table now references a different one. That's why the cache needs no invalidation logic, and why suspending (returning
the nodes) is the only thing that empties it.

## Common mistakes

- **Benchmarking with the result cache on.** The second run is a cache hit, not a faster query.
- **Generating dashboard SQL with varying whitespace, aliases or literal order**, which defeats result reuse.
- **`RANDOM()` or `UUID_STRING()` in dashboard queries**, which are never reusable.
- **Sharing one warehouse between unrelated workloads** to "share the cache". They evict each other's data and compete for
  CPU. Separate warehouses still win.
- **Setting auto-suspend by rule of thumb.** Decide from query arrival gaps and what users will tolerate.

## Exercise

1. In a trial account, run an aggregation over `SNOWFLAKE_SAMPLE_DATA.TPCH_SF10.LINEITEM` three times: normally, again
   (look for the result-reuse node), and again with `USE_CACHED_RESULT = FALSE` (look at the percentage scanned from
   cache). Then suspend and resume the warehouse and run it a fourth time.
2. Make the second run miss the result cache by changing only the letter case of a keyword. Then make it miss by
   inserting one row into an unrelated part of the table.
3. Use `SHOW TABLES` with `RESULT_SCAN` to list the five largest tables in a database by `"bytes"`.
4. Change the model's assumptions to match your own warehouse (measure warm and cold times for a real dashboard query)
   and find the auto-suspend that minimizes cost while keeping average latency under your target.
