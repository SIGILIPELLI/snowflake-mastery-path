---
description: "Level 2 project: a stream, a deduplicating MERGE task and an incremental dynamic table, with the MERGE logic tested locally against full rebuilds."
---

# 10 · Project — An Incremental ELT Pipeline

This project builds the pipeline shape that most Snowflake platforms run in some form:

```text
files ──COPY/Snowpipe──▶ raw.orders ──append-only stream──▶ task: MERGE (dedup, ignore stale)
                                                                    │
                                                                    ▼
                                                     analytics.orders_clean ──dynamic table──▶ daily_revenue
```

Raw data is appended exactly as it arrives, including duplicates from at-least-once delivery and later updates to earlier
orders (refunds). A task consumes only the new rows through a stream, deduplicates them, and upserts them into a
current-state table, ignoring updates older than what's already there. A dynamic table keeps a daily aggregate
incrementally up to date. Health and correctness checks are part of the deliverable.

!!! note "What was tested, and how"
    The Snowflake script below was syntax-checked with `sqlglot` (10 statements parsed fully; `CREATE STREAM`,
    `CREATE TASK` and `ALTER TASK` were checked against the documentation). The heart of the pipeline, the **MERGE
    statement**, was tested for real. It was translated to DuckDB with `sqlglot` and run against seven batches of
    generated changes, with a batch table standing in for the stream, and after every batch the result was compared with
    a full rebuild from all raw rows. Streams, tasks and dynamic tables themselves weren't run.

## The pipeline

`pipeline.sql`:

```sql
-- Level 2 project: an incremental ELT pipeline (raw -> clean -> aggregates), fully inside Snowflake.

USE ROLE SYSADMIN;
CREATE WAREHOUSE IF NOT EXISTS pipeline_wh
  WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE INITIALLY_SUSPENDED = TRUE
  STATEMENT_TIMEOUT_IN_SECONDS = 1800;

CREATE DATABASE IF NOT EXISTS shop;
CREATE SCHEMA IF NOT EXISTS shop.raw;
CREATE SCHEMA IF NOT EXISTS shop.analytics;

-- 1. Raw landing table: append-only, loaded by Snowpipe or a scheduled COPY (lesson 05)
CREATE TABLE IF NOT EXISTS shop.raw.orders (
    order_id NUMBER(38,0), customer_id NUMBER(38,0), ordered_at TIMESTAMP_NTZ, channel VARCHAR, sku VARCHAR,
    quantity NUMBER(10,0), unit_price NUMBER(10,2), status VARCHAR, updated_at TIMESTAMP_NTZ,
    file_name VARCHAR, loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()::TIMESTAMP_NTZ
) DATA_RETENTION_TIME_IN_DAYS = 1;

-- 2. An append-only stream: every row ever loaded, consumed once (lesson 06)
CREATE STREAM IF NOT EXISTS shop.raw.orders_changes ON TABLE shop.raw.orders APPEND_ONLY = TRUE;

-- 3. The clean, current-state table
CREATE TABLE IF NOT EXISTS shop.analytics.orders_clean (
    order_id NUMBER(38,0), customer_id NUMBER(38,0), ordered_at TIMESTAMP_NTZ, channel VARCHAR, sku VARCHAR,
    quantity NUMBER(10,0), unit_price NUMBER(10,2), status VARCHAR, updated_at TIMESTAMP_NTZ
) CLUSTER BY (TO_DATE(ordered_at));

-- 4. The merge task: deduplicate the batch, ignore stale updates, upsert (lessons 06-07)
CREATE OR REPLACE TASK shop.raw.merge_orders
  WAREHOUSE = pipeline_wh
  SCHEDULE = '5 MINUTE'
  OVERLAP_POLICY = NO_OVERLAP
  SUSPEND_TASK_AFTER_NUM_FAILURES = 3
  WHEN SYSTEM$STREAM_HAS_DATA('shop.raw.orders_changes')
AS
MERGE INTO shop.analytics.orders_clean AS t
USING (
    SELECT order_id, customer_id, ordered_at, channel, sku, quantity, unit_price, status, updated_at
    FROM shop.raw.orders_changes
    QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) = 1
) AS s
ON t.order_id = s.order_id
WHEN MATCHED AND s.updated_at > t.updated_at THEN UPDATE SET
    customer_id = s.customer_id, ordered_at = s.ordered_at, channel = s.channel, sku = s.sku,
    quantity = s.quantity, unit_price = s.unit_price, status = s.status, updated_at = s.updated_at
WHEN NOT MATCHED THEN INSERT (order_id, customer_id, ordered_at, channel, sku, quantity, unit_price, status, updated_at)
    VALUES (s.order_id, s.customer_id, s.ordered_at, s.channel, s.sku, s.quantity, s.unit_price, s.status, s.updated_at);

ALTER TASK shop.raw.merge_orders RESUME;

-- 5. Aggregates as dynamic tables (lesson 08)
CREATE OR REPLACE DYNAMIC TABLE shop.analytics.daily_revenue
  TARGET_LAG = '30 minutes'
  WAREHOUSE = pipeline_wh
  REFRESH_MODE = INCREMENTAL
AS
SELECT TO_DATE(ordered_at) AS order_date, channel,
       COUNT(*) AS orders,
       SUM(IFF(status = 'paid', quantity * unit_price, 0))     AS paid_revenue,
       SUM(IFF(status = 'refunded', quantity * unit_price, 0)) AS refunded_amount
FROM shop.analytics.orders_clean
GROUP BY TO_DATE(ordered_at), channel;

-- 6. Pipeline health: one query to run (or alert on) every hour
SELECT 'merge task' AS component, MAX(IFF(state = 'SUCCEEDED', completed_time, NULL)) AS last_success,
       COUNT_IF(state = 'FAILED') AS failures_24h
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
       SCHEDULED_TIME_RANGE_START => DATEADD('hour', -24, CURRENT_TIMESTAMP()), TASK_NAME => 'MERGE_ORDERS'))
UNION ALL
SELECT 'daily_revenue refresh', MAX(IFF(state = 'SUCCEEDED', refresh_end_time, NULL)),
       COUNT_IF(state = 'FAILED')
FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLE_REFRESH_HISTORY(NAME => 'SHOP.ANALYTICS.DAILY_REVENUE'));

-- 7. Correctness check: the incremental table must equal a rebuild from raw
SELECT COUNT(*) AS mismatches FROM (
    SELECT order_id, status FROM shop.analytics.orders_clean
    EXCEPT
    SELECT order_id, status FROM shop.raw.orders
    QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) = 1
);
```

Choices worth noticing:

- **`APPEND_ONLY = TRUE`** on the stream. The raw table is only ever appended to, so an append-only stream is the right
  (and cheaper) type (lesson 06).
- **`QUALIFY ROW_NUMBER() … = 1`** in the `USING` subquery: one row per `order_id` per batch, the latest by `updated_at`.
- **`WHEN MATCHED AND s.updated_at > t.updated_at`**: a late-arriving, older update can never overwrite newer data.
- **`WHEN SYSTEM$STREAM_HAS_DATA`** so the warehouse stays asleep when nothing arrived, plus `OVERLAP_POLICY = NO_OVERLAP`
  and `SUSPEND_TASK_AFTER_NUM_FAILURES` (lesson 07).
- **`REFRESH_MODE = INCREMENTAL`** on the dynamic table, so if the query ever stops supporting incremental refresh, creation
  fails instead of silently switching to full refreshes (lesson 08).
- **Short retention on the raw table** (it's reloadable from files), and a **clustering key** on the clean table, which
  receives scattered updates (lesson 02).

## Testing the MERGE logic

The `MERGE`, as it appears in the task (`merge.sql`):

```sql
MERGE INTO analytics.orders_clean AS t
USING (
    SELECT order_id, customer_id, ordered_at, channel, sku, quantity, unit_price, status, updated_at
    FROM raw.orders_changes
    QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) = 1
) AS s
ON t.order_id = s.order_id
WHEN MATCHED AND s.updated_at > t.updated_at THEN UPDATE SET
    customer_id = s.customer_id, ordered_at = s.ordered_at, channel = s.channel, sku = s.sku,
    quantity = s.quantity, unit_price = s.unit_price, status = s.status, updated_at = s.updated_at
WHEN NOT MATCHED THEN INSERT (order_id, customer_id, ordered_at, channel, sku, quantity, unit_price, status, updated_at)
    VALUES (s.order_id, s.customer_id, s.ordered_at, s.channel, s.sku, s.quantity, s.unit_price, s.status, s.updated_at)
```

The test harness (`test_pipeline.py`) generates batches with new orders, status changes to earlier orders, and exact
duplicates, applies them incrementally, and compares the result with a rebuild after every batch:

```python
"""Test the pipeline's MERGE logic locally: DuckDB stands in for Snowflake, and a batch table stands in for the
append-only stream (rows loaded since the last successful merge). After every batch the incremental result must
equal a full rebuild from all raw rows."""
import random, duckdb, sqlglot
random.seed(7)
con = duckdb.connect()
con.sql("CREATE SCHEMA raw; CREATE SCHEMA analytics;")
cols = "order_id BIGINT, customer_id BIGINT, ordered_at TIMESTAMP, channel VARCHAR, sku VARCHAR, quantity INT, unit_price DECIMAL(10,2), status VARCHAR, updated_at TIMESTAMP"
con.sql(f"CREATE TABLE raw.orders ({cols}, batch_id INT)")
con.sql(f"CREATE TABLE analytics.orders_clean ({cols})")
merge_sql = sqlglot.transpile(open("merge.sql").read(), read="snowflake", write="duckdb")[0]

def make_batch(batch_id, next_id, existing):
    rows, t = [], f"2026-10-{batch_id + 1:02d} 12:00:00"
    for _ in range(500):                                   # new orders
        rows.append((next_id, random.randint(1, 500), t, random.choice(["web", "app", "store"]),
                     f"SKU-{random.randint(1, 50):04d}", random.randint(1, 3), round(random.uniform(5, 99), 2), "paid", t))
        next_id += 1
    for oid in random.sample(existing, min(60, len(existing))):   # status changes to earlier orders
        rows.append((oid, 1, t, "web", "SKU-0001", 1, 10.00, "refunded", t))
    rows += random.sample(rows, 25)                        # exact duplicates, as at-least-once delivery produces
    random.shuffle(rows)
    return rows, next_id

def full_rebuild():
    return con.sql("""SELECT order_id, status FROM raw.orders
                      QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) = 1
                      ORDER BY order_id""").fetchall()

next_id, existing = 1, []
for b in range(1, 6):
    rows, next_id = make_batch(b, next_id, existing)
    con.executemany("INSERT INTO raw.orders VALUES (?,?,?,?,?,?,?,?,?, " + str(b) + ")", rows)
    # 'stream' = rows not yet consumed (this batch)
    con.sql(f"CREATE OR REPLACE VIEW raw.orders_changes AS SELECT * EXCLUDE (batch_id) FROM raw.orders WHERE batch_id = {b}")
    con.sql(merge_sql)
    existing = [r[0] for r in con.sql("SELECT order_id FROM analytics.orders_clean").fetchall()]
    inc = con.sql("SELECT order_id, status FROM analytics.orders_clean ORDER BY order_id").fetchall()
    full = full_rebuild()
    dupes = con.sql("SELECT COUNT(*) - COUNT(DISTINCT order_id) FROM analytics.orders_clean").fetchone()[0]
    refunded = sum(1 for _, s in inc if s == "refunded")
    print(f"batch {b}: raw rows {con.sql('SELECT COUNT(*) FROM raw.orders').fetchone()[0]:>5}  clean rows {len(inc):>5}  "
          f"refunded {refunded:>4}  duplicate ids {dupes}  incremental == full rebuild: {inc == full}")

# batch 6: a late, OUT-OF-ORDER update (older updated_at) for an order that was refunded in batch 2
victim = con.sql("SELECT order_id FROM analytics.orders_clean WHERE status = 'refunded' ORDER BY order_id LIMIT 1").fetchone()[0]
con.execute("INSERT INTO raw.orders VALUES (?, 1, '2026-10-01 12:00:00', 'web', 'SKU-0001', 1, 10.00, 'paid', '2026-10-01 13:00:00', 6)", [victim])
con.sql("CREATE OR REPLACE VIEW raw.orders_changes AS SELECT * EXCLUDE (batch_id) FROM raw.orders WHERE batch_id = 6")
con.sql(merge_sql)
print(f"batch 6: late stale update for order {victim}: status now "
      f"{con.sql(f'SELECT status FROM analytics.orders_clean WHERE order_id = {victim}').fetchone()[0]!r} (guard kept the newer value); "
      f"incremental == full rebuild: {con.sql('SELECT order_id, status FROM analytics.orders_clean ORDER BY order_id').fetchall() == full_rebuild()}")

# what happens without the QUALIFY dedup in the USING clause?
no_dedup = sqlglot.transpile(open("merge.sql").read().replace(
    "QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) = 1", ""), read="snowflake", write="duckdb")[0]
con.sql("CREATE OR REPLACE VIEW raw.orders_changes AS SELECT * EXCLUDE (batch_id) FROM raw.orders WHERE batch_id = 5")
try:
    con.sql(no_dedup); print("merge without dedup: succeeded")
except Exception as e:
    print("merge without dedup:", type(e).__name__, str(e).splitlines()[0][:150])

# batch 7: a brand-new order delivered twice, merged WITHOUT the dedup
con.execute("INSERT INTO raw.orders VALUES (999999, 1, '2026-10-09 12:00:00', 'web', 'SKU-0001', 1, 10.00, 'paid', '2026-10-09 12:00:00', 7)")
con.execute("INSERT INTO raw.orders VALUES (999999, 1, '2026-10-09 12:00:00', 'web', 'SKU-0001', 1, 10.00, 'paid', '2026-10-09 12:00:00', 7)")
con.sql("CREATE OR REPLACE VIEW raw.orders_changes AS SELECT * EXCLUDE (batch_id) FROM raw.orders WHERE batch_id = 7")
con.sql(no_dedup)
print("batch 7 without dedup: rows for order 999999 in orders_clean =",
      con.sql("SELECT COUNT(*) FROM analytics.orders_clean WHERE order_id = 999999").fetchone()[0])
```

```text
batch 1: raw rows   525  clean rows   500  refunded    0  duplicate ids 0  incremental == full rebuild: True
batch 2: raw rows  1110  clean rows  1000  refunded   60  duplicate ids 0  incremental == full rebuild: True
batch 3: raw rows  1695  clean rows  1500  refunded  114  duplicate ids 0  incremental == full rebuild: True
batch 4: raw rows  2280  clean rows  2000  refunded  172  duplicate ids 0  incremental == full rebuild: True
batch 5: raw rows  2865  clean rows  2500  refunded  229  duplicate ids 0  incremental == full rebuild: True
batch 6: late stale update for order 1: status now 'refunded' (guard kept the newer value); incremental == full rebuild: True
merge without dedup: succeeded
batch 7 without dedup: rows for order 999999 in orders_clean = 2
```

What the run shows:

- **Batches 1–5:** after each batch, the incremental table equals a full rebuild (`True`), with no duplicate order IDs,
  even though every batch contained 25 duplicated rows and, from batch 2, 60 refunds of earlier orders.
- **Batch 6:** a stale update (older `updated_at`) for an already-refunded order arrived late. The guard kept
  `'refunded'`, and the table still equals the rebuild.
- **Without the `QUALIFY` dedup:** re-merging batch 5 didn't fail in DuckDB, and in **batch 7** a new order delivered
  twice was **inserted twice**. Snowflake's protection against ambiguous merges (`ERROR_ON_NONDETERMINISTIC_MERGE`, on by
  default) is documented for updating or deleting a target row that matches several source rows, so it wouldn't catch
  these duplicate **inserts** either. Deduplicating the source is what prevents them, in any engine.

## Running it in Snowflake

1. Run `pipeline.sql` (`snow sql -c trial -f pipeline.sql`).
2. Load the Level 1 order files into `shop.raw.orders` with an `updated_at` column (for a first load, `updated_at =
   ordered_at`). Within five minutes the task merges them, and within the target lag the dynamic table shows daily revenue.
3. Simulate refunds by appending rows for existing order IDs with `status = 'refunded'` and a later `updated_at`, plus
   some exact duplicates. Check that `orders_clean` reflects the refunds without duplicates.
4. Run the health query and the correctness check (`mismatches` must be 0).

## How It Actually Works

Each part does one job with clear guarantees. The **append-only stream** records rows added to `raw.orders` since the
task's last committed `MERGE`. If the task run fails, its transaction doesn't commit, the offset doesn't move, and the
next run sees the same rows plus newer ones. Because the `MERGE` is **idempotent** for a given set of rows (dedup plus
"only if newer"), replaying rows after a failure, or receiving the same row twice, can't change the result. That's what
makes the pipeline safe under the at-least-once behaviour of loaders, streams and retries. The **dynamic table**
reads `orders_clean` through change tracking and recomputes only the affected days and channels. The **health query** and
the **correctness check** turn the silent failure modes (a suspended task, a failing refresh, a logic error) into numbers
someone can alert on.

## Common mistakes

- **No dedup in the `MERGE` source.** Duplicate inserts, as batch 7 showed.
- **No "only if newer" guard.** Late events overwrite current state.
- **A standard stream on an append-only table**, which costs more for no benefit.
- **Forgetting to resume the task.**
- **No correctness check.** Incremental pipelines drift silently. Compare against a rebuild regularly.
- **A dynamic table on `AUTO`** that quietly falls back to full refresh on a large table.

## Exercise

1. Run the pipeline in a trial account with the Level 1 data, then append 1,000 refunds and 100 duplicates and confirm
   `mismatches = 0`.
2. Break the pipeline on purpose: change the task to reference a missing column. What does the health query show, what
   happens to the stream, and does the data catch up after you fix it?
3. Extend `test_pipeline.py` with deletions (an `is_deleted` flag) and add a `WHEN MATCHED AND s.is_deleted THEN DELETE`
   branch. Keep the full-rebuild comparison passing.
4. Replace the task and `MERGE` with a dynamic table that computes `orders_clean` directly from `raw.orders` using
   `QUALIFY`. Check which refresh mode Snowflake chooses, and compare the cost with the task-based version.
