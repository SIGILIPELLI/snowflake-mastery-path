---
description: "Snowpark Python: sessions from connections.toml, lazy DataFrames pushed down as SQL, the to_pandas trap, pandas on Snowflake, and local unit tests."
---

# 02 · Snowpark for Python

SQL is excellent at set logic and awkward at everything around it: reuse, parameters, branching, tests. **Snowpark** is
Snowflake's library for writing data transformations in Python (and Java and Scala) **without moving the data out of
Snowflake**. You build a query with a DataFrame API. Snowpark turns it into SQL, and the warehouse runs it. This lesson
covers how that works, the one habit that defeats it, and how to unit-test transformations with no account at all, which
was how every result below was produced.

## A session

A Snowpark session is a connection. It can reuse the same `connections.toml` profile Snowflake CLI uses (Level 1 · 02):

```python
from snowflake.snowpark import Session

session = Session.builder.config("connection_name", "learn").create()
print(session.get_current_role(), session.get_current_warehouse())
```

## DataFrames are lazy plans

A Snowpark DataFrame is **not data**. It's a description of a query. Each method returns a new, bigger description, and
nothing runs until you call an **action** (`collect()`, `show()`, `count()`, `to_pandas()`, `write.save_as_table()`). At that
point, Snowpark generates SQL for the whole chain, and Snowflake executes it on the session's warehouse:

```python
from snowflake.snowpark.functions import col, sum as sum_

orders = session.table("sales.mart.orders")
eu_daily = (orders.filter(col("region") == "EU")
                  .group_by("order_date")
                  .agg(sum_("amount").alias("revenue")))      # still nothing has run
eu_daily.explain()                                            # prints the generated SQL and the plan
eu_daily.write.save_as_table("sales.mart.eu_daily_revenue", mode="overwrite")   # runs one INSERT … SELECT in Snowflake
```

The aggregation runs where the data is, with pruning, the caches and the warehouse's full parallelism (Level 2). Python
only sends SQL and receives a status.

### The habit that defeats it

```python
pdf = session.table("sales.mart.orders").to_pandas()          # pulls the whole table into your laptop's memory
eu = pdf[pdf["REGION"] == "EU"].groupby("ORDER_DATE")["AMOUNT"].sum()
```

This works on a sample and fails in production. Every row crosses the network, and the work runs on one Python process
instead of a warehouse. Rule of thumb: **filter and aggregate in Snowpark, and call `to_pandas()` only on results small
enough to read**. If your team thinks in pandas, Snowflake's **pandas on Snowflake** (the Modin-based
`snowflake.snowpark.modin` API, installed with `snowflake-snowpark-python[modin]`) offers pandas syntax that is executed in
Snowflake instead of locally. Per its documentation, converting back with `to_pandas()` still materialises everything locally.

## Transformations as functions

Snowpark's real advantage over SQL files is structure. Write each transformation as a function from DataFrames to a
DataFrame, with no session and no SQL strings inside, and it becomes reusable and testable. `transforms.py`, a daily
report over a raw feed that re-sends orders when they change:

```python
"""Transformations for the daily customer report. Pure functions of DataFrames: no session, no SQL strings."""
from snowflake.snowpark import DataFrame
from snowflake.snowpark.functions import col, sum as sum_, count, count_distinct, row_number
from snowflake.snowpark.window import Window

def latest_version(orders: DataFrame) -> DataFrame:
    """Keep the most recent version of each order (the raw feed re-sends updated orders)."""
    w = Window.partition_by("order_id").order_by(col("updated_at").desc())
    return orders.with_column("rn", row_number().over(w)).filter(col("rn") == 1).drop("rn")

def region_summary(orders: DataFrame) -> DataFrame:
    return (latest_version(orders)
            .filter(col("status") != "cancelled")
            .group_by("region")
            .agg(count("order_id").alias("orders"),
                 count_distinct("customer_id").alias("customers"),
                 sum_("amount").alias("revenue")))
```

## Testing without an account

The Snowpark library includes a **local testing mode**: `Session.builder.config("local_testing", True)` creates an in-memory
session that executes DataFrame operations in Python. It's designed for exactly this kind of unit test. `test_transforms.py`
(each case states one rule the report must follow):

```python
import datetime as dt
import pytest
from snowflake.snowpark import Session
from transforms import region_summary

@pytest.fixture(scope="module")
def session():
    return Session.builder.config("local_testing", True).create()

def t(h): return dt.datetime(2026, 10, 14, h)

@pytest.fixture
def orders(session):
    return session.create_dataframe([
        # order_id, customer_id, region, amount, status, updated_at
        (1, 101, "EU", 100.0, "placed",    t(9)),
        (1, 101, "EU", 120.0, "placed",    t(11)),   # order 1 updated: only this version counts
        (2, 101, "EU",  30.0, "placed",    t(10)),   # same customer, second order
        (3, 102, "EU",  50.0, "placed",    t(9)),
        (3, 102, "EU",  50.0, "cancelled", t(12)),   # order 3 later cancelled
        (4, 201, "US",  80.0, "placed",    t(9)),
    ], schema=["order_id", "customer_id", "region", "amount", "status", "updated_at"])

def by_region(df):
    return {r["REGION"]: (r["ORDERS"], r["CUSTOMERS"], r["REVENUE"]) for r in df.collect()}

def test_updated_orders_count_once(orders):
    assert by_region(region_summary(orders))["EU"][0] == 2

def test_revenue_uses_latest_amount_and_skips_cancelled(orders):
    assert by_region(region_summary(orders))["EU"][2] == 150.0

def test_customers_are_distinct(orders):
    assert by_region(region_summary(orders))["EU"][1] == 1

def test_regions_are_independent(orders):
    assert by_region(region_summary(orders))["US"] == (1, 1, 80.0)
```

The first run, with `count("customer_id")` as originally written:

```text
E       assert np.int64(2) == 1
FAILED test_transforms.py::test_customers_are_distinct - assert np.int64(2) == 1
1 failed, 3 passed in 0.87s
```

Customer 101 placed two EU orders, so `count` counted them twice. It counts non-null values, not distinct ones.
`count_distinct` fixed it:

```text
test_transforms.py::test_updated_orders_count_once PASSED
test_transforms.py::test_revenue_uses_latest_amount_and_skips_cancelled PASSED
test_transforms.py::test_customers_are_distinct PASSED
test_transforms.py::test_regions_are_independent PASSED
============================== 4 passed in 0.69s ===============================
```

And the result itself (still the local session, Snowpark 1.53.0):

```text
-------------------------------------------------
|"REGION"  |"ORDERS"  |"CUSTOMERS"  |"REVENUE"  |
-------------------------------------------------
|EU        |2         |1            |150.0      |
|US        |1         |1            |80.0       |
-------------------------------------------------
```

Order 1 counts once at its updated amount (120), order 3 is dropped as cancelled, and EU revenue is 120 + 30 = 150.

### What local testing can't tell you

Per Snowflake's documentation, the local mode **doesn't parse SQL strings** (`session.sql(…)`, string filters), implements
only a **subset** of Snowflake's functions, ignores query tags and query history, and doesn't validate packages. It also
doesn't generate real SQL: in this run, `df.queries` returned a placeholder (`SELECT MOCK_TEST_FAKE_QUERY()`). So use it for
**logic**, which is where most bugs are, and run a smaller set of integration tests against a real development database for
SQL generation, performance and permissions. The documented way to keep one test suite for both is a fixture that creates
either a local or a real session from a command-line flag, and `pytest.mark.skipif` for cases the local mode can't support.

## How It Actually Works

Each DataFrame holds a **logical plan**: a tree of operations (scan, filter, project, aggregate, join, window). Methods add
nodes. An action asks Snowpark's plan compiler to turn the tree into SQL text, which the client sends to Snowflake like any
other query. That's why Snowpark code shows up in `QUERY_HISTORY` as ordinary SQL, why it gets the same optimizer, pruning and
caches, and why it uses the session's role and warehouse. Python itself does very little until data comes back.

In local testing mode the same plan tree is walked by an **in-memory executor** written in Python (using pandas). It
implements each node's semantics (grouping, windows, nulls) as Snowflake documents them, for the operations it supports. That
gives fast, deterministic unit tests, at the cost of fidelity at the edges, so a test passing locally means the logic is
right, not that the generated SQL is.

## Common mistakes

- **`to_pandas()` on large tables**, then doing the work in pandas. Push the work down and bring back results.
- **Loops of small actions.** Calling `.collect()` per customer is a query per customer. Express it as one DataFrame
  operation (a join or a window) instead.
- **SQL strings inside transformations** (`filter("amount > 20")`, `session.sql(…)`). They're untestable locally and lose
  the API's checks. Use column expressions.
- **No tests, or only end-to-end tests** against a shared database. Unit-test the logic locally, integration-test the rest.
- **Treating a passing local test as proof the warehouse will behave the same.** Check unsupported functions and run
  integration tests.

## Exercise

1. Install `snowflake-snowpark-python` (with `pandas`), copy `transforms.py` and `test_transforms.py`, and run the tests.
   Then break `latest_version` (order by `updated_at` ascending) and confirm which tests fail.
2. Add a fifth test: an order whose only version is cancelled must not create a region row with zero orders. Does the
   current code pass it?
3. With a trial account, run `region_summary` on `sales.mart.orders`, call `.explain()`, and compare the generated SQL with
   what you would have written.
4. Rewrite the report with pandas on Snowflake and compare the query count in `QUERY_HISTORY` for both versions.
