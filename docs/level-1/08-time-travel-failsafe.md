---
description: "Snowflake Time Travel (AT, BEFORE, UNDROP), what Fail-safe is and isn't, retention by edition, and the storage a fully rewritten table can quietly hold."
---

# 08 · Time Travel, UNDROP & Fail-safe

Because Snowflake never modifies micro-partitions in place (lesson 01), the previous versions of your data still exist
for a while after every `UPDATE`, `DELETE`, `TRUNCATE` or `DROP`. **Time Travel** lets you query and restore those
versions yourself. **Fail-safe** keeps them for a further 7 days, recoverable only by Snowflake Support. Both are
excellent insurance, and both have a storage cost that's easy to multiply without noticing.

## Querying the past

```sql
-- the table as it was 5 minutes ago
SELECT COUNT(*) FROM learn_db.raw.orders AT(OFFSET => -60*5);

-- as of a specific time
SELECT COUNT(*) FROM learn_db.raw.orders AT(TIMESTAMP => '2026-10-13 09:00:00'::TIMESTAMP_LTZ);

-- just before a particular statement ran (use its query ID from the history)
SELECT COUNT(*) FROM learn_db.raw.orders BEFORE(STATEMENT => '01b2c3d4-0000-1234-0000-000000000abc');
```

- `OFFSET` is in seconds relative to now, so negative means "ago".
- `TIMESTAMP` is a point in time. Cast it explicitly (`TIMESTAMP_LTZ`), or the session's time zone and the default
  timestamp mapping can shift it by hours.
- `BEFORE(STATEMENT => …)` is the precise tool after an accident: "the table just before *that* `DELETE`".

A request beyond the retention period fails with an error rather than returning partial data.

## Recovering from mistakes

Here's a realistic accident: a `DELETE` with the wrong condition. First, find the statement:

```sql
DELETE FROM learn_db.raw.orders WHERE status = 'paid';     -- meant: status = 'cancelled'

SELECT query_id, query_text, start_time
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE query_text ILIKE 'DELETE FROM learn_db.raw.orders%'
ORDER BY start_time DESC
LIMIT 1;
```

Then put the deleted rows back, without touching anything changed since:

```sql
INSERT INTO learn_db.raw.orders
SELECT * FROM learn_db.raw.orders BEFORE(STATEMENT => '<query_id of the DELETE>')
WHERE status = 'paid';
```

Or restore the whole table to that point by cloning it, and swap it in:

```sql
CREATE TABLE learn_db.raw.orders_restored CLONE learn_db.raw.orders
  BEFORE(STATEMENT => '<query_id of the DELETE>');
ALTER TABLE learn_db.raw.orders SWAP WITH learn_db.raw.orders_restored;
```

`SWAP WITH` exchanges the two tables in one metadata operation, so readers never see a missing table.

## UNDROP

```sql
DROP TABLE learn_db.raw.orders;
UNDROP TABLE learn_db.raw.orders;

SHOW TABLES HISTORY LIKE 'ORDERS' IN SCHEMA learn_db.raw;   -- includes dropped versions and when
```

`UNDROP` works for tables, schemas and databases (and, per the docs, several other object types, such as dynamic tables
and Iceberg tables) within the retention period. It fails if an object with the same name already exists, so rename the
new one first. That's common when someone dropped a table and then recreated it empty. Dropping a schema or database and
`UNDROP`ping it restores everything inside.

## Retention

| | Standard Edition | Enterprise and higher |
|---|---|---|
| Default retention | 1 day | 1 day |
| Maximum, permanent objects | 1 day | **90 days** |
| Transient and temporary objects | 0 or 1 day | 0 or 1 day |
| Fail-safe after retention | 7 days (permanent only) | 7 days (permanent only) |

```sql
ALTER TABLE learn_db.raw.orders SET DATA_RETENTION_TIME_IN_DAYS = 30;   -- Enterprise+
ALTER DATABASE learn_db SET DATA_RETENTION_TIME_IN_DAYS = 7;             -- inherited by new children
SHOW TABLES LIKE 'ORDERS' IN SCHEMA learn_db.raw;                        -- retention_time column
```

Retention is set at the account, database, schema or table level, and child objects inherit unless they set their
own. An account administrator can also set `MIN_DATA_RETENTION_TIME_IN_DAYS`, which puts a floor under everything: the
effective retention is the larger of the two. That's a good guardrail against someone setting 0 on a production table.

## Fail-safe is not a backup you control

After Time Travel retention ends, data from **permanent** tables moves into Fail-safe for **7 days**. You can't query it,
clone it or restore it yourself. Only Snowflake Support can attempt recovery, as a best-effort service for disasters.
Transient and temporary tables have **no** Fail-safe. Plan your recovery on Time Travel (and replication, Level 4 · 07),
and think of Fail-safe as a last resort you hope never to need, but still pay storage for.

## The storage cost of history

Time Travel and Fail-safe store **retired micro-partitions**, and you pay for them like active data. A table that's
rewritten often retires a lot. This model computes what a 100 GB table holds if it's fully rewritten once a day (for
example `CREATE OR REPLACE TABLE … AS SELECT` or `INSERT OVERWRITE` in a nightly job). It's Python run locally,
following the documented retention rules, not a measurement:

```python
SIZE_GB = 100          # compressed size of the table
def held(retention_days, failsafe_days):
    # each daily rewrite retires one full copy; copies live retention + failsafe days after being retired
    return SIZE_GB * (1 + retention_days + failsafe_days)
```

```text
table kind                             retention fail-safe  GB held x active
permanent, Standard default                    1         7      900        9
permanent, 30-day retention                   30         7     3800       38
permanent, 90-day retention                   90         7     9800       98
transient, 1-day retention                     1         0      200        2
transient, 0-day retention                     0         0      100        1
```

A permanent table rewritten daily holds **9×** its size even at the default 1-day retention, because of Fail-safe. Set
90-day retention "to be safe" on it, and it holds **98×**. The same table as **transient** holds 2× or 1×. Real tables
usually change only part of their micro-partitions per day, so the multiplier is lower, but the shape is right. For
tables you can rebuild from their sources, such as staging tables, intermediate ETL tables and anything recreated
nightly, use transient tables and short retention, and keep long retention for the tables that are the system of
record.

You can see where storage goes:

```sql
SELECT table_catalog, table_schema, table_name,
       active_bytes / POWER(1024, 3)      AS active_gb,
       time_travel_bytes / POWER(1024, 3) AS time_travel_gb,
       failsafe_bytes / POWER(1024, 3)    AS failsafe_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE deleted IS NULL OR deleted = FALSE
ORDER BY time_travel_bytes + failsafe_bytes DESC
LIMIT 20;
```

(`ACCOUNT_USAGE` views lag real time, by minutes to a few hours depending on the view. Level 3 · 09 covers them.)

## How It Actually Works

Every committed change produces a new **table version**: a new list of micro-partition files, recorded in the metadata
with a timestamp. Files that a new version no longer references aren't deleted. They're marked as retired at that
time. A Time Travel query (`AT`, `BEFORE`) simply asks the metadata for the file list that was current at the requested
moment and scans those files, so querying the past costs the same as querying the present, plus any storage that was
kept. `UNDROP` restores the dropped object's metadata entry, and its files are still there.

When a retired file's age exceeds the retention period, it moves out of the Time Travel window into Fail-safe, where it's
still stored, but no longer reachable through SQL. After 7 more days it's permanently deleted. `TABLE_STORAGE_METRICS`
reports bytes in each of the three states. A clone (lesson 09) adds a twist: a retired file still referenced by another
table's current version isn't "history" for that table, so the accounting follows the file's owner.

## Common mistakes

- **Long retention on frequently rewritten tables.** It multiplies storage, as the model shows.
- **Permanent tables for staging and scratch data.** You pay 7 days of Fail-safe on every rewrite.
- **Relying on Fail-safe as a backup.** You can't access it yourself.
- **Time Travel with an ambiguous timestamp literal.** Cast to `TIMESTAMP_LTZ` or use `OFFSET`/`STATEMENT`.
- **`UNDROP` after recreating the table under the same name.** Rename the new one first.
- **Retention 0 on important tables** "to save money". Use `MIN_DATA_RETENTION_TIME_IN_DAYS` as a guardrail.

## Exercise

1. Delete all `paid` orders, then restore exactly those rows with `BEFORE(STATEMENT => …)`, and check that the row count
   matches the original.
2. Drop the `orders` table, create a new empty `orders`, then get the original back with `UNDROP` (rename the empty one
   first). Use `SHOW TABLES HISTORY` to see both.
3. Extend the storage model so that only 10% of the table's micro-partitions change per day. What does a permanent
   table with 30-day retention hold now?
4. Run the `TABLE_STORAGE_METRICS` query in a trial account after a few days of exercises and find the table with the
   largest Time Travel bytes. Should it be transient?
