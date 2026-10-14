---
description: "Snowflake dynamic tables: TARGET_LAG and DOWNSTREAM, refresh modes and the silent fallback to full refresh, refresh history, and when to use streams."
---

# 08 · Dynamic Tables

Streams and tasks (lessons 06–07) build incremental pipelines **imperatively**: you write the `MERGE`, schedule it, chain
it, and handle failures. A **dynamic table** does it **declaratively**: you write the `SELECT` that defines the result,
say how fresh it must be, and Snowflake works out the dependencies, schedules the refreshes, and (where it can) refreshes
only what changed. For many transformation pipelines that removes most of the code. The trade-offs are in knowing when
"only what changed" quietly turns into "everything, every time".

## A dynamic table

```sql
CREATE OR REPLACE DYNAMIC TABLE learn_db.analytics.orders_enriched
  TARGET_LAG = '15 minutes'
  WAREHOUSE = etl_wh
  REFRESH_MODE = AUTO
  INITIALIZE = ON_CREATE
AS
SELECT o.order_id, o.ordered_at, TO_DATE(o.ordered_at) AS order_date, o.channel, o.sku,
       o.quantity, o.unit_price, o.quantity * o.unit_price AS amount, o.status,
       c.v:country::VARCHAR AS country
FROM learn_db.raw.orders o
JOIN learn_db.raw.customers_json c ON c.v:customer_id::NUMBER = o.customer_id;
```

- **`TARGET_LAG`** is a freshness goal: the dynamic table's data should never be more than 15 minutes behind its sources.
  Snowflake schedules refreshes to meet it. The documented minimum is 60 seconds, and actual lag can exceed the target if
  refreshes take longer than the target allows.
- **`WAREHOUSE`** runs the refreshes, so its credits are the cost.
- **`REFRESH_MODE`**: `AUTO` (the default) picks incremental if the query supports it, otherwise full. `INCREMENTAL` and
  `FULL` force a mode. Newer modes, `ADAPTIVE` (incremental, reinitializing after large upstream changes) and
  `CUSTOM_INCREMENTAL`, are listed in the current reference.
- **`INITIALIZE`**: `ON_CREATE` fills the table during `CREATE`, and `ON_SCHEDULE` defers it to the first scheduled
  refresh.

## Pipelines of dynamic tables

Dynamic tables can read other dynamic tables. Intermediate ones can use `TARGET_LAG = DOWNSTREAM`, meaning "refresh only
when something downstream needs fresh data":

```sql
CREATE OR REPLACE DYNAMIC TABLE learn_db.analytics.orders_paid
  TARGET_LAG = DOWNSTREAM
  WAREHOUSE = etl_wh
AS
SELECT * FROM learn_db.analytics.orders_enriched WHERE status = 'paid';

CREATE OR REPLACE DYNAMIC TABLE learn_db.analytics.daily_revenue_by_country
  TARGET_LAG = '1 hour'
  WAREHOUSE = etl_wh
AS
SELECT order_date, country, COUNT(*) AS orders, SUM(amount) AS revenue
FROM learn_db.analytics.orders_paid
GROUP BY order_date, country;
```

Snowflake builds the dependency graph from the queries themselves, with no `AFTER` clauses and no scheduling code. If the
final table needs one-hour freshness, the upstream `DOWNSTREAM` tables refresh as needed to provide it, and all tables in
one refresh read a consistent snapshot of the sources. Snowsight shows the graph and each table's lag.

## The silent fallback to full refresh

**Incremental** refresh processes only the rows that changed since the last refresh. **Full** refresh recomputes the
entire result every time. With `REFRESH_MODE = AUTO`, if the query contains a construct that incremental refresh doesn't
support, Snowflake's documentation says it "selects a full refresh instead, which reprocesses all data on every refresh".
There's no error. The set of supported constructs keeps growing, and the current supported-queries page lists additions
from 2025 and 2026 such as `UNION`, Cortex AI functions in `SELECT`, `MIN_BY`/`MAX_BY`, more outer-join patterns, and
volatile scalar UDFs. So check the mode Snowflake actually chose for each table:

```sql
SHOW DYNAMIC TABLES IN SCHEMA learn_db.analytics;
-- columns refresh_mode (what it actually uses) and refresh_mode_reason (why it fell back)
```

For a small table the difference doesn't matter. For a large one it's the whole bill: a full refresh of a 1 TB result
every 15 minutes is 96 full recomputations a day. Two protective habits:

- On large tables, set `REFRESH_MODE = INCREMENTAL` explicitly. Then an unsupported construct makes `CREATE` **fail**
  instead of silently choosing full refresh.
- Keep the heavy, incremental-friendly logic (filters, joins, aggregations) in the large upstream dynamic tables, and
  put constructs that force full refresh into small downstream ones.

## Monitoring and control

```sql
SELECT name, state, refresh_action, refresh_trigger, data_timestamp,
       refresh_start_time, refresh_end_time,
       DATEDIFF('second', refresh_start_time, refresh_end_time) AS seconds,
       state_message
FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLE_REFRESH_HISTORY(NAME_PREFIX => 'LEARN_DB.ANALYTICS.'))
ORDER BY refresh_start_time DESC
LIMIT 50;

ALTER DYNAMIC TABLE learn_db.analytics.daily_revenue_by_country SUSPEND;
ALTER DYNAMIC TABLE learn_db.analytics.daily_revenue_by_country RESUME;
ALTER DYNAMIC TABLE learn_db.analytics.daily_revenue_by_country REFRESH;            -- refresh now
ALTER DYNAMIC TABLE learn_db.analytics.daily_revenue_by_country SET TARGET_LAG = '4 hours';
```

`refresh_action` shows whether each refresh was `INCREMENTAL`, `FULL`, `NO_DATA` (nothing changed, so little work), or a
reinitialization. Refreshes that keep taking longer than the target lag are the main operational signal. Widen the lag,
size up the warehouse, or simplify the query.

## Dynamic tables, streams and tasks, or a view?

| Need | Use |
|---|---|
| A transformation result that should stay within N minutes of its sources | **dynamic table** |
| Complex procedural logic, side effects, calls to external systems, notifications | **streams + tasks** (or procedures) |
| Exact control over when and how each step runs | **streams + tasks** |
| Always-current results, cheap to compute at query time | a **view** |
| An expensive aggregate on one table, transparently used by queries (Enterprise) | a **materialized view** |

Snowflake's documentation includes a decision guide and a migration guide from streams and tasks to dynamic tables, which
says something about the intended direction. For new transformation pipelines, start with dynamic tables and drop to
streams and tasks where you need imperative control.

## How It Actually Works

A dynamic table is a real table (micro-partitions, Time Travel, clustering) plus a stored query, a refresh mode and a
target lag. Snowflake tracks the **graph** of dynamic tables and their base tables, and a scheduler picks refresh times
for each table so that, given recent refresh durations, its data will be within the target lag. Refreshes of tables in
the same graph are coordinated to read the same **data timestamp**, which is how downstream tables stay consistent with
upstream ones.

Incremental refresh is built on the same change-tracking machinery as streams: Snowflake reads the changed rows of
each source since the previous refresh's data timestamp and applies a **differentiated** version of the query to them.
For a filter or projection, that's just running the query on the changed rows. For a join, it's changed-left with
all-right plus all-left with changed-right. For an aggregation, it's recomputing only the groups whose inputs changed.
When the query contains an operator Snowflake can't differentiate this way, the only correct option is to recompute
everything, which is the full refresh. The cost of an incremental refresh scales with the changes (and with how
scattered they are across micro-partitions), and the cost of a full refresh scales with the size of the inputs.

## Common mistakes

- **Assuming incremental.** Check `refresh_mode` and `refresh_mode_reason`, and pin `INCREMENTAL` on big tables.
- **A target lag tighter than the refresh takes**, which means constant refreshing and missed targets.
- **Very short lags everywhere.** Each refresh costs warehouse time. Use `DOWNSTREAM` for intermediates and set the lag
  from real freshness needs.
- **Putting side effects in the pipeline.** Dynamic tables are for deterministic transformations.
- **Using a shared interactive warehouse** for refreshes. Give pipelines their own warehouse.
- **Forgetting dynamic tables have Time Travel and Fail-safe storage**, which grows with full refreshes that rewrite
  everything (Level 1 · 08).

## Exercise

1. Build the three-table pipeline above on the course data, then check `SHOW DYNAMIC TABLES` for each table's actual
   refresh mode.
2. Add a construct to `orders_enriched` that your account's documentation lists as unsupported for incremental refresh
   (check the current supported-queries page), and observe what `AUTO` chooses and what `INCREMENTAL` does.
3. Insert 1,000 new orders and watch `DYNAMIC_TABLE_REFRESH_HISTORY` for the next refreshes of each table. Which ones
   refresh, and with which `refresh_action`?
4. Rebuild the same result with a stream on `raw.orders` and a task graph. Compare lines of code, and credits used over a day.
