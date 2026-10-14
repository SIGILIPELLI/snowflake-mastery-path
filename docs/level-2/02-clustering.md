---
description: "Snowflake clustering keys and Automatic Clustering: choosing columns and order, cost, the text-prefix trap, sorting on load, and search optimization."
---

# 02 · Clustering Keys & Automatic Clustering

The last lesson showed that pruning depends on how rows are grouped into micro-partitions: a one-day filter read 1 of
98 partitions when the data was in time order and 98 of 98 when it wasn't. A **clustering key** tells Snowflake which
grouping you want, and **Automatic Clustering** keeps it that way in the background as data changes, for a price. This
lesson covers when that price is worth paying, how to choose the key, and the cheaper alternatives.

## No layout serves every filter

The same 200,000 orders written in four orders (98 row groups each, real Parquet min/max statistics, DuckDB locally),
with the number of row groups three filters must scan:

```text
layout                                    day   customer    channel
no order (shuffled)                     98/98      98/98      98/98
CLUSTER BY (ordered_at)                  1/98      98/98      98/98
CLUSTER BY (customer_id)                98/98       1/98      98/98
CLUSTER BY (channel, ordered_at)         6/98      98/98      33/98
```

- Ordering by `ordered_at` makes date filters nearly free and does nothing for customers.
- Ordering by `customer_id` is the mirror image.
- `(channel, ordered_at)` serves **both** channel filters (33/98, one-third of the data for one of three channels) and
  date filters (6/98: for each channel, the rows of one day sit together) reasonably well. Neither is as good as a
  dedicated layout.

So clustering is a choice about **which queries matter most**. Look at the filters in your heaviest, most frequent
queries (Level 2 · 03 shows how to find them) before choosing.

## Defining a clustering key

```sql
ALTER TABLE learn_db.raw.orders CLUSTER BY (TO_DATE(ordered_at), channel);

-- or at creation
CREATE TABLE learn_db.raw.events (event_ts TIMESTAMP_NTZ, event_type VARCHAR, user_id NUMBER, payload VARIANT)
  CLUSTER BY (TO_DATE(event_ts), event_type);

SHOW TABLES LIKE 'ORDERS' IN SCHEMA learn_db.raw;     -- cluster_by and automatic_clustering columns
SELECT SYSTEM$CLUSTERING_INFORMATION('learn_db.raw.orders');
```

Snowflake's documented guidance for choosing keys:

- **At most 3 or 4 columns or expressions.** More tends to cost more than it gains.
- **Order columns from lowest to highest cardinality.** A high-cardinality column first makes the later columns nearly
  useless, just as `customer_id` before `ordered_at` would scatter dates.
- **Avoid the extremes of cardinality.** A boolean barely prunes. A nanosecond timestamp or a unique ID is expensive to
  keep clustered. Use an **expression** that reduces cardinality: `TO_DATE(event_ts)` instead of `event_ts`, or
  `TRUNC(id, -6)` for numeric IDs.
- **Prefer columns used in selective filters**, then columns used in joins.

### The text-prefix trap

Snowflake's documentation adds a detail that's easy to miss: when clustering on a **text** column, the clustering
metadata tracks only **the first several bytes, typically 5 or 6**. Our SKUs look like `SKU-0417`, so their first 5
bytes are `SKU-0` for every row. To clustering, they're all the same value.

This model (Parquet locally, not Snowflake) shows what bounds truncated to a prefix would mean for skipping data on
a table ordered by `sku`, filtering for one SKU:

```text
full-length      min/max: scans  1 of 98 row groups for sku = 'SKU-0417'
6-byte prefix    min/max: scans 13 of 98 row groups for sku = 'SKU-0417'
5-byte prefix    min/max: scans 98 of 98 row groups for sku = 'SKU-0417'
```

With a 5-byte prefix, every group looks identical and nothing can be told apart. The docs' statement concerns the
**clustering** metadata (what Automatic Clustering and the depth calculation see). Whether query-time pruning uses
longer bounds isn't specified there, so treat the model as an illustration of the clustering side. The practical rule
is the same either way: **don't cluster on text whose distinguishing characters come after a common prefix.** Cluster
on an expression that puts the distinguishing part first, such as `TRY_TO_NUMBER(SUBSTR(sku, 5))`, or on a numeric
surrogate key.

## Automatic Clustering: what it does and costs

Once a table has a key, **Automatic Clustering** monitors it and, when enough new or changed micro-partitions overlap,
rewrites them in key order in the background. It's a **serverless** feature: it doesn't use your warehouses, it's
billed in credits at its own rate (in the Service Consumption Table), and it also creates new micro-partitions, which
means more Time Travel and Fail-safe storage for the retired ones.

```sql
ALTER TABLE learn_db.raw.orders SUSPEND RECLUSTER;    -- pause background work
ALTER TABLE learn_db.raw.orders RESUME RECLUSTER;

SELECT start_time, end_time, credits_used, num_bytes_reclustered, num_rows_reclustered
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE table_name = 'ORDERS'
ORDER BY start_time DESC;
```

Snowflake's own guidance is that clustering keys "are not intended for all tables". It's optimal when you need the
fastest response regardless of cost, or when the query savings outweigh the clustering credits. In practice the good
candidates are:

- **large** tables (many thousands of micro-partitions, typically terabytes, since small tables are scanned quickly anyway);
- queried **frequently** with **selective** filters on the key;
- that **change** in ways that degrade natural order (late-arriving data, updates and merges scattered across history).

A table that's appended in time order and queried by time is **already** well clustered by time. Adding a time
clustering key mostly adds cost.

## Cheaper alternatives

**Sort on load.** If the data arrives in batches, order each batch by the columns you filter on. Micro-partitions are
formed in write order (lesson 01), so a sorted insert produces clustered micro-partitions at no extra cost:

```sql
INSERT INTO learn_db.raw.orders
SELECT * FROM learn_db.raw.orders_staging
ORDER BY TO_DATE(ordered_at), channel;
```

For a one-time fix of a badly ordered table, rewrite it sorted with `CREATE OR REPLACE TABLE … AS SELECT … ORDER BY …`
(watch the Time Travel storage from lesson 08).

**Search optimization for point lookups.** The `customer = 4242` filter can't be served by a time-ordered layout. The
**search optimization service** (Enterprise Edition) builds a separate persistent access structure for selective
equality and some other predicates (point lookups, `IN` lists, substring and semi-structured searches), so such
queries skip micro-partitions without changing the table's order:

```sql
ALTER TABLE learn_db.raw.orders ADD SEARCH OPTIMIZATION ON EQUALITY(customer_id, sku);
DESCRIBE SEARCH OPTIMIZATION ON learn_db.raw.orders;
```

It's also serverless and billed for building, maintaining and storing the structure. A common design for large fact
tables is a time clustering key (or natural time order) for range scans, plus search optimization on the ID columns
used in lookups.

**Materialized views** (Enterprise) can keep a differently clustered copy of a subset for a specific query pattern, at
the cost of storage and maintenance.

## How It Actually Works

Automatic Clustering doesn't sort the entire table. It continuously evaluates the clustering state from micro-partition
metadata (overlap and depth for the key), selects sets of micro-partitions whose key ranges overlap heavily, and
rewrites just those, merging and sorting their rows into new micro-partitions with narrow ranges. New data is clustered
incrementally the same way. It aims for "good enough" clustering that improves pruning, not a perfect global sort,
because a perfect sort of a constantly changing large table would be far too expensive. When micro-partitions reach a
state where their ranges no longer overlap, they're in what the docs call a constant state and aren't reclustered
again.

Every rewrite is a normal DML-style change: new micro-partitions, retired old ones, Time Travel and Fail-safe on the
retired ones. That's why clustering a table that receives scattered updates (lesson 09's model) can cost both
reclustering credits and storage. The scattered changes keep breaking the order, and the service keeps restoring it.

## Common mistakes

- **Clustering small tables.** They scan quickly anyway, so you pay for nothing.
- **Clustering on a high-cardinality raw column** (timestamps, IDs). Use an expression that reduces cardinality.
- **High cardinality first** in a multi-column key.
- **Text keys with a long common prefix.** Clustering tracks only the first 5–6 bytes.
- **Adding a time key to an append-only, time-ordered table.** It's already clustered.
- **Never checking `AUTOMATIC_CLUSTERING_HISTORY`.** Clustering credits can quietly exceed what queries save.
- **Using clustering for point lookups.** Search optimization is the tool for that.

## Exercise

1. Add `CLUSTER BY (customer_id, ordered_at)` to the layout experiment in this order and in the reverse order, and fill in
   the table for both. Which order serves the day filter better, and why?
2. In Snowflake, create a 50-million-row table from `SNOWFLAKE_SAMPLE_DATA.TPCH_SF10.LINEITEM` in random order, record
   `SYSTEM$CLUSTERING_INFORMATION` on `l_shipdate`, add `CLUSTER BY (l_shipdate)`, and check again over the next hours
   together with `AUTOMATIC_CLUSTERING_HISTORY` credits.
3. Instead of a clustering key, rebuild the same table with `ORDER BY l_shipdate` and compare `partitions_scanned` for a
   one-week filter with the automatically clustered version. Which was cheaper overall?
4. Design the physical layout for a 20 TB `events` table queried mostly by `event_date` ranges, often filtered by
   `tenant_id` (2,000 tenants), and occasionally by `event_id` lookups. Which key, in which order, plus which other feature?
