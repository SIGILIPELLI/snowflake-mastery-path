---
description: "Level 1 project: a Snowflake sales database from scratch, CSV and JSON loads with validation, typed views, and eight answers checked against a key."
---

# 10 · Project — Load & Analyse a Sales Dataset

This project puts Level 1 together. Starting from an empty account, you'll create a warehouse, a database and schemas;
stage, validate and load the orders CSVs and the customers JSON from lesson 05; build typed, deduplicated views;
answer eight questions; and check your answers automatically against a **key**.

!!! note "How this project was verified without a Snowflake account"
    The SQL script was parsed statement by statement with `sqlglot`'s Snowflake dialect (23 of 23 statements parsed
    into full syntax trees). The **answer key** was computed by loading the same generated files into DuckDB 1.5.6 with
    the same error handling (the three planted bad rows rejected). The **checker script** was tested by running it
    against that DuckDB database through a small fake `snowflake.connector`. When you run the project in Snowflake, your
    numbers should match the key. If they don't, the checker tells you which question to investigate.

## The script

`project.sql`. Run it top to bottom in a Snowsight worksheet (or `snow sql -c trial -f project.sql`), after uploading
the files as described in step 2's comments:

```sql
-- Level 1 project: load and analyse the sales dataset.
-- Run as a role that can create databases and warehouses (SYSADMIN in a trial account).

-- 1. Compute and storage containers ------------------------------------------------------
CREATE WAREHOUSE IF NOT EXISTS learn_wh
  WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE INITIALLY_SUSPENDED = TRUE
  STATEMENT_TIMEOUT_IN_SECONDS = 900;
USE WAREHOUSE learn_wh;

CREATE DATABASE IF NOT EXISTS sales_lab;
CREATE SCHEMA IF NOT EXISTS sales_lab.raw;
CREATE SCHEMA IF NOT EXISTS sales_lab.analytics;

-- 2. Stage and file formats --------------------------------------------------------------
CREATE STAGE IF NOT EXISTS sales_lab.raw.landing;
CREATE FILE FORMAT IF NOT EXISTS sales_lab.raw.csv_gz
  TYPE = CSV COMPRESSION = GZIP SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF = ('', 'NULL') ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE;
CREATE FILE FORMAT IF NOT EXISTS sales_lab.raw.json_lines TYPE = JSON;

-- upload from your machine first:
--   snow stage copy "./orders_part*.csv.gz" @sales_lab.raw.landing/orders/ -c trial
--   snow stage copy ./customers.json @sales_lab.raw.landing/customers/ -c trial

-- 3. Raw tables ---------------------------------------------------------------------------
CREATE TRANSIENT TABLE IF NOT EXISTS sales_lab.raw.orders (
    order_id NUMBER(38,0), customer_id NUMBER(38,0), ordered_at TIMESTAMP_NTZ, channel VARCHAR,
    sku VARCHAR, quantity NUMBER(10,0), unit_price NUMBER(10,2), status VARCHAR,
    file_name VARCHAR, loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()::TIMESTAMP_NTZ
);
CREATE TRANSIENT TABLE IF NOT EXISTS sales_lab.raw.customers_json (
    v VARIANT, file_name VARCHAR, loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()::TIMESTAMP_NTZ
);

-- 4. Validate, then load ---------------------------------------------------------------------
COPY INTO sales_lab.raw.orders (order_id, customer_id, ordered_at, channel, sku, quantity, unit_price, status)
  FROM @sales_lab.raw.landing/orders/
  FILE_FORMAT = (FORMAT_NAME = 'sales_lab.raw.csv_gz')
  VALIDATION_MODE = RETURN_ALL_ERRORS;                     -- expect exactly 3 errors

COPY INTO sales_lab.raw.orders (order_id, customer_id, ordered_at, channel, sku, quantity, unit_price, status, file_name)
  FROM (SELECT $1, $2, $3, $4, $5, $6, $7, $8, METADATA$FILENAME FROM @sales_lab.raw.landing/orders/)
  FILE_FORMAT = (FORMAT_NAME = 'sales_lab.raw.csv_gz')
  ON_ERROR = 'CONTINUE';

COPY INTO sales_lab.raw.customers_json (v, file_name)
  FROM (SELECT $1, METADATA$FILENAME FROM @sales_lab.raw.landing/customers/)
  FILE_FORMAT = (FORMAT_NAME = 'sales_lab.raw.json_lines');

-- 5. Modelled layer ---------------------------------------------------------------------------
CREATE OR REPLACE VIEW sales_lab.analytics.customers AS
SELECT v:customer_id::NUMBER AS customer_id, v:name::VARCHAR AS name, v:country::VARCHAR AS country,
       TO_DATE(v:signup::VARCHAR) AS signup_date, v:contact.email::VARCHAR AS email,
       v:contact.phone::VARCHAR AS phone, v:tags AS tags
FROM sales_lab.raw.customers_json;

CREATE OR REPLACE VIEW sales_lab.analytics.orders AS
SELECT order_id, customer_id, ordered_at, channel, sku, quantity, unit_price,
       quantity * unit_price AS amount, status
FROM sales_lab.raw.orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY loaded_at DESC) = 1;

-- 6. Checks and analysis (compare with the answer key) ---------------------------------------
-- Q1 rows loaded
SELECT COUNT(*) AS rows_loaded FROM sales_lab.raw.orders;
-- Q2 monthly paid revenue
SELECT TO_CHAR(DATE_TRUNC('month', ordered_at), 'YYYY-MM') AS month, SUM(amount) AS revenue
FROM sales_lab.analytics.orders WHERE status = 'paid' GROUP BY 1 ORDER BY 1;
-- Q3 refund rate by channel
SELECT channel, ROUND(AVG(IFF(status = 'refunded', 1, 0)), 4) AS refund_rate
FROM sales_lab.analytics.orders GROUP BY 1 ORDER BY 1;
-- Q4 top 3 countries by paid revenue
SELECT c.country, SUM(o.amount) AS revenue
FROM sales_lab.analytics.orders o JOIN sales_lab.analytics.customers c USING (customer_id)
WHERE o.status = 'paid' GROUP BY 1 ORDER BY revenue DESC LIMIT 3;
-- Q5 customers without any orders
SELECT COUNT(*) AS customers_without_orders
FROM sales_lab.analytics.customers c
WHERE NOT EXISTS (SELECT 1 FROM sales_lab.analytics.orders o WHERE o.customer_id = c.customer_id);
-- Q6 customers with paid orders in 6 or more distinct months
SELECT COUNT(*) AS customers_active_6plus_months FROM (
  SELECT customer_id FROM sales_lab.analytics.orders WHERE status = 'paid'
  GROUP BY 1 HAVING COUNT(DISTINCT DATE_TRUNC('month', ordered_at)) >= 6);
-- Q7 duplicate order ids in the raw table
SELECT COUNT(*) AS duplicate_order_ids FROM (
  SELECT order_id FROM sales_lab.raw.orders GROUP BY 1 HAVING COUNT(*) > 1);
-- Q8 JSON nulls in contact.phone
SELECT COUNT_IF(IS_NULL_VALUE(v:contact.phone)) AS null_phone FROM sales_lab.raw.customers_json;
```

Design choices worth noticing:

- **Transient raw tables.** Raw data can be reloaded from the stage, so it doesn't need 7 days of Fail-safe (lesson 08).
- **Validate before loading.** `VALIDATION_MODE = RETURN_ALL_ERRORS` should report exactly the three planted errors. Then
  `ON_ERROR = 'CONTINUE'` loads everything else.
- **Lineage columns.** `METADATA$FILENAME` and `loaded_at` let you trace every row to a file and a load.
- **Views as the contract.** The `analytics` views cast VARIANT paths to real types and deduplicate with `QUALIFY`, so
  analysts never touch raw JSON or duplicate rows.
- **One small warehouse** with a 60-second auto-suspend and a 15-minute statement timeout (lesson 04).

## The answer key

`answer_key.json`, computed from the generated data:

```json
{
 "rows_loaded": 199997,
 "monthly_paid_revenue": [
  [
   "2026-01",
   3733367.04
  ],
  [
   "2026-02",
   3447838.33
  ],
  [
   "2026-03",
   3731570.31
  ],
  [
   "2026-04",
   3660444.72
  ],
  [
   "2026-05",
   3704260.2
  ],
  [
   "2026-06",
   3603539.23
  ],
  [
   "2026-07",
   3640109.25
  ],
  [
   "2026-08",
   3716939.64
  ],
  [
   "2026-09",
   3205541.51
  ]
 ],
 "refund_rate_by_channel": [
  [
   "app",
   0.1987
  ],
  [
   "store",
   0.2014
  ],
  [
   "web",
   0.2019
  ]
 ],
 "top3_countries_paid_revenue": [
  [
   "US",
   4407427.29
  ],
  [
   "FR",
   4261663.88
  ],
  [
   "ES",
   3994211.92
  ]
 ],
 "customers_without_orders": 0,
 "customers_active_6plus_months": 4991,
 "duplicate_order_ids": 0,
 "null_phone": 714
}
```

A few of these are worth predicting before you run anything:

- `rows_loaded` is **199,997**: 200,000 minus the three planted errors.
- September's revenue is lower than the other months because the generator only covers 270 days from 1 January, so
  September is partial. That's a data-coverage effect, not a business trend. Always check the date range before
  comparing periods.
- `duplicate_order_ids` is 0 after one load. Run the `ON_ERROR = 'CONTINUE'` `COPY` again: load metadata should keep it
  at 0 (lesson 05). Then run it with `FORCE = TRUE` and see what the check says.
- `null_phone` is **714**: every 7th customer has `"phone": null` (lesson 07).

## The checker

`check_answers.py` runs the same eight questions through the Python connector and compares them with the key:

```python
"""Run the project's check queries in Snowflake and compare them with answer_key.json.
Usage: python check_answers.py <connection name from config.toml>"""
import json, sys
import snowflake.connector

KEY = json.load(open("answer_key.json"))
Q = {
 "rows_loaded": "SELECT COUNT(*) FROM sales_lab.raw.orders",
 "monthly_paid_revenue": """SELECT TO_CHAR(DATE_TRUNC('month', ordered_at), 'YYYY-MM'), SUM(amount)
     FROM sales_lab.analytics.orders WHERE status = 'paid' GROUP BY 1 ORDER BY 1""",
 "refund_rate_by_channel": """SELECT channel, ROUND(AVG(IFF(status = 'refunded', 1, 0)), 4)
     FROM sales_lab.analytics.orders GROUP BY 1 ORDER BY 1""",
 "top3_countries_paid_revenue": """SELECT c.country, SUM(o.amount) AS revenue
     FROM sales_lab.analytics.orders o JOIN sales_lab.analytics.customers c USING (customer_id)
     WHERE o.status = 'paid' GROUP BY 1 ORDER BY revenue DESC LIMIT 3""",
 "customers_without_orders": """SELECT COUNT(*) FROM sales_lab.analytics.customers c WHERE NOT EXISTS
     (SELECT 1 FROM sales_lab.analytics.orders o WHERE o.customer_id = c.customer_id)""",
 "customers_active_6plus_months": """SELECT COUNT(*) FROM (SELECT customer_id FROM sales_lab.analytics.orders
     WHERE status = 'paid' GROUP BY 1 HAVING COUNT(DISTINCT DATE_TRUNC('month', ordered_at)) >= 6)""",
 "duplicate_order_ids": """SELECT COUNT(*) FROM (SELECT order_id FROM sales_lab.raw.orders
     GROUP BY 1 HAVING COUNT(*) > 1)""",
 "null_phone": "SELECT COUNT_IF(IS_NULL_VALUE(v:contact.phone)) FROM sales_lab.raw.customers_json",
}

def normalize(rows):
    out = [[float(x) if isinstance(x, (int, float)) or type(x).__name__ == "Decimal" else x for x in r] for r in rows]
    return out[0][0] if len(out) == 1 and len(out[0]) == 1 else out

with snowflake.connector.connect(connection_name=sys.argv[1]) as conn:
    cur = conn.cursor()
    failures = 0
    for name, sql in Q.items():
        got = normalize(cur.execute(sql).fetchall())
        want = KEY[name]
        ok = got == want if not isinstance(want, list) else [list(map(lambda v: float(v) if isinstance(v, (int, float)) else v, w)) for w in want] == got
        failures += not ok
        print(f"{'PASS' if ok else 'FAIL'}  {name}" + ("" if ok else f"\n      expected {want}\n      got      {got}"))
    print(f"\n{len(Q) - failures}/{len(Q)} checks match the answer key")
```

```bash
python check_answers.py trial        # 'trial' is the connection name from lesson 02
```

### Testing the checker without Snowflake

A checker that has never run is a guess, so it was exercised against the DuckDB copy of the data. A fake
`snowflake.connector` module translates each query from Snowflake SQL to DuckDB SQL with `sqlglot` and runs it:

```text
Argument 'format' is not supported for expression 'ToChar' when targeting DuckDB.
PASS  rows_loaded
FAIL  monthly_paid_revenue
      expected [['2026-01', 3733367.04], ['2026-02', 3447838.33], ...]
      got      [['2026-01-01 00:00:00', 3733367.04], ['2026-02-01 00:00:00', 3447838.33], ...]
PASS  refund_rate_by_channel
PASS  top3_countries_paid_revenue
PASS  customers_without_orders
PASS  customers_active_6plus_months
PASS  duplicate_order_ids
PASS  null_phone

7/8 checks match the answer key
```

Seven checks pass. The one failure is the **test double's** limitation, not the query's: `sqlglot` couldn't translate
Snowflake's `TO_CHAR(…, 'YYYY-MM')` format to DuckDB (it printed a warning), so the months came back as full timestamps.
The revenue values match exactly. In Snowflake, `TO_CHAR` with that format returns `2026-01`. This is a useful reminder
that a translated test proves the logic, not every function's exact output. The real run is in Snowflake.

## How It Actually Works

Everything you built is metadata plus files. The warehouse is a compute definition that runs only during the `COPY`
statements and queries. The `COPY` statements turned 3 MB of CSV and 0.9 MB of JSON into micro-partitions (only a
handful at this size). The views are stored queries that the optimizer expands into every query that uses them, so
`QUALIFY` deduplication and the VARIANT casts run each time a view is read. For a 200,000-row table that's trivial. For a
billion rows you'd materialize the cleaned data instead, as Level 2 does with streams, tasks and dynamic tables.

## Common mistakes

- **Skipping validation** and discovering bad rows in a dashboard.
- **Comparing a partial month with full months.**
- **Running the load twice with `FORCE = TRUE`.** The duplicate check will catch it.
- **Leaving the warehouse with the default 10-minute auto-suspend** for a lab you use for five minutes a day.
- **Querying raw VARIANT in reports** instead of the typed view.

## Exercise

1. Run the whole project in a trial account and get 8/8 from the checker. Record the credits used by `learn_wh` in Snowsight's
   cost view.
2. Add a question to the key and the checker: average order value per channel for paid orders. Compute the expected value
   yourself before running it in Snowflake.
3. Make the load idempotent end to end: running `project.sql` twice should leave every check passing. What needs to
   change, if anything?
4. Clone `sales_lab` to `sales_lab_dev`, delete all refunded orders in the clone, and show that the production checker
   still passes while a checker pointed at the clone fails the refund-rate question.
