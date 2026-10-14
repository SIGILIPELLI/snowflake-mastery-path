---
description: "How Snowflake separates storage, compute and cloud services, what that means for concurrency and cost, and how editions and credits fit together."
---

# 01 · How Snowflake Is Built: Storage, Compute & Services

Snowflake is a cloud data platform: a SQL database for analytics that runs only as a managed service on AWS, Azure
or Google Cloud. You never install it, size disks or tune a buffer pool. You also never get to look underneath, which
is why the most useful thing to learn first is the *shape* of the system. Almost every Snowflake behaviour that
surprises people (why two teams can query the same table without slowing each other down, why a query can be instant
the second time, why an idle warehouse still costs money, why you can undo a `DROP TABLE`) follows from one design
decision: **storage, compute and coordination are three separate layers**, each scaled and billed on its own.

!!! note "About this course and what was run"
    Snowflake has no local edition. Everything runs in a Snowflake account in the cloud, and this course was written
    **without** access to one. So no output in this course is presented as coming from Snowflake. Instead:

    - Every SQL statement is written against Snowflake's documented syntax and was **parsed with the Snowflake
      dialect of `sqlglot` 30.22** to catch syntax mistakes. Statements that `sqlglot` can only partly parse (such as
      `CREATE STREAM` and `CREATE TASK`) were checked against Snowflake's documentation instead.
    - Where a concept can be shown with real computation (window functions, `QUALIFY`, JSON flattening, min/max
      pruning, billing arithmetic), it is shown with **DuckDB 1.5.6** or a small Python model on a laptop, and
      labelled that way. DuckDB isn't Snowflake, so its output format, and sometimes its behaviour, differ.
    - Facts that change over time (prices, credit rates, feature status, deadlines) are taken from Snowflake's
      documentation as of October 2026, and you should re-check them. Snowflake ships changes every few weeks.

    With a free trial account, you can run every statement yourself. The lessons tell you what to look for.

## The three layers

```text
┌───────────────────────────────────────────────────────────────────┐
│ CLOUD SERVICES: authentication, metadata, query parsing and         │
│ optimization, transactions, access control, result cache            │
└───────────────────────────────────────────────────────────────────┘
      │                │                  │
┌───────────┐   ┌────────────┐   ┌────────────────┐
│ Warehouse │   │ Warehouse  │   │ Warehouse      │  ← COMPUTE: independent clusters,
│ BI_WH (S) │   │ ETL_WH (L) │   │ DS_WH (XS)     │    start/stop/resize at will
└───────────┘   └────────────┘   └────────────────┘
      │                │                  │
┌───────────────────────────────────────────────────────────────────┐
│ STORAGE: tables as immutable, compressed, columnar micro-partition │
│ files in the cloud provider's object storage                       │
└───────────────────────────────────────────────────────────────────┘
```

**Storage.** When you load data, Snowflake reorganizes it into its own columnar format and writes it as
**micro-partitions**: immutable files of 50–500 MB of uncompressed data each (smaller once compressed), stored in the
cloud provider's object storage (S3, Azure Blob or GCS) in Snowflake's account, not yours. You can't see these files
directly. For every micro-partition, Snowflake records metadata such as each column's minimum and maximum value and
the number of distinct values. That metadata is what lets queries skip most of a table (Level 2 · 01). Because files
are never modified in place, an `UPDATE` writes new micro-partitions and retires old ones. Keeping the retired ones for
a while is what makes Time Travel and cloning possible (lessons 08 and 09).

**Compute.** Queries run on **virtual warehouses**: clusters of cloud servers that Snowflake provisions for you, in
T-shirt sizes from X-Small to 6X-Large. A warehouse holds no data. It reads micro-partitions from storage, caches
them on local SSD while it runs, and can be suspended (costing nothing) and resumed in seconds. Different workloads
can use different warehouses against **the same tables** without competing for CPU. A nightly ETL job on a Large
warehouse doesn't slow down the BI dashboard on a Small one. That's the central promise of the architecture.

**Cloud services.** A shared, always-on layer handles everything that isn't scanning data: logins, the metadata
catalogue, parsing and optimizing SQL, transaction management, access control, and the **result cache**, which can
return an identical query's results without starting a warehouse at all (Level 2 · 04). Some operations, like `SHOW
TABLES`, `COUNT(*)` on a whole table, or `MIN`/`MAX` of a column, can often be answered from metadata alone.

## Why separation matters, concretely

| Situation | Traditional shared-nothing warehouse | Snowflake |
|---|---|---|
| ETL load runs while analysts query | Both compete for the same nodes | Separate warehouses, no CPU contention |
| Data grows 10× | Add nodes and rebalance data | Storage grows on its own, and compute is unchanged |
| A big query once a month | Size the cluster for the peak | Resize a warehouse for that query, then shrink it |
| Nobody is querying at night | Cluster still runs (and bills) | Warehouses suspend, so compute costs nothing |
| A new team needs the same data | Copy data to their cluster | Point their warehouse at the same tables |

What separation doesn't remove is **contention for the same warehouse**. Twenty analysts sharing one Small warehouse
queue behind each other. Level 4 · 06 covers multi-cluster warehouses, which add clusters as queries queue.

## How you pay: credits

Compute is billed in **credits**. A running warehouse consumes credits per second (with a 60-second minimum each time
it starts) at a rate that doubles with each size step. That's 1 credit per hour for X-Small, 2 for Small, 4 for
Medium, and so on up to 512 for 6X-Large, for standard "Gen1" warehouses (Lesson 04 has the table). The dollar price of
a credit depends on your edition, cloud, region and contract, so this course talks in credits.

Other charges:

- **Storage**: a monthly rate per terabyte, on the compressed size, including data kept for Time Travel and Fail-safe
  (lesson 08). It's usually small compared with compute.
- **Cloud services**: charged only for the part of a day's cloud-services usage that exceeds **10% of that day's
  warehouse usage**. In most accounts that's nothing. Workloads with huge numbers of tiny queries, or heavy use of
  metadata commands, can exceed it.
- **Serverless features** (Snowpipe, automatic clustering, search optimization, serverless tasks and others) run on
  Snowflake-managed compute, each with its own rate. They don't count toward the 10% cloud-services allowance.
- **Data transfer** out of the region or cloud.

Level 4 · 05 shows how to see where credits go and how to cap them.

## Editions

The edition decides which features you get (and the credit price). From Snowflake's documentation:

| Feature | Standard | Enterprise | Business Critical | Virtual Private Snowflake |
|---|---|---|---|---|
| Time Travel retention | 1 day | up to 90 days | up to 90 days | up to 90 days |
| Multi-cluster warehouses | — | ✓ | ✓ | ✓ |
| Materialized views, column-level security (masking) | — | ✓ | ✓ | ✓ |
| Tri-Secret Secure (customer-managed key) | — | — | ✓ | ✓ |
| Private connectivity (PrivateLink) | — | — | ✓ | ✓ |
| Failover and failback between accounts | — | — | ✓ | ✓ |
| Isolated environment (dedicated metadata store and compute) | — | — | — | ✓ |

Several lessons depend on Enterprise features (masking policies, 90-day Time Travel, multi-cluster warehouses), and
they say so. A trial account typically lets you choose an edition.

## Where Snowflake fits

Snowflake is built for **analytics**: large scans, aggregations and joins over historical data, many concurrent
readers, and batch or micro-batch loading. It's a poor fit as the primary database behind an application that does
thousands of single-row inserts and updates per second. Use PostgreSQL (see the
[PostgreSQL Mastery Path](https://sigilipelli.github.io/postgresql-mastery-path/)) or similar, and load Snowflake from
it. (Snowflake does offer hybrid tables for row-oriented workloads. They're out of scope here.)

This course assumes you know SQL. If `JOIN`, `GROUP BY` and window functions are new to you, start with the
[SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/). Python appears in the Snowpark and connector
lessons, and the [Python Mastery Path](https://sigilipelli.github.io/python-mastery-path/) covers what you need.

## How It Actually Works

When you submit a query, the **cloud services** layer authenticates you, checks your role's privileges, parses and
optimizes the SQL, and consults the **metadata** for every micro-partition of every table involved. Using each
partition's min/max values, it eliminates partitions that can't contain matching rows, often the vast majority. Only
then does it hand a plan, plus the list of surviving micro-partition files, to a **warehouse**. The warehouse's nodes
fetch those files from object storage (or their local SSD cache if they've read them recently), decompress only the
columns the query needs, execute the plan in parallel, and return results through the services layer, which also
stores them in the **result cache** for 24 hours.

A write follows the same path in reverse. The warehouse produces new micro-partition files, and the services layer
**commits** the change by atomically updating the metadata to point at the new files instead of the old ones. Nothing is
overwritten. The old files remain referenced by Time Travel until their retention expires, then by Fail-safe, and are
deleted afterwards. That copy-on-write design is why writes are transactional, why many readers never block writers,
and why "undo" features cost only storage.

Because warehouses are stateless with respect to data, a warehouse can be suspended between queries, resized
between statements, or given extra clusters, and the data doesn't move. The trade-off is that every warehouse
start begins with a cold local cache. Lesson 04 and Level 2 · 04 show what that costs.

## Common mistakes

- **One warehouse for everything.** You lose the isolation that separation provides. Give ETL, BI and data science
  their own warehouses.
- **Treating Snowflake like an OLTP database.** Single-row inserts in a loop are slow and expensive. Load in batches.
- **Thinking storage is the bill.** Compute (warehouses running) usually dominates. An idle-but-running warehouse is
  the classic waste.
- **Assuming every feature exists in every edition.** Masking policies and long Time Travel need Enterprise or higher.
- **Expecting to tune the database the old way.** There are no indexes or vacuum to manage. The levers are data
  layout (clustering), warehouse sizing and query design.

## Exercise

1. Sign up for a Snowflake trial in a region near you, and note your account's edition, cloud and region (Snowsight shows
   them in the account menu, or run `SELECT CURRENT_ACCOUNT(), CURRENT_REGION();`).
2. Draw the three-layer diagram for a company with a nightly load, a BI tool used by 50 people during office hours, and
   two data scientists. Which warehouses would you create, what size, and when would each be running?
3. Using the credit table in this lesson, work out the credits per day for a Medium warehouse that runs 9 hours a day
   on weekdays, and a Large one that runs 1 hour each night. Which costs more per month?
4. List three Snowflake features this course will rely on that need Enterprise Edition or above, and check whether your
   trial account has them.
