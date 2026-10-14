---
description: "Snowflake's object hierarchy, permanent, transient and temporary tables, identifier case rules, key data types and the constraints it doesn't enforce."
---

# 03 · Databases, Schemas, Tables & Data Types

Snowflake's objects nest in a fixed hierarchy, and most of the rules you'll trip over at the start are about names,
table kinds and types. Three of them catch nearly everyone: unquoted identifiers are stored in **upper case**, primary
keys are **not enforced**, and there are three timestamp types that behave differently.

## The hierarchy

```text
Organization
└── Account
    ├── Users, Roles, Warehouses, Integrations, Network policies   (account-level objects)
    └── Database
        └── Schema
            ├── Tables, Views, Materialized views, Dynamic tables
            ├── Stages, File formats, Pipes, Streams, Tasks
            ├── Sequences, Functions (UDFs), Procedures
            └── Policies (masking, row access), Tags
```

A fully qualified name has three parts: `database.schema.object`. Warehouses, users and roles live at the account level
and have one-part names.

```sql
CREATE DATABASE IF NOT EXISTS learn_db COMMENT = 'Snowflake Mastery Path labs';
CREATE SCHEMA IF NOT EXISTS learn_db.raw;
CREATE SCHEMA IF NOT EXISTS learn_db.analytics WITH MANAGED ACCESS;
```

`WITH MANAGED ACCESS` makes the schema owner (not each object's owner) control grants inside it. That's a good default for
shared schemas (Level 3 · 01).

## Identifiers: the upper-case rule

Unquoted identifiers are **case-insensitive and stored in upper case**. Quoted identifiers are **case-sensitive and stored
exactly as written**:

```sql
CREATE TABLE learn_db.raw.Customers (id INT);          -- stored as CUSTOMERS
SELECT * FROM learn_db.raw.customers;                   -- works: resolves to CUSTOMERS
CREATE TABLE learn_db.raw."orderItems" (id INT);       -- stored as orderItems
SELECT * FROM learn_db.raw.orderItems;                  -- fails: looks for ORDERITEMS
SELECT * FROM learn_db.raw."orderItems";                -- works
```

This bites most often when tools create tables with quoted lower-case or mixed-case names (some loaders and DataFrame
writers do), and then every query needs quotes. Prefer unquoted `snake_case` names and let them become upper case. The
`QUOTED_IDENTIFIERS_IGNORE_CASE` parameter changes this behaviour account-wide, and it's best decided before any objects
exist.

## Three kinds of table

```sql
CREATE TABLE learn_db.raw.orders_history (order_id NUMBER, amount NUMBER(12,2));      -- permanent
CREATE TRANSIENT TABLE learn_db.raw.orders_staging (order_id NUMBER, amount NUMBER(12,2));
CREATE TEMPORARY TABLE session_scratch (order_id NUMBER, amount NUMBER(12,2));
```

| | Permanent | Transient | Temporary |
|---|---|---|---|
| Lives until | dropped | dropped | the session ends |
| Visible to | anyone with privileges | anyone with privileges | only the creating session |
| Time Travel | up to 90 days (Enterprise+), 1 on Standard | 0 or 1 day | 0 or 1 day |
| Fail-safe (7 days) | yes | **no** | **no** |

Transient tables exist to save storage on data you can rebuild: staging tables, intermediate ETL results, scratch
tables. A permanent table that you rewrite completely every hour keeps every old version in Time Travel and then in
Fail-safe, and you pay storage for all of it (lesson 08). Transient databases and schemas make everything created in
them transient by default.

## Data types worth knowing

| Type | Notes |
|---|---|
| `NUMBER(p, s)` | Exact decimal. `INT`, `INTEGER`, `BIGINT` are all aliases for `NUMBER(38,0)`. Use it for money. |
| `FLOAT` / `DOUBLE` | 64-bit binary floating point. Fine for measurements, wrong for money. |
| `VARCHAR(n)` | Default length 16,777,216 characters, maximum 128 MB. The declared length has **no storage or performance cost**, so don't agonize over it. |
| `BINARY` | Default 8 MB, maximum 64 MB. |
| `BOOLEAN` | `TRUE` / `FALSE` / `NULL`. |
| `DATE`, `TIME` | As expected. |
| `TIMESTAMP_NTZ` | "No time zone": wall-clock time, no zone stored or applied. |
| `TIMESTAMP_LTZ` | Stored as UTC, *displayed* in the session's `TIMEZONE`. |
| `TIMESTAMP_TZ` | Stores the time **and** its UTC offset. |
| `VARIANT`, `OBJECT`, `ARRAY` | Semi-structured data (lesson 07). A single value can be up to 128 MB. |
| `GEOGRAPHY`, `GEOMETRY` | Spatial types. |
| `VECTOR(FLOAT, n)` | Fixed-length embedding vectors (Level 4 · 04). |

Plain `TIMESTAMP` is an alias whose meaning is set by the `TIMESTAMP_TYPE_MAPPING` parameter (`TIMESTAMP_NTZ` by
default). That means two accounts with different settings interpret the same DDL differently. Name the specific type
in DDL. The usual choice is `TIMESTAMP_NTZ` holding **UTC** for event times, or `TIMESTAMP_TZ` when the original
offset matters.

### Why NUMBER for money: a demonstration

Snowflake's `FLOAT` is IEEE-754 binary floating point, like Python's `float` and most databases' `DOUBLE`. The rounding
it produces can be shown on any engine. Here's DuckDB 1.5.6 locally (not Snowflake), summing 0.1 a million times:

```sql
-- DuckDB 1.5.6 (local), not Snowflake
SELECT SUM(0.1::DOUBLE) AS sum_double, SUM(0.1::DECIMAL(38,2)) AS sum_decimal
FROM range(1000000);
```

```text
┌────────────────────┬───────────────┐
│     sum_double     │  sum_decimal  │
├────────────────────┼───────────────┤
│ 100000.00000000087 │     100000.00 │
└────────────────────┴───────────────┘
```

In Snowflake, the same principle applies to `FLOAT` versus `NUMBER(38,2)`. Exact output digits may differ, because the
order of floating-point additions in a parallel engine isn't fixed. That's a second reason not to use floats for
anything that must reconcile.

## Constraints: declared, not enforced

```sql
CREATE TABLE learn_db.raw.customers (
    customer_id NUMBER(38,0) NOT NULL PRIMARY KEY,
    email       VARCHAR NOT NULL UNIQUE,
    country     VARCHAR(2),
    created_at  TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()::TIMESTAMP_NTZ
);
```

On standard Snowflake tables, **only `NOT NULL` is enforced**. `PRIMARY KEY`, `UNIQUE` and `FOREIGN KEY` are accepted and
recorded as metadata, and BI tools and the optimizer can read them, but inserting two rows with the same
`customer_id` succeeds. (Hybrid tables do enforce them. They're a separate, row-oriented table type not covered here.)
Two consequences:

- **Deduplication is your job.** Load pipelines must use `MERGE` or `QUALIFY ROW_NUMBER() = 1` (lesson 06), not rely on the
  table to reject duplicates.
- **Declaring constraints is still worthwhile.** It documents intent, and some tools use it for joins and lineage. Add
  `RELY` only if you guarantee the data really satisfies them, because it lets the optimizer trust them (for example, to
  eliminate unnecessary joins).

## Sequences and identity columns

```sql
CREATE SEQUENCE learn_db.raw.order_seq START = 1 INCREMENT = 1;

CREATE TABLE learn_db.raw.orders (
    order_id    NUMBER(38,0) DEFAULT learn_db.raw.order_seq.NEXTVAL,
    customer_id NUMBER(38,0) NOT NULL,
    amount      NUMBER(12,2) NOT NULL,
    ordered_at  TIMESTAMP_NTZ NOT NULL
);

CREATE TABLE learn_db.raw.events (
    event_id NUMBER(38,0) IDENTITY(1,1),
    payload  VARIANT
);
```

Sequence values are **unique and increasing, but not gap-free**, and under concurrency they aren't necessarily in commit
order. Never use them as a count, or as proof of ordering.

## How It Actually Works

All of these objects are **metadata** in the cloud services layer. Creating a database or schema writes catalogue
entries and touches no storage. A table is a catalogue entry plus a list of the micro-partition files that hold its
data, and that list is what changes on every commit. The differences between permanent, transient and temporary tables
are differences in how long retired micro-partitions are kept (Time Travel and Fail-safe) and in who can see the
catalogue entry (temporary tables are scoped to a session).

Types matter for storage because data is stored **per column**, compressed with encodings chosen per micro-partition.
Values of an exact `NUMBER` type are stored as integers scaled by the declared scale, and Snowflake picks a compact
physical width for the values actually present, which is why `NUMBER(38,0)` costs no more than a "smaller" integer
type. Strings are stored at their actual length, which is why `VARCHAR`'s declared maximum costs nothing. Constraints
other than `NOT NULL` would require checking every write against existing data across many micro-partitions, which
is expensive in a scan-oriented, append-optimized engine. That's why they're informational.

## Common mistakes

- **Quoted mixed-case identifiers** from tools, forcing quotes forever. Standardize on unquoted names.
- **Trusting `PRIMARY KEY`** to prevent duplicates. It doesn't.
- **Using plain `TIMESTAMP`** in DDL and getting `NTZ` in one account and `LTZ` in another. Name the type.
- **Storing local wall-clock times** without a zone in `TIMESTAMP_NTZ`, and then being unable to compare across regions.
  Store UTC.
- **Permanent tables for disposable data.** You pay Fail-safe storage on every rewrite. Use transient tables.
- **`FLOAT` for money.** Use `NUMBER(p, s)`.

## Exercise

1. Create `Customers`, `"Customers"` and `"customers"` in one schema. How many tables exist? Write the `SELECT` that reads
   each one.
2. Insert two rows with the same primary key into `learn_db.raw.customers`, then write a query that finds all
   duplicated keys, and a `QUALIFY` query that keeps only the newest row per key.
3. Set your session `TIMEZONE` to `America/New_York`, insert `CURRENT_TIMESTAMP()` into columns of each timestamp type,
   change the session time zone to `Asia/Kolkata`, and select them again. Which values changed?
4. Create a transient schema and a table inside it, then check `SHOW TABLES` for its `kind` and `retention_time`.
