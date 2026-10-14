---
description: "Snowflake Level 2: pruning, clustering, Query Profile, caches, Snowpipe, streams, tasks, dynamic tables, unloading and Iceberg, and an ELT pipeline."
---

# Level 2 · Performance & Data Engineering

Level 2 is about two things: making queries read less, and keeping data up to date continuously without reprocessing
everything. Both come down to how Snowflake stores data in micro-partitions. Along the way you'll see a one-day query
read 1 of 98 partitions or all 98 depending only on row order, a text clustering key that can't tell any SKUs apart, a
join that silently made revenue six times too high, a warehouse that would wake 288 times a day to merge nothing, and a
`MERGE` that inserted the same order twice.

!!! note "What was run"
    As in Level 1, nothing here was run on Snowflake. Pruning, clustering layouts and partitioned exports were
    demonstrated for real with Parquet row-group statistics in DuckDB 1.5.6, which work on the same min/max principle.
    The project's `MERGE` logic was tested locally against seven batches of changes. Costs are small models of the
    documented billing rules, with their assumptions stated. All Snowflake SQL was syntax-checked and verified against the
    October 2026 documentation, including several features that changed in 2025–2026 (Snowpipe's per-GB pricing, task
    overlap policies, dynamic table refresh modes).

**Prerequisites:** Level 1.

## Modules

1. **[Micro-Partitions & Pruning](01-micro-partitions-pruning.md)** — min/max metadata, pruning shown with real Parquet
   statistics, overlap and depth, and filters that can and can't prune.
2. **[Clustering Keys & Automatic Clustering](02-clustering.md)** — choosing keys and their order, the text-prefix
   trap, the cost trade-off, sort-on-load and search optimization.
3. **[Query Profile & Performance Tuning](03-query-profile-tuning.md)** — queuing, exploding joins, `UNION`, spilling and
   pruning, and a workflow to find the queries worth tuning.
4. **[The Three Caches](04-caching-layers.md)** — result, warehouse and metadata caches, their exact rules, and the
   auto-suspend trade-off modelled.
5. **[Continuous Loading with Snowpipe](05-snowpipe.md)** — auto-ingest pipes, per-GB billing, the 14-day re-staging trap
   and Snowpipe Streaming.
6. **[Streams: Change Data Capture](06-streams-cdc.md)** — net-change semantics, consuming with DML, transactions,
   staleness and the `CHANGES` clause.
7. **[Tasks & Task Graphs](07-tasks.md)** — schedules, stream-gated and triggered tasks, serverless compute, graphs with
   finalizers, and failure handling.
8. **[Dynamic Tables](08-dynamic-tables.md)** — declarative pipelines, target lag, refresh modes and the silent fallback to
   full refresh.
9. **[Unloading, External Tables & Iceberg](09-unloading-iceberg.md)** — partitioned unloads, external tables over your
   files, and Snowflake-managed vs externally managed Iceberg tables.
10. **[Project — An Incremental ELT Pipeline](10-project-incremental-pipeline.md)** — stream, deduplicating `MERGE` task and
    incremental dynamic table, with the merge logic tested against full rebuilds.

## After Level 2

You can build fast queries and continuous pipelines. Level 3 is about who may see and do what: roles and privileges,
authentication, network controls, masking and row access policies, tags, secure views, data sharing, encryption, and
auditing with account usage and access history, ending with a governed multi-team platform.
