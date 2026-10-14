---
description: "Snowflake streams for change data capture: net-change semantics, offsets that advance only on DML, staleness, append-only streams and the CHANGES clause."
---

# 06 · Streams: Change Data Capture

Incremental pipelines need to answer one question efficiently: **what changed since I last looked?** Reprocessing a whole
table every hour doesn't scale, and timestamp columns miss updates and deletes. A **stream** is a Snowflake object
that tracks changes to a table (or view) from a point called its **offset**, and returns exactly those changes when you
query it. Streams are the building block for incremental `MERGE`s, and with tasks (next lesson), for complete
pipelines.

## Creating and reading a stream

```sql
CREATE STREAM IF NOT EXISTS learn_db.raw.orders_changes ON TABLE learn_db.raw.orders;

SELECT *, METADATA$ACTION, METADATA$ISUPDATE, METADATA$ROW_ID
FROM learn_db.raw.orders_changes;
```

A stream has the source table's columns plus three metadata columns:

- `METADATA$ACTION`: `INSERT` or `DELETE`.
- `METADATA$ISUPDATE`: `TRUE` when the record is part of an `UPDATE`. An update appears as a **pair**: a `DELETE` of
  the old row and an `INSERT` of the new one, both with `ISUPDATE = TRUE`.
- `METADATA$ROW_ID`: a stable identifier for the row, used to match those pairs.

`SYSTEM$STREAM_HAS_DATA('learn_db.raw.orders_changes')` returns whether there's anything to process, cheaply. Tasks use it
to skip runs (lesson 07).

## Net changes, not a log

A stream returns the **difference between two table versions**: the version at its offset and the current one. It
isn't a log of every statement. From Snowflake's documented semantics, here's what a standard stream would return for
some sequences of changes between two consumptions:

| Changes since the offset | Stream contents |
|---|---|
| `INSERT` row 1 | one `INSERT` row, `ISUPDATE = FALSE` |
| `UPDATE` existing row 2 | `DELETE` (old values) + `INSERT` (new values), both `ISUPDATE = TRUE` |
| `INSERT` row 3, then `UPDATE` it | **one** `INSERT` with the final values, `ISUPDATE = FALSE` (it's a new row as far as the delta is concerned) |
| `INSERT` row 4, then `DELETE` it | **nothing** |
| `UPDATE` row 5 three times | one `DELETE`/`INSERT` pair: original values → latest values |
| `DELETE` existing row 6 | one `DELETE` row |

That's exactly what you need to bring a downstream table up to date, and exactly what you *don't* get if you want an
audit trail of every intermediate value. For that, write changes to a history table as they happen, or query history
with Time Travel.

## Consuming a stream: the offset advances only on DML

The rule that matters most: **querying a stream doesn't move its offset**. The offset advances only when the stream is
read by a **DML statement** (`INSERT`, `MERGE`, `UPDATE`, `DELETE`, `CREATE TABLE AS …`) that **commits**. Then the
changes are considered consumed.

```sql
MERGE INTO learn_db.analytics.orders_clean AS t
USING (
    SELECT * FROM learn_db.raw.orders_changes
    WHERE NOT (METADATA$ACTION = 'DELETE' AND METADATA$ISUPDATE)      -- drop the "before" half of updates
) AS s
ON t.order_id = s.order_id
WHEN MATCHED AND s.METADATA$ACTION = 'DELETE' THEN DELETE
WHEN MATCHED AND s.METADATA$ACTION = 'INSERT' THEN UPDATE SET
     t.customer_id = s.customer_id, t.ordered_at = s.ordered_at, t.channel = s.channel, t.sku = s.sku,
     t.quantity = s.quantity, t.unit_price = s.unit_price, t.status = s.status
WHEN NOT MATCHED AND s.METADATA$ACTION = 'INSERT' THEN INSERT
     (order_id, customer_id, ordered_at, channel, sku, quantity, unit_price, status)
     VALUES (s.order_id, s.customer_id, s.ordered_at, s.channel, s.sku, s.quantity, s.unit_price, s.status);
```

Filtering out the `DELETE` half of update pairs leaves, per row, either an `INSERT` (new or updated row: upsert it) or a
genuine `DELETE` (remove it). That's the standard CDC merge. If the raw table can receive duplicates for the same
`order_id` in one batch, deduplicate inside the `USING` subquery with `QUALIFY` (Level 1 · 06), or the merge will fail as
nondeterministic.

When **several statements** must see the same changes (for example, write to two target tables), wrap them in an
explicit transaction. Within it, the stream has **repeatable read** semantics, so every statement sees the same
records, and the offset advances once, at `COMMIT`:

```sql
BEGIN;
INSERT INTO learn_db.analytics.order_events SELECT order_id, METADATA$ACTION, CURRENT_TIMESTAMP() FROM learn_db.raw.orders_changes;
MERGE INTO learn_db.analytics.orders_clean AS t USING learn_db.raw.orders_changes AS s ON t.order_id = s.order_id
  WHEN MATCHED AND s.METADATA$ACTION = 'DELETE' AND NOT s.METADATA$ISUPDATE THEN DELETE;
COMMIT;
```

Without the transaction, the first statement would consume the changes and the second would see an empty stream.

## Stream types

| Type | Records | Use |
|---|---|---|
| Standard (default) | inserts, updates, deletes (net) | general CDC, merges |
| `APPEND_ONLY = TRUE` | inserts only | append-only sources such as event logs, raw loads, which is cheaper to compute |
| `INSERT_ONLY = TRUE` | inserts only | external tables and externally managed Iceberg tables |

```sql
CREATE STREAM learn_db.raw.orders_appends ON TABLE learn_db.raw.orders APPEND_ONLY = TRUE;
```

For a raw table that's only ever loaded by `COPY`/Snowpipe, an append-only stream is the natural choice. One table can
have **many streams**, each with its own offset, so several independent consumers can each process all changes at their
own pace.

## Staleness

A stream depends on the source table's history (Time Travel, lesson 08). If its offset falls outside the source's
retention period, it becomes **stale**: the unconsumed changes are gone, and it must be recreated. To give streams slack,
Snowflake temporarily extends a table's retention for an unconsumed stream, up to `MAX_DATA_EXTENSION_TIME_IN_DAYS`
(default 14 days). Snowflake's documentation gives examples of how often a stream must be consumed:

| `DATA_RETENTION_TIME_IN_DAYS` | `MAX_DATA_EXTENSION_TIME_IN_DAYS` | consume at least every |
|---|---|---|
| 14 | 0 | 14 days |
| 1 | 14 | 14 days |
| 0 | 90 | 90 days |

```sql
SHOW STREAMS IN SCHEMA learn_db.raw;      -- the stale and stale_after columns
DESCRIBE STREAM learn_db.raw.orders_changes;
```

A pipeline that pauses for a long holiday, or a dev stream nobody consumes, goes stale. Monitor `stale_after`.

## The CHANGES clause: read-only change queries

To *look* at changes between two points without creating a stream or moving any offset, query the table with
`CHANGES` (change tracking must be enabled on the table, which creating a stream does automatically):

```sql
ALTER TABLE learn_db.raw.orders SET CHANGE_TRACKING = TRUE;

SELECT *
FROM learn_db.raw.orders
  CHANGES (INFORMATION => DEFAULT)
  AT (OFFSET => -60*60);          -- net changes during the last hour
```

`INFORMATION => APPEND_ONLY` restricts it to inserts. This is good for audits, debugging and ad-hoc reconciliation, and
not a replacement for a stream in a pipeline (nothing tracks what you've processed).

## How It Actually Works

A stream stores **no data**. It stores an **offset**: a reference to a table version. When you query it, Snowflake
compares the table's micro-partitions at the offset version with those at the current version. Micro-partitions present
in both are unchanged and skipped. Those added or removed between the versions are read, and with the hidden row-level
change-tracking columns that enabling change tracking adds to the table, Snowflake computes the net inserts and deletes
per row. That's why the result is a net delta, why it costs more when changes are scattered across many
micro-partitions (lesson 09's model again), and why an append-only stream, which only has to read added
micro-partitions, is cheaper.

Consuming a stream in a committed DML transaction records the table version the statement read as the new offset.
Querying it alone doesn't commit anything, so nothing moves. And since the offset is just a version reference, the stream
depends on that version still being retained: when Time Travel no longer covers it, there's nothing to compare against,
and the stream is stale.

## Common mistakes

- **Expecting a `SELECT` to consume the stream.** Only committed DML moves the offset.
- **Two statements reading the same stream without a transaction.** The second one sees nothing.
- **Forgetting the `DELETE` half of update pairs** in a merge, which deletes rows that were only updated.
- **Expecting every intermediate change.** Streams return net changes between offsets.
- **Letting streams go stale** in rarely run pipelines or dev environments.
- **Standard streams on append-only tables.** Use `APPEND_ONLY = TRUE`.
- **Short retention plus `MAX_DATA_EXTENSION_TIME_IN_DAYS = 0`**, which leaves no slack for a delayed consumer.

## Exercise

1. Create a stream on `raw.orders`, then insert a row, update an existing row, insert-then-delete another, and update a
   fourth row twice. Query the stream and compare it with the table in this lesson.
2. Run the CDC `MERGE` and confirm the stream is empty afterwards. Then show that a plain `SELECT` from the stream doesn't
   empty it.
3. Write the two-statement transaction above without `BEGIN`/`COMMIT`, and show that the second statement processes
   nothing.
4. Create a stream on a table with `DATA_RETENTION_TIME_IN_DAYS = 0` and `MAX_DATA_EXTENSION_TIME_IN_DAYS = 0`, change
   the table, and inspect `stale_after` in `SHOW STREAMS`. What happens?
