---
description: "Snowflake Level 1: architecture, clients, objects and types, warehouses and credits, loading, Snowflake SQL, JSON, Time Travel, cloning and a project."
---

# Level 1 · Snowflake Foundations

Level 1 takes you from "Snowflake is a cloud data warehouse" to building and querying a small, well-organized
database. Along the way it covers the decisions that determine your bill and your data quality from day one: how
long a warehouse idles after a burst of work, why 199,997 of 200,000 rows load, why a `FLATTEN` quietly drops a
quarter of your customers, why a permanent table rewritten nightly holds nine times its size, and why updating
0.001% of a table can rewrite 39% of it.

!!! note "What was run, and what wasn't"
    This course was written without access to a Snowflake account, and no output is presented as Snowflake output.
    Every SQL statement was parsed with `sqlglot`'s Snowflake dialect and checked against Snowflake's documentation.
    Results shown for queries come from running the same SQL on the same generated data in **DuckDB 1.5.6**, and are
    labelled as such. Cost and storage figures come from small Python models of the documented billing rules, also
    labelled. Snowflake CLI 3.29.0 and the Python connector were run locally for everything that doesn't contact
    Snowflake. A free trial account is enough to run every exercise yourself.

**Prerequisites:** comfortable with SQL (joins, `GROUP BY`; see the
[SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/)) and basic Python for the client and project lessons.

## Modules

1. **[How Snowflake Is Built: Storage, Compute & Services](01-architecture.md)** — the three layers, why they're
   separate, credits, the 10% cloud-services rule and editions.
2. **[Accounts, Snowsight, Snowflake CLI & Python](02-accounts-and-clients.md)** — account identifiers, session context,
   Snowflake CLI connections with key-pair auth, and the Python connector.
3. **[Databases, Schemas, Tables & Data Types](03-objects-and-types.md)** — the upper-case identifier rule, permanent vs
   transient vs temporary tables, types that matter, and unenforced constraints.
4. **[Virtual Warehouses: Sizing, Suspend & Credits](04-virtual-warehouses.md)** — sizes and rates, risky defaults, and a
   billing model where 160 seconds of work cost up to 4,114 seconds.
5. **[Loading Data: Stages, File Formats & COPY INTO](05-loading-copy-into.md)** — stages, file formats, validation,
   `ON_ERROR`, 64-day load metadata and file sizing, on a dataset with planted errors.
6. **[Snowflake SQL: Windows, QUALIFY & Extensions](06-snowflake-sql.md)** — `QUALIFY`, ranking with ties, running totals,
   `GROUP BY ALL`, `EXCLUDE`, `PIVOT`, `MERGE` and `ASOF JOIN`.
7. **[Semi-Structured Data: VARIANT & FLATTEN](07-semi-structured.md)** — paths and casts, SQL `NULL` vs JSON null, the
   `OUTER` flag, and how VARIANT is stored.
8. **[Time Travel, UNDROP & Fail-safe](08-time-travel-failsafe.md)** — querying and restoring the past, retention by
   edition, and the storage that history holds.
9. **[Zero-Copy Cloning & Environments](09-zero-copy-cloning.md)** — clones for dev and test, what gets cloned, and when
   scattered updates make a clone expensive.
10. **[Project — Load & Analyse a Sales Dataset](10-project-sales-analytics.md)** — an end-to-end script, an answer key
    and an automated checker.

## After Level 1

You can build, load and query a Snowflake database correctly. Level 2 is about doing it fast and continuously: how
micro-partitions and pruning decide what a query reads, clustering, the Query Profile, the three caches, and the
continuous-data features (Snowpipe, streams, tasks and dynamic tables), ending with an incremental pipeline.
