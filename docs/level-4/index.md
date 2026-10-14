---
description: "Snowflake Level 4: UDFs, Snowpark, Python UDFs, Cortex AI, cost management, scaling, replication, dbt CI/CD, migration and a cost-aware capstone."
---

# Level 4 · Building, Scaling & Operating

Level 4 is about building on Snowflake and running it well: code that runs next to the data, AI functions in SQL, and the
operational disciplines (cost, scaling, continuity, deployment, migration) that decide whether a platform stays healthy as
it grows. Along the way you'll see a "vectorized" handler run more than twice as slowly as the plain one, label descriptions
multiply an AI function's input tokens about six times, 86% of a warehouse bill attributed to nobody, four small clusters
beat a larger warehouse when queries don't scale with size, a dbt incremental model pass every test while serving stale
revenue, and translated SQL that compiles but returns different answers.

!!! note "What was run"
    Nothing here was run on Snowflake. Snowpark code ran in Snowpark's own **local testing mode** (Snowpark 1.53.0), a dbt
    project ran with dbt-core 1.12.5 on DuckDB 1.5.6, UDF handlers and a UDTF ran as plain Python, and the capstone's
    chargeback query ran in DuckDB with pytest and a mutation check. Costs, scaling and data loss come from small models
    with their assumptions stated. All Snowflake SQL was syntax-checked and verified against the October 2026 documentation,
    including features that are new in 2025–2026: Cortex AI functions and model access control, Adaptive Warehouses,
    Gen2 as the default, Optimized Refresh and RPO Assurance, and dbt Projects on Snowflake.

**Prerequisites:** Levels 1–3, and Python for lessons 02, 03 and 10.

## Modules

1. **[UDFs & Stored Procedures](01-udfs-procedures.md)** — SQL UDFs and table functions, Snowflake Scripting with
   transactions, and owner's vs caller's rights.
2. **[Snowpark for Python](02-snowpark-python.md)** — lazy DataFrames pushed down as SQL, the `to_pandas` trap, pandas on
   Snowflake, and unit tests in local testing mode.
3. **[Python UDFs, Vectorized UDFs & UDTFs](03-python-udfs.md)** — runtimes and packages, external access, when batching
   helps (measured), and a sessionizing table function.
4. **[Cortex AI Functions](04-cortex-ai.md)** — classify, filter, sentiment and structured extraction in SQL, access
   control, data residency and token costs.
5. **[Cost Management: Monitors, Budgets & Attribution](05-cost-management.md)** — where credits go, the idle gap in
   per-query attribution, tags, resource monitors and budgets.
6. **[Warehouse Scaling: Up, Out & Adaptive](06-warehouse-scaling.md)** — scale-up vs scale-out modelled, multi-cluster
   policies, query acceleration, Gen2 and Adaptive Warehouses.
7. **[Replication, Failover & Business Continuity](07-replication-bcdr.md)** — replication and failover groups, refresh
   modes, data loss modelled, Client Redirect and a runbook.
8. **[dbt & CI/CD for Snowflake](08-dbt-cicd.md)** — a tested dbt project, an incremental bug only a rebuild comparison
   caught, clones per pull request, and dbt Projects on Snowflake.
9. **[Migrating to Snowflake](09-migration.md)** — translation and its semantic traps, moving data, and reconciliation
   that catches export bugs.
10. **[Capstone — A Cost-Aware Data Platform](10-capstone-cost-aware-platform.md)** — owned warehouses, monitors and
    budgets, and a tested monthly chargeback that assigns every credit.

## After Level 4

You've covered Snowflake from architecture to operations. Good next steps: run the capstone in a trial account with your
own workloads, take the [dbt and data engineering](https://sigilipelli.github.io/data-engineering-mastery-path/) path
further, or pair this course with the [PySpark Mastery Path](https://sigilipelli.github.io/pyspark-mastery-path/) to
compare the two platforms on the same pipeline.
