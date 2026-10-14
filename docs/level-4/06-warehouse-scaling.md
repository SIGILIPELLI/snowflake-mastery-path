---
description: "Scaling Snowflake compute: scale up vs out modelled, multi-cluster policies, query acceleration, Gen2, Snowpark-optimized and Adaptive Warehouses."
---

# 06 · Warehouse Scaling: Up, Out & Adaptive

A warehouse that's too small makes people wait. One that's too big burns credits. Snowflake gives you several levers, and
they solve **different** problems: a bigger size makes individual queries faster, more clusters let more queries run at once,
query acceleration helps the odd huge scan, and newer warehouse types change the trade-offs altogether. This lesson shows which
lever fits which symptom, with a model that makes the main trade-off visible and states the assumption it hinges on.

## Two symptoms, two levers

From the Query Profile and `QUERY_HISTORY` (Level 2 · 03), slow queries come in two kinds:

| Symptom | Evidence | Lever |
|---|---|---|
| **Each query is slow**, even alone | long execution time, spilling to local or remote storage | **scale up** (a bigger size) |
| **Queries wait** for their turn | high `queued_overload_time`, fine execution time | **scale out** (more clusters) |

```sql
SELECT warehouse_name,
       COUNT(*)                                          AS queries,
       ROUND(AVG(queued_overload_time) / 1000, 1)        AS avg_queued_s,
       ROUND(AVG(execution_time) / 1000, 1)              AS avg_exec_s,
       SUM(IFF(bytes_spilled_to_remote_storage > 0, 1, 0)) AS remote_spills
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND warehouse_name IS NOT NULL
GROUP BY warehouse_name
ORDER BY avg_queued_s DESC;
```

## Scale up or scale out? A model

Eighty dashboard queries arrive within 20 seconds at 09:00, a typical Monday-morning burst. Each takes 32 seconds of work on a
Small warehouse. The deciding question is how much faster a query gets on a bigger warehouse, and that depends on the query, so
the model runs twice: once with perfect scaling, once with the poor scaling typical of small queries:

```python
# Model (not Snowflake): 80 dashboard queries arrive in the first 20 s of 09:00, each 32 s of work on a Small warehouse.
# Assumptions, stated because they decide the answer:
#  - a cluster runs at most 8 queries at once (the default MAX_CONCURRENCY_LEVEL) with no slowdown below that;
#  - doubling the size multiplies a query's duration by SPEEDUP: 0.5 is perfect scaling, 0.8 is what small
#    queries that can't use the extra nodes look like (both are assumptions; measure your own);
#  - an extra cluster starts as soon as a query queues and is usable after 0 s; idle clusters stop when empty;
#  - credits: Small = 2/h, Medium = 4/h, Large = 8/h per cluster, per second, 60 s minimum per cluster start.
import statistics
arrivals = [i * 0.25 for i in range(80)]

def simulate(credits_per_h, duration, max_clusters):
    clusters = []                                          # each: list of slot-free times, start time, last end
    finish = []
    for t in arrivals:
        best = None
        for c in clusters:                                 # earliest free slot on running clusters
            slot = min(range(8), key=lambda i: c["slots"][i])
            if best is None or c["slots"][slot] < best[0]:
                best = (c["slots"][slot], c, slot)
        if best is None or (best[0] > t and len(clusters) < max_clusters):
            c = {"slots": [t] * 8, "start": t, "end": t}   # queued (or first query): start a cluster
            clusters.append(c); best = (t, c, 0)
        begin, c, slot = max(best[0], t), best[1], best[2]
        end = begin + duration
        c["slots"][slot] = end; c["end"] = max(c["end"], end)
        finish.append(end - t)
    credits = sum(max(c["end"] - c["start"], 60) for c in clusters) * credits_per_h / 3600
    return len(clusters), statistics.median(finish), sorted(finish)[int(0.95 * len(finish)) - 1], max(finish), credits

for speedup in (0.5, 0.8):
    print(f"\nSPEEDUP per size doubling = {speedup}")
    print(f"{'configuration':<36}{'clusters':>9}{'p50 s':>8}{'p95 s':>8}{'credits':>9}")
    for name, cph, dur, mx in [("Small, 1 cluster", 2, 32, 1),
                               ("Medium, 1 cluster (scale up)", 4, 32 * speedup, 1),
                               ("Large, 1 cluster (scale up)", 8, 32 * speedup ** 2, 1),
                               ("Small, up to 4 clusters (scale out)", 2, 32, 4)]:
        n, p50, p95, _, cr = simulate(cph, dur, mx)
        print(f"{name:<36}{n:>9}{p50:>8.0f}{p95:>8.0f}{cr:>9.3f}")
```

```text

SPEEDUP per size doubling = 0.5
configuration                        clusters   p50 s   p95 s  credits
Small, 1 cluster                            1     167     302    0.179
Medium, 1 cluster (scale up)                1      79     142    0.180
Large, 1 cluster (scale up)                 1      35      62    0.182
Small, up to 4 clusters (scale out)         4      56      80    0.182

SPEEDUP per size doubling = 0.8
configuration                        clusters   p50 s   p95 s  credits
Small, 1 cluster                            1     167     302    0.179
Medium, 1 cluster (scale up)                1     132     238    0.286
Large, 1 cluster (scale up)                 1     104     187    0.459
Small, up to 4 clusters (scale out)         4      56      80    0.182
```

With **perfect** scaling, all options cost about the same, because twice the credits per hour for half the time is the same
money, and a Large warehouse gives the best latency. With **poor** scaling, scaling up costs up to 2.5 times more and still
leaves queries waiting, while **four Small clusters** cut p95 latency from 302 to 80 seconds for the same credits as one.
Real queries sit somewhere between, and the model ignores cluster start-up time, so treat the numbers as a shape, not a
forecast. The rule it supports: **scale out for concurrency; scale up only for queries that get faster when they get more
compute**. Test that by running the same query on two sizes with the result cache off.

## Multi-cluster warehouses

Multi-cluster warehouses need **Enterprise Edition**. Set a range of clusters, and Snowflake starts and stops clusters within it:

```sql
CREATE OR REPLACE WAREHOUSE bi_wh
  WAREHOUSE_SIZE = 'SMALL'
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 4
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE;
```

- **Auto-scale** mode (`MIN < MAX`) starts and stops clusters with demand. **Maximized** mode (`MIN = MAX`) runs them all
  whenever the warehouse runs.
- **`STANDARD`** (the default) favours starting clusters to avoid queuing. It adds a cluster when a query queues or Snowflake
  estimates the running clusters can't take more, and removes the least-loaded clusters after a sustained period of low load.
- **`ECONOMY`** favours full clusters over speed. It adds a cluster only if Snowflake estimates enough load to keep it busy for
  **at least 6 minutes**, and marks one for shutdown when it has less than 6 minutes of work left. Queries may queue longer.

Each running cluster bills like a separate warehouse of that size. Per the documentation's example, a Medium warehouse with up to
3 clusters can consume at most 12 credits per hour. Use `ECONOMY` for batch-ish workloads that can tolerate some queuing, and
`STANDARD` for interactive dashboards.

## Query acceleration

The **Query Acceleration Service (QAS)** offloads parts of eligible queries to shared serverless compute. Per the documentation,
it targets **large scans with selective filters** and **large inserts, copies, updates and deletes**. Results depend on
server availability, so gains can vary over time. It's billed as its own service type. `QUERY_ACCELERATION_MAX_SCALE_FACTOR`
caps how much it can lease: per the documentation, a scale factor of 5 on a Medium warehouse (4 credits per hour) allows up
to 20 extra credits per hour. The default is 2.

```sql
-- which past queries would have benefited?
SELECT query_id, warehouse_name, upper_limit_scale_factor, eligible_query_acceleration_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_ACCELERATION_ELIGIBLE
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY eligible_query_acceleration_time DESC
LIMIT 20;

ALTER WAREHOUSE reporting_wh SET ENABLE_QUERY_ACCELERATION = TRUE QUERY_ACCELERATION_MAX_SCALE_FACTOR = 4;
```

Per the documentation, QAS is **enabled by default** on Gen2 standard warehouses and multi-cluster warehouses, so check
`SHOW WAREHOUSES` before assuming it's off, and watch the `QUERY_ACCELERATION` line in your metering.

## Warehouse types

| Type | What it is | When |
|---|---|---|
| **Standard Gen1** (`RESOURCE_CONSTRAINT = STANDARD_GEN_1`) | the original hardware | existing workloads, sizes up to 6X-Large |
| **Standard Gen2** (`GENERATION = '2'` / `STANDARD_GEN_2`) | newer hardware and faster DML and scans; per the docs, the default for new standard warehouses where available | most new workloads; not available for 5X- and 6X-Large |
| **Snowpark-optimized** (`WAREHOUSE_TYPE = 'SNOWPARK-OPTIMIZED'`) | much more memory per node, optional x86 (`MEMORY_16X_x86`) | memory-hungry Python: ML training, large UDF libraries |
| **Adaptive** (`CREATE ADAPTIVE WAREHOUSE`) | a shared, account-dedicated compute pool that sizes, scales and routes each query itself | workloads where you'd rather set limits than sizes (Enterprise+, select regions) |

Gen2 credit rates differ from Gen1. They're in the Service Consumption Table, so compare cost per **job**, not per hour.

**Adaptive Warehouses** are the biggest change. Per the documentation (GA in select regions as of October 2026), you no longer
choose a size, cluster counts, QAS settings or suspend behaviour. You set two limits instead:
`MAX_QUERY_PERFORMANCE_LEVEL` (the most compute any single query may get, expressed as a size, default `XLARGE`) and
`QUERY_THROUGHPUT_MULTIPLIER` (how much total concurrent work it may run, relative to a system baseline, default 2). Billing is
**per query**, visible in `QUERY_METERING_HISTORY`, and budgets and resource monitors still apply.

```sql
CREATE ADAPTIVE WAREHOUSE analytics_awh
  WITH MAX_QUERY_PERFORMANCE_LEVEL = LARGE
       QUERY_THROUGHPUT_MULTIPLIER = 2;
```

The trade-off is control. The model above, and most of this lesson, is about choices an Adaptive Warehouse makes for you.
Try one alongside a standard warehouse on a copy of a real workload, and compare cost per job and p95 latency before moving
anything.

## How It Actually Works

A standard warehouse cluster is a fixed set of compute nodes. Size sets the number of nodes, and each size doubles them. A
query is split across the cluster's nodes, so a bigger size helps only if the query has enough work to divide: large scans,
joins and sorts benefit, small lookups don't. Up to `MAX_CONCURRENCY_LEVEL` queries (8 by default) share a cluster. When it's
saturated, new queries **queue**. A multi-cluster warehouse adds whole clusters, each with its own nodes and local cache, and
the services layer routes new queries to clusters with capacity. That's why scale-out fixes queuing without making any single
query faster, and why caches warm separately on each cluster. QAS lends extra workers for the scan-heavy fragments of one query.
An Adaptive Warehouse drops fixed clusters altogether and gives each query an allocation from a shared pool, within your
limits. The same principle runs underneath, matching compute to the parallelism a query can actually use, but it's decided
per query instead of per warehouse.

## Common mistakes

- **Scaling up to fix queuing.** It costs more per hour and leaves queries waiting if they don't speed up. Add clusters.
- **Scaling out to fix spilling.** More clusters of the same size spill the same way. Scale up.
- **`ECONOMY` for interactive dashboards.** Users wait for clusters that start only when there are 6 minutes of work.
- **Maximized mode by accident** (`MIN = MAX`), paying for all clusters whenever the warehouse runs.
- **Comparing Gen1 and Gen2 on credits per hour.** Compare credits per job.
- **Assuming QAS is off.** It's on by default for Gen2 and multi-cluster warehouses.

## Exercise

1. Run the queue and spill query for your warehouses. Classify each as healthy, needs scale-up or needs scale-out.
2. Run one heavy query (an aggregation over `SNOWFLAKE_SAMPLE_DATA.TPCH_SF100.LINEITEM`) on Small, Medium and Large with
   `USE_CACHED_RESULT = FALSE`. Compute its SPEEDUP and its credits for each size, then rerun `scaling_model.py` with that value.
3. Fire 50 concurrent copies of a dashboard query (for example with a small Python script and the connector) at a Small
   warehouse with `MAX_CLUSTER_COUNT = 1`, then 4. Compare queued time.
4. If Adaptive Warehouses are available in your region, run the same workload on one and compare cost per job from
   `QUERY_METERING_HISTORY` with your standard warehouse's `WAREHOUSE_METERING_HISTORY`.
