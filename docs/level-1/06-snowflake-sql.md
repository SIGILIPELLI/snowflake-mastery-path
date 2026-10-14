---
description: "Snowflake SQL that saves work: QUALIFY, window functions, GROUP BY ALL, EXCLUDE, PIVOT, MERGE, ASOF JOIN, IFF and TRY_ functions, with real results."
---

# 06 · Snowflake SQL: Windows, QUALIFY & Extensions

Snowflake speaks standard ANSI SQL with a set of extensions that make analytical queries shorter and clearer. This
lesson covers the ones you'll use weekly, on the orders and customers loaded in lesson 05. It assumes you know joins and
`GROUP BY`. The [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/) covers those.

!!! note "Where the results come from"
    The statements below are Snowflake SQL. Their **results** were produced by running the same queries on the same
    generated data in **DuckDB 1.5.6** on a laptop, because DuckDB implements these extensions (`QUALIFY`, `GROUP BY ALL`,
    `EXCLUDE`, `PIVOT`, `ASOF JOIN`) with the same semantics. The numbers are what Snowflake should return for this
    dataset. Formatting, column types and the naming of `PIVOT` columns differ. Where the dialects differ, the lesson
    says so.

## QUALIFY: filter on a window function

The classic "latest row per key" query normally needs a subquery. `QUALIFY` filters on window functions directly, the
way `HAVING` filters on aggregates:

```sql
SELECT customer_id, order_id, ordered_at, unit_price * quantity AS amount
FROM learn_db.raw.orders
WHERE status = 'paid'
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY ordered_at DESC) = 1
ORDER BY customer_id
LIMIT 3;
```

```text
customer_id | order_id | ordered_at          | amount
          1 |    79422 | 2026-09-10 05:34:43 | 428.58
          2 |   171272 | 2026-08-19 08:41:43 | 634.02
          3 |   194457 | 2026-09-25 21:42:12 | 847.80
```

The evaluation order is `FROM` → `WHERE` → `GROUP BY` → `HAVING` → window functions → **`QUALIFY`** → `ORDER BY`
→ `LIMIT`. This is the standard way to **deduplicate** in Snowflake, which matters because primary keys aren't
enforced (lesson 03):

```sql
CREATE OR REPLACE TABLE learn_db.raw.orders_dedup AS
SELECT *
FROM learn_db.raw.orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY ordered_at DESC) = 1;
```

## Ranking, with ties

"Top 3 SKUs by units per channel":

```sql
SELECT channel, sku, SUM(quantity) AS units,
       RANK() OVER (PARTITION BY channel ORDER BY SUM(quantity) DESC) AS rnk
FROM learn_db.raw.orders
WHERE status = 'paid'
GROUP BY channel, sku
QUALIFY rnk <= 3
ORDER BY channel, rnk;
```

```text
channel | sku      | units | rnk
app     | SKU-0699 |   183 |   1
app     | SKU-0411 |   172 |   2
app     | SKU-0491 |   170 |   3
store   | SKU-0662 |   159 |   1
store   | SKU-0349 |   155 |   2
store   | SKU-0663 |   154 |   3
web     | SKU-0697 |   162 |   1
web     | SKU-0630 |   161 |   2
web     | SKU-0635 |   161 |   2
```

Look at `web`: two SKUs tied at 161 units, both are rank 2, and there's no rank 3. `RANK` leaves gaps,
`DENSE_RANK` doesn't, and `ROW_NUMBER` breaks ties arbitrarily (add tie-breaker columns to its `ORDER BY` if the
result must be stable). The window function can rank an **aggregate** (`SUM(quantity)`) because windows are evaluated
after `GROUP BY`.

## Running totals and period comparisons

```sql
SELECT DATE_TRUNC('month', ordered_at) AS month,
       SUM(unit_price * quantity) AS revenue,
       SUM(SUM(unit_price * quantity)) OVER (ORDER BY DATE_TRUNC('month', ordered_at)) AS running_revenue,
       LAG(SUM(unit_price * quantity)) OVER (ORDER BY DATE_TRUNC('month', ordered_at)) AS prev_month
FROM learn_db.raw.orders
WHERE status = 'paid'
GROUP BY 1
ORDER BY 1;
```

```text
month               | revenue    | running_revenue
2026-01-01 00:00:00 | 3733367.04 |      3733367.04
2026-02-01 00:00:00 | 3447838.33 |      7181205.37
2026-03-01 00:00:00 | 3731570.31 |     10912775.68
2026-04-01 00:00:00 | 3660444.72 |     14573220.40
```

(First four months shown. The DuckDB run didn't include the `LAG` column.) A window with `ORDER BY` and no frame
clause uses `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, i.e. a running total. Write the frame explicitly
(`ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` for a 3-month moving sum) whenever it matters.

## Shorter queries: GROUP BY ALL, EXCLUDE, RENAME, REPLACE

```sql
SELECT channel, status, COUNT(*) AS orders, SUM(unit_price * quantity) AS revenue
FROM learn_db.raw.orders
GROUP BY ALL
ORDER BY channel, status;
```

```text
channel | status    | orders | revenue
app     | cancelled |  13458 |  3641501.90
app     | paid      |  40061 | 10892006.16
app     | refunded  |  13269 |  3525570.92
store   | cancelled |  13339 |  3596972.63
store   | paid      |  39918 | 10765993.11
...
```

`GROUP BY ALL` groups by every non-aggregate expression in the `SELECT` list, so you can't forget one. `SELECT *` gets
modifiers to drop, rename or replace columns without listing all the others:

```sql
SELECT * EXCLUDE (sku, status) FROM learn_db.raw.orders ORDER BY order_id LIMIT 2;
SELECT * RENAME (ordered_at AS order_ts) FROM learn_db.raw.orders LIMIT 2;
SELECT * REPLACE (ROUND(unit_price, 0) AS unit_price) FROM learn_db.raw.orders LIMIT 2;
```

```text
order_id | customer_id | ordered_at          | channel | quantity | unit_price
       1 |        1186 | 2026-05-20 21:41:10 | store   |        1 |     165.36
       2 |         953 | 2026-01-15 03:00:49 | store   |        2 |      29.36
```

(Output of the `EXCLUDE` query.)

## PIVOT and UNPIVOT

```sql
SELECT *
FROM (SELECT channel, status, quantity FROM learn_db.raw.orders)
PIVOT (SUM(quantity) FOR status IN ('paid', 'refunded', 'cancelled')) AS p (channel, paid, refunded, cancelled)
ORDER BY channel;
```

```text
channel | paid  | refunded | cancelled
app     | 85856 |    28041 |     28870
store   | 85700 |    28890 |     28475
web     | 85537 |    28721 |     28288
```

In Snowflake, pivoted columns are named after the values **including their quotes** (`'paid'`), which is awkward to
reference. The column alias list after `AS p (...)` gives them clean names. `PIVOT … IN (ANY ORDER BY status)` pivots on
whatever values exist, without listing them. `UNPIVOT` reverses the operation.

## Conditional and safe expressions

```sql
SELECT order_id,
       IFF(quantity >= 3, 'bulk', 'single')                AS order_kind,
       ZEROIFNULL(quantity) * unit_price                   AS amount,
       TRY_TO_NUMBER('two')                                AS bad_number,     -- NULL, not an error
       TRY_TO_DATE('2026-02-30')                           AS bad_date,       -- NULL
       sku ILIKE 'sku-00%'                                 AS low_sku,
       channel LIKE ANY ('web', 'app')                     AS online,
       DIV0(unit_price, 0)                                 AS safe_div        -- 0 instead of division error
FROM learn_db.raw.orders
LIMIT 5;
```

The `TRY_` family (`TRY_TO_NUMBER`, `TRY_TO_DATE`, `TRY_CAST`, …) is the query-time version of the error handling from
lesson 05. `ILIKE` is case-insensitive `LIKE`, and `LIKE ANY`/`ILIKE ANY` match against several patterns.

## MERGE: upserts

Loading changed records into a table is a `MERGE`:

```sql
MERGE INTO learn_db.analytics.customers AS t
USING learn_db.raw.customer_updates AS s
  ON t.customer_id = s.customer_id
WHEN MATCHED AND s.is_deleted THEN DELETE
WHEN MATCHED THEN UPDATE SET t.name = s.name, t.country = s.country, t.updated_at = s.updated_at
WHEN NOT MATCHED AND NOT s.is_deleted THEN
  INSERT (customer_id, name, country, updated_at) VALUES (s.customer_id, s.name, s.country, s.updated_at);
```

If the source has **two rows for the same key**, the result is ambiguous. Snowflake's `ERROR_ON_NONDETERMINISTIC_MERGE`
parameter (on by default) makes such a merge fail instead of updating the row unpredictably. Deduplicate the source
with `QUALIFY` first.

## ASOF JOIN: "the latest value at that time"

Matching each order to the price in effect when it was placed is an inequality join that's painful in standard SQL.
`ASOF JOIN` does it directly:

```sql
SELECT o.order_id, o.ordered_at, o.sku, p.list_price
FROM learn_db.raw.orders o
ASOF JOIN learn_db.raw.price_changes p
  MATCH_CONDITION (o.ordered_at >= p.effective_at)
  ON o.sku = p.sku;
```

For each order, it picks the price row with the latest `effective_at` that is not after `ordered_at`, for the same
SKU.

## Sampling

```sql
SELECT * FROM learn_db.raw.orders SAMPLE (1);                -- about 1% of rows
SELECT * FROM learn_db.raw.orders SAMPLE SYSTEM (1) SEED (7); -- about 1% of micro-partitions, repeatable
```

Row sampling (`BERNOULLI`, the default) reads everything and keeps each row with the given probability. `SYSTEM` (block)
sampling picks whole micro-partitions, so it reads less but is clumpier.

## How It Actually Works

`QUALIFY`, `GROUP BY ALL` and the `SELECT *` modifiers are rewrites done by the compiler in the cloud services layer:
`QUALIFY` becomes a filter applied after the window-function step, `GROUP BY ALL` expands to the explicit expression
list, `EXCLUDE` expands to the remaining column list. They cost nothing at runtime.

Window functions are executed by **partitioning** rows by the `PARTITION BY` key across the warehouse's workers,
**sorting** each partition by the `ORDER BY` key, then streaming through the sorted rows computing running values. A
window over a very skewed key (one customer with half the orders) lands mostly on one worker, which is a common cause of
slow window queries (Level 2 · 03). `MERGE` is a join between target and source, followed by rewriting the
micro-partitions that contain matched rows (immutable files again, lesson 01), so it's much cheaper when matched rows
are concentrated in few micro-partitions (Level 2 · 01–02).

## Common mistakes

- **`ROW_NUMBER` without a tie-breaker** for deduplication, giving different "latest" rows on different runs.
- **Confusing `RANK`, `DENSE_RANK` and `ROW_NUMBER`** when there are ties, like the `web` channel above.
- **Relying on the default window frame.** Write `ROWS BETWEEN …` when you mean a fixed number of rows.
- **`MERGE` from a source with duplicate keys.** Deduplicate first.
- **Pivoted column names with quotes in them.** Alias them.
- **`TRY_` functions hiding data problems.** Count the `NULL`s they produce.

## Exercise

1. Write a query that returns, for each customer, their first and latest paid order and the number of days between them,
   in one pass with window functions.
2. Compute a 7-day moving average of daily paid revenue using an explicit `ROWS` frame. Then explain how the result would
   differ with `RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW` if some days had no orders.
3. Create a `customer_updates` table with an update, a new customer, a deletion, and **two** updates for one customer, and
   run the `MERGE` above. What happens? Fix it with `QUALIFY`.
4. Build a small `price_changes` table (three SKUs, a few price changes each) and use `ASOF JOIN` to compute revenue at
   list price. Check three orders by hand.
