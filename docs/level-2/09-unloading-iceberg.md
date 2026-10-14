---
description: "Getting data out of Snowflake: partitioned COPY INTO a stage, external tables over your files, and Iceberg tables managed by Snowflake or a catalog."
---

# 09 · Unloading, External Tables & Iceberg

So far, data has gone **into** Snowflake's own storage. Two other directions matter in real platforms: getting data
**out** (to feed another system, a partner, or a data lake), and querying data that **stays** in your own cloud storage
in open formats, either read-only through external tables or fully managed through **Apache Iceberg** tables, which
other engines can read too.

## Unloading with COPY INTO a location

The same `COPY` command runs in reverse: from a table or query into a stage.

```sql
CREATE FILE FORMAT IF NOT EXISTS learn_db.raw.parquet_fmt TYPE = PARQUET;

COPY INTO @learn_db.raw.s3_landing/exports/paid_orders/
FROM (
    SELECT o.*, TO_CHAR(ordered_at, 'YYYY-MM') AS month
    FROM learn_db.raw.orders o
    WHERE status = 'paid'
)
PARTITION BY ('month=' || month || '/channel=' || channel)
FILE_FORMAT = (FORMAT_NAME = 'learn_db.raw.parquet_fmt')
MAX_FILE_SIZE = 268435456          -- target 256 MB per file (default 16 MB, maximum 5 GB)
HEADER = TRUE                      -- keep column names in Parquet
INCLUDE_QUERY_ID = TRUE;           -- unique file names, so concurrent unloads can't overwrite each other
```

From the `COPY INTO <location>` reference:

- Unloading runs **in parallel**, so it produces many files whose number and sizes depend on the data and the warehouse.
  `MAX_FILE_SIZE` is a target per thread (default 16 MB, maximum 5 GB).
- `SINGLE = TRUE` writes one file, which uses one thread: slow for large data, and still subject to the size limit.
- `PARTITION BY` takes an expression that evaluates to a string and uses it as a path prefix. That's the usual way to
  produce `key=value/` directory layouts that other engines recognize as partitions.

Here's what such a layout looks like and how a reader benefits from it. DuckDB 1.5.6 locally (not Snowflake) wrote the
course's paid orders partitioned by month and channel:

```text
27 files; first three:
  export/month=2026-01/channel=app/data_0.parquet
  export/month=2026-01/channel=store/data_0.parquet
  export/month=2026-01/channel=web/data_0.parquet
```

```sql
-- DuckDB 1.5.6 (local), not Snowflake
SELECT COUNT(*) AS paid_web_june FROM read_parquet('export/**/*.parquet', hive_partitioning=true)
WHERE month = '2026-06' AND channel = 'web';
```

```text
paid_web_june: 4408
```

A reader filtering on partition columns opens only the matching directory, one file out of 27 here. That's partition
pruning at the directory level, the file-system version of lesson 01's min/max pruning. To download unloaded files from
an **internal** stage, use `GET` (or `snow stage copy @stage/path ./local/`).

## External tables: query files where they are

An **external table** puts a table definition over files in an external stage. The data stays in your bucket and
Snowflake reads it at query time:

```sql
CREATE EXTERNAL TABLE learn_db.raw.orders_lake (
    month      VARCHAR       AS (SPLIT_PART(SPLIT_PART(METADATA$FILENAME, 'month=', 2), '/', 1)),
    order_id   NUMBER        AS (VALUE:order_id::NUMBER),
    channel    VARCHAR       AS (VALUE:channel::VARCHAR),
    amount     NUMBER(12,2)  AS (VALUE:quantity::NUMBER * VALUE:unit_price::NUMBER(10,2))
)
PARTITION BY (month)
LOCATION = @learn_db.raw.s3_landing/exports/paid_orders/
FILE_FORMAT = (FORMAT_NAME = 'learn_db.raw.parquet_fmt')
AUTO_REFRESH = TRUE;
```

Every row is exposed as a `VALUE` variant, and columns are expressions over it. Partition columns derived from
`METADATA$FILENAME` let Snowflake skip whole directories. `AUTO_REFRESH` keeps the file list current using cloud
notifications, like Snowpipe. External tables are **read-only** and generally slower than native tables (no
micro-partition statistics of Snowflake's own, and files read from your storage each time). They're good for exploring a
lake, for rarely queried archives, and as a staging point. For data you query often, load it, or use Iceberg.

## Apache Iceberg tables

**Iceberg** is an open table format: Parquet data files plus metadata files that track snapshots, schemas and
partitions, so many engines (Spark, Trino, DuckDB, Flink, Snowflake) can read and write the same table. Snowflake supports
Iceberg tables in two forms, per its documentation:

| | Snowflake-managed (Snowflake is the catalog) | Externally managed (another catalog) |
|---|---|---|
| Writes from Snowflake | yes, full DML | limited / read-focused |
| Table maintenance (e.g. compaction) | Snowflake does it | the external catalog/engine |
| Data location | your bucket (via an **external volume**) or Snowflake storage | your bucket |
| Other engines read it via | the Snowflake Horizon Catalog's Iceberg REST API, or Open Catalog sync | their own catalog |
| Needs | external volume (or `SNOWFLAKE_MANAGED`) | external volume + **catalog integration** |

A Snowflake-managed Iceberg table in your own S3 bucket:

```sql
CREATE EXTERNAL VOLUME lake_vol
  STORAGE_LOCATIONS = ((
    NAME = 'lake-eu'
    STORAGE_PROVIDER = 'S3'
    STORAGE_BASE_URL = 's3://my-company-lake/iceberg/'
    STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-lake'
  ));

CREATE ICEBERG TABLE learn_db.analytics.orders_iceberg (
    order_id NUMBER(38,0), ordered_at TIMESTAMP_NTZ, channel VARCHAR, amount NUMBER(12,2), status VARCHAR
)
  CATALOG = 'SNOWFLAKE'
  EXTERNAL_VOLUME = 'lake_vol'
  BASE_LOCATION = 'orders_iceberg';

INSERT INTO learn_db.analytics.orders_iceberg
SELECT order_id, ordered_at, channel, quantity * unit_price, status FROM learn_db.raw.orders;
```

From then on it behaves much like a normal table in Snowflake (queries, DML, Time Travel within Iceberg's snapshot
model), but its data and metadata are open files in your bucket that other engines can read. The cost model shifts
too: you pay your cloud provider for that storage, rather than Snowflake.

Iceberg support is an area of fast change in Snowflake (catalog options, REST catalog access, supported types and
operations). Snowflake's own documentation index advises preferring the current SQL reference over remembered syntax
for Iceberg tables. Check it before relying on a specific capability.

## How It Actually Works

An unload is a parallel query whose final operator writes files instead of returning rows. Each warehouse thread writes
its share, rolling to a new file when it reaches `MAX_FILE_SIZE`, which is why you get many files by default and one only
with `SINGLE`. `PARTITION BY` routes each row to a file stream keyed by the expression's value.

An external table is metadata over a list of files. Snowflake keeps the file list (refreshed by notifications or
`ALTER EXTERNAL TABLE … REFRESH`) and, at query time, applies partition-column filters to the file paths, then reads the
surviving files from your storage. An Iceberg table keeps its own metadata tree (a metadata file pointing to manifest
lists, manifests and data files, each manifest carrying column statistics), which is conceptually the same thing
Snowflake does internally for micro-partitions, written in an open format. When Snowflake is the catalog, it writes and
maintains that tree. When another catalog is, Snowflake reads the tree that catalog points to.

## Common mistakes

- **`SINGLE = TRUE` for large unloads.** One thread is slow, and the 5 GB file limit still applies.
- **Re-running an unload into the same path** and mixing old and new files. Use a fresh prefix, or `INCLUDE_QUERY_ID`
  plus an `OVERWRITE` strategy.
- **Heavy, frequent queries on external tables.** Load the data or use Iceberg.
- **No partition columns on external tables over large directory trees.** Every query lists and reads everything.
- **Assuming every Snowflake feature works identically on Iceberg tables.** Check the current reference.
- **Forgetting egress.** Unloading across regions or clouds incurs data-transfer charges.

## Exercise

1. Unload the paid orders to an internal stage partitioned by month and channel, then use `LIST` and `GET` (or `snow stage
   copy`) to download them and read them with DuckDB or pandas.
2. Create an external table over the unloaded files with a `month` partition column, and compare the
   `partitions_scanned` and elapsed time of a one-month query against the native table.
3. Create a Snowflake-managed Iceberg table on a trial account's external volume (or with
   `EXTERNAL_VOLUME = 'SNOWFLAKE_MANAGED'`), insert the orders, and look at the files that appear in the storage location.
4. Write down which of your organization's datasets should be native tables, external tables, or Iceberg tables, and why.
