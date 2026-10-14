---
description: "Migrating to Snowflake: inventory, SQL translation and the semantic traps it misses, moving data as Parquet, and reconciliation that finds export bugs."
---

# 09 · Migrating to Snowflake

Most Snowflake estates start as a migration from somewhere: Teradata, SQL Server, Oracle, Redshift, BigQuery, a Hadoop
cluster. The hard part is rarely moving bytes. It's proving that the new system gives the **same answers**. This lesson walks
through the phases, shows automated SQL translation producing code that runs but means something different, and builds the
reconciliation that catches what translation and export get wrong.

## The phases

1. **Inventory.** List objects (tables, views, procedures, jobs) and, more usefully, the **queries actually run**, from the old
   system's query logs. Unused objects don't need migrating, and the most-run queries are your test suite.
2. **Translate** DDL, views, procedures and ETL code into Snowflake SQL.
3. **Move data**: a one-off historical load, then the ongoing pipeline.
4. **Validate**: reconcile the same questions on both systems, for every table and the top queries.
5. **Run in parallel, then cut over** consumers (BI tools, applications, downstream exports) one at a time, with a rollback plan.

Snowflake's own tooling covers phase 2: **SnowConvert AI** converts SQL from, per the documentation, Teradata, Oracle, SQL Server,
Redshift, Azure Synapse, Sybase IQ, Spark SQL, Databricks SQL, BigQuery, PostgreSQL and others, and the **Snowpark Migration
Accelerator** handles Spark code. Neither was run for this course. Whatever converts the code, phases 4 and 5 are yours.

## Translation runs, but does it mean the same?

Open-source **sqlglot** (30.22, run locally) transpiles between dialects. Eight typical source queries, translated to Snowflake:

```python
import sqlglot
cases = [
    ("tsql", "SELECT TOP 10 customer_id, ISNULL(region, 'n/a') AS region, DATEADD(day, -7, GETDATE()) AS since FROM dbo.orders WITH (NOLOCK) ORDER BY amount DESC"),
    ("tsql", "SELECT customer_id, total_items / order_count AS avg_items FROM dbo.customer_stats"),
    ("tsql", "SELECT first_name + ' ' + last_name AS full_name FROM dbo.customers"),
    ("redshift", "SELECT order_id, NVL(amount, 0) AS amount, GETDATE() AS loaded_at FROM sales.orders WHERE order_date >= DATEADD(month, -1, CURRENT_DATE)"),
    ("redshift", "SELECT LISTAGG(sku, ',') WITHIN GROUP (ORDER BY sku) FROM sales.order_lines GROUP BY order_id"),
    ("bigquery", "SELECT user_id, ARRAY_AGG(event ORDER BY ts LIMIT 5) AS last_events FROM `proj.web.events` GROUP BY user_id"),
    ("bigquery", "SELECT * FROM `proj.web.events` WHERE DATE(ts) = CURRENT_DATE() AND SAFE_CAST(value AS INT64) > 10"),
    ("bigquery", "SELECT EXTRACT(DAYOFWEEK FROM ts) AS dow, COUNT(*) FROM `proj.web.events` GROUP BY dow"),
]
for dialect, sql in cases:
    print(f"-- {dialect}: {sql}")
    try:
        print(sqlglot.transpile(sql, read=dialect, write="snowflake", pretty=False)[0])
    except Exception as e:
        print("!! ", type(e).__name__, e)
    print()
```

```text
-- tsql: SELECT TOP 10 customer_id, ISNULL(region, 'n/a') AS region, DATEADD(day, -7, GETDATE()) AS since FROM dbo.orders WITH (NOLOCK) ORDER BY amount DESC
SELECT customer_id, COALESCE(region, 'n/a') AS region, DATEADD(DAY, -7, CURRENT_TIMESTAMP()) AS since FROM dbo.orders ORDER BY amount DESC NULLS LAST LIMIT 10

-- tsql: SELECT customer_id, total_items / order_count AS avg_items FROM dbo.customer_stats
SELECT customer_id, total_items / order_count AS avg_items FROM dbo.customer_stats

-- tsql: SELECT first_name + ' ' + last_name AS full_name FROM dbo.customers
SELECT first_name + ' ' + last_name AS full_name FROM dbo.customers

-- redshift: SELECT order_id, NVL(amount, 0) AS amount, GETDATE() AS loaded_at FROM sales.orders WHERE order_date >= DATEADD(month, -1, CURRENT_DATE)
SELECT order_id, COALESCE(amount, 0) AS amount, CURRENT_TIMESTAMP() AS loaded_at FROM sales.orders WHERE order_date >= DATEADD(MONTH, -1, CAST(CURRENT_DATE AS TIMESTAMP))

-- redshift: SELECT LISTAGG(sku, ',') WITHIN GROUP (ORDER BY sku) FROM sales.order_lines GROUP BY order_id
SELECT LISTAGG(sku, ',') WITHIN GROUP (ORDER BY sku) FROM sales.order_lines GROUP BY order_id

-- bigquery: SELECT user_id, ARRAY_AGG(event ORDER BY ts LIMIT 5) AS last_events FROM `proj.web.events` GROUP BY user_id
SELECT user_id, ARRAY_AGG(event ORDER BY ts NULLS FIRST LIMIT 5) AS last_events FROM "proj"."web"."events" GROUP BY user_id

-- bigquery: SELECT * FROM `proj.web.events` WHERE DATE(ts) = CURRENT_DATE() AND SAFE_CAST(value AS INT64) > 10
SELECT * FROM "proj"."web"."events" WHERE DATE(ts) = CURRENT_DATE AND CAST(value AS BIGINT) > 10

-- bigquery: SELECT EXTRACT(DAYOFWEEK FROM ts) AS dow, COUNT(*) FROM `proj.web.events` GROUP BY dow
SELECT DATE_PART(DAYOFWEEK, ts) AS dow, COUNT(*) FROM "proj"."web"."events" GROUP BY dow
```

Every output parses, and several are subtly wrong. Each problem below is confirmed against Snowflake's documentation:

| Query | What changed silently |
|---|---|
| T-SQL `total_items / order_count` | In SQL Server, integer ÷ integer is integer division (7 / 2 = 3). In Snowflake, division of integers returns a decimal (3.5). Averages change. |
| T-SQL `first_name + ' ' + last_name` | `+` concatenates strings in SQL Server. In Snowflake, `+` is arithmetic, so this fails at run time. Use `\|\|` or `CONCAT`. |
| T-SQL `TOP 10 … ORDER BY amount DESC` | Handled correctly: the translator added `NULLS LAST`, because the two systems order NULLs differently. Good translators know these rules. Your hand-written code may not. |
| BigQuery `SAFE_CAST(value AS INT64)` | Became `CAST`, which **errors** on bad values instead of returning NULL. The Snowflake equivalent is `TRY_CAST`. |
| BigQuery `` `proj.web.events` `` | Became `"proj"."web"."events"`: quoted, therefore **case-sensitive lowercase**. Snowflake stores unquoted names in upper case, so this won't find a table created as `events`. |
| BigQuery `EXTRACT(DAYOFWEEK …)` | BigQuery numbers days 1 (Sunday) to 7. Snowflake's `DAYOFWEEK` with the default `WEEK_START = 0` returns 0 (Sunday) to 6. Every value is off by one. |
| BigQuery `ARRAY_AGG(… LIMIT 5)` | Snowflake's `ARRAY_AGG` syntax has no `LIMIT`. This fails to compile in Snowflake, so at least it fails loudly. |
| Redshift `NVL`, `GETDATE()`, `LISTAGG` | Translated correctly. |

The pattern: translation tools are good at **syntax**, but they can't know your column types or your intent, so **semantic**
differences slip through code that compiles. The only defence is running the translated query on both systems and comparing the
results, which is why the inventory's top queries become the test suite.

## Moving the data

The standard path for historical data is: export from the old system to **Parquet** in cloud storage, then load with `COPY`
(Level 1 · 05):

```sql
CREATE STAGE migration.raw.legacy_export
  URL = 's3://my-company-migration/legacy/'
  STORAGE_INTEGRATION = s3_migration_int
  FILE_FORMAT = (TYPE = PARQUET);

COPY INTO sales.raw.orders
  FROM @migration.raw.legacy_export/orders/
  MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE
  ON_ERROR = ABORT_STATEMENT;
```

Parquet carries types, which is both its strength and the risk. The export tool decides how decimals and timestamps are written,
and it doesn't always decide well.

## Reconciliation finds what exports get wrong

A local experiment (DuckDB 1.5.6, not Snowflake) on the course's 200,000 orders. The "legacy" table has `DECIMAL(10,2)` prices and
timestamps without a time zone, recorded in Berlin local time. Three export jobs are reconciled per day against the source:

```python
# DuckDB 1.5.6 locally, not Snowflake. The "legacy warehouse" is the course's orders table: DECIMAL(10,2) prices and
# TIMESTAMP (no time zone) order times recorded in Europe/Berlin local time. Three export jobs write Parquet for loading:
#   faithful : types unchanged
#   double   : prices written as DOUBLE (a common export-tool default)
#   utc      : timestamps reinterpreted as Berlin time and converted to UTC (a common "normalise to UTC" step)
# Reconciliation compares source and export per day: row count, revenue rounded to cents, and exact revenue.
import duckdb, sys
con = duckdb.connect()
con.execute(f"attach '{sys.argv[1]}' as legacy (read_only)")
con.execute("set TimeZone = 'Europe/Berlin'")
exports = {
    "faithful": "select * from legacy.orders",
    "double":   "select * replace (cast(unit_price as double) as unit_price) from legacy.orders",
    "utc":      "select * replace (timezone('UTC', ordered_at::timestamptz) as ordered_at) from legacy.orders",
}
for name, q in exports.items():
    con.execute(f"copy ({q}) to 'export_{name}.parquet' (format parquet)")

print(f"{'export':<10}{'days':>6}{'count differs':>15}{'cents differ':>14}{'exact differs':>15}")
for name in exports:
    days, n, cents, exact = con.execute(f"""
        with src as (select cast(ordered_at as date) d, count(*) n, sum(quantity * unit_price) r
                     from legacy.orders group by 1),
             tgt as (select cast(ordered_at as date) d, count(*) n, sum(quantity * unit_price) r
                     from 'export_{name}.parquet' group by 1)
        select count(*),
               count_if(src.n is distinct from tgt.n),
               count_if(round(src.r, 2) is distinct from round(tgt.r, 2)),
               count_if(src.r::double is distinct from tgt.r::double)
        from src full join tgt using (d)""").fetchone()
    print(f"{name:<10}{days:>6}{n:>15}{cents:>14}{exact:>15}")
```

```text
export      days  count differs  cents differ  exact differs
faithful     270              0             0              0
double       270              0             0            247
utc          271            263           271            271
```

- **faithful**: identical on every measure. That's the baseline a correct pipeline produces.
- **double**: counts and revenue rounded to cents match every day, but exact sums differ on 247 of 270 days. Floating-point
  prices accumulate tiny errors. Here they vanished at cent precision, but they won't always (large sums, ratios, `=` comparisons
  in joins). Keep money in `NUMBER`, and make the reconciliation compare **exactly** so you see the drift and decide on purpose.
- **utc**: the "normalise to UTC" step moved every order by one or two hours, so orders near midnight moved to a different day.
  Row counts differ on 263 days, and a day appears that didn't exist. Nothing failed, and the total row count is still 200,000. Only
  a per-day reconciliation shows it.

The Snowflake side of the same checks:

```sql
-- per-day profile, to compare with the same query on the legacy system
SELECT CAST(ordered_at AS DATE) AS d, COUNT(*) AS n, SUM(quantity * unit_price) AS revenue
FROM sales.raw.orders
GROUP BY d
ORDER BY d;

-- a whole-table fingerprint, for tables copied between Snowflake accounts or schemas
SELECT HASH_AGG(*) FROM sales.raw.orders;
SELECT HASH_AGG(*) FROM sales_migrated.raw.orders;
```

`HASH_AGG` is fast and order-independent, but it can only compare **Snowflake to Snowflake**, since the legacy system computes
different hashes. Across systems, compare profiles: counts, sums, min and max per key period and per important column, and
distinct counts for keys.

## Cutting over

Run both systems in parallel long enough to cover your business cycles (month end at least), with the daily reconciliation running
automatically and failing loudly. Move consumers one by one, starting with the least critical, and keep the old system read-only
until the last consumer has moved and a full cycle has passed. Then decommission it, and write down which of the differences you
accepted on purpose. Someone will ask in a year.

## How It Actually Works

A SQL translator parses source code into a syntax tree, rewrites nodes whose syntax differs (`TOP` → `LIMIT`, `ISNULL` →
`COALESCE`) using rules for each dialect pair, and prints the tree in the target dialect. It sees **text**, not data. It usually
doesn't know column types (is `a / b` integer division?), session settings (`WEEK_START`) or which objects exist, and that's exactly
where the semantic traps above live. Reconciliation works one level down: it compares **results** computed independently on each
system, so it catches translation errors, export errors and loading errors alike, whatever their cause. That's why it's the part
of a migration you can't skip or automate away.

## Common mistakes

- **Migrating everything.** Use query logs to find what's actually used.
- **Trusting code that compiles.** Integer division, string `+`, `CAST` vs `TRY_CAST`, week numbering and NULL ordering all compile.
- **Exporting money as floating point** and timestamps through an unreviewed time-zone conversion.
- **Reconciling only total row counts.** Compare per key period, and compare values exactly.
- **Quoted lower-case identifiers** from translated code, creating a mixture of case-sensitive and normal names.
- **Cutting over all consumers at once** with no parallel run.

## Exercise

1. Pick five queries your team runs often on another database. Translate them with sqlglot (`sqlglot.transpile(sql, read=…,
   write="snowflake")`) and review each against the table above.
2. Run `reconcile.py` on your machine, then add a fourth export that writes `quantity` as a string. What does the reconciliation
   show?
3. Write the per-day profile query for the legacy system you know best, and a script that diffs its output against Snowflake's.
4. In a trial account, load the `export_utc.parquet` file from the experiment and find the shifted orders with SQL alone.
