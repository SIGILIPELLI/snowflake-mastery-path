---
description: "Capstone: a cost-aware Snowflake platform with owned warehouses, monitors, budgets and a tested chargeback that assigns every credit, idle included."
---

# 10 · Capstone — A Cost-Aware Data Platform

The final project brings Level 4 together around the question every growing Snowflake account eventually faces: **who is
spending what, and is it worth it?** You'll set up a platform where every credit has an owner, runaway spend is stopped or
flagged, and a monthly chargeback assigns **all** compute, including the idle time that per-query attribution leaves out (lesson
05). The chargeback is the component everything else depends on, so it's tested, and the tests themselves are tested.

!!! note "What was run"
    The chargeback query was run in DuckDB 1.5.6 locally (not Snowflake) against synthetic tables shaped like the ACCOUNT_USAGE
    views, with seven pytest tests and a mutation check. The platform SQL, which embeds the same query, was syntax-checked and
    checked against Snowflake's October 2026 documentation, but not executed.

## Requirements

1. Every warehouse has a **cost owner**: a team for dedicated warehouses, and a team per query for shared ones.
2. Sandbox spend is **capped** daily. Shared spend is **flagged** before it gets out of hand. The whole account has a forecast
   with alerts.
3. A monthly **chargeback** assigns 100% of warehouse compute, with idle time shared in proportion to usage, and makes unowned
   spend visible instead of hiding it.
4. Platform health is one number: the share of credits nobody owns.

## The platform

`platform.sql`: ownership, workload warehouses, monitors, budget, and the chargeback as a scheduled task:

```sql
-- 1. ownership vocabulary and the FinOps schema
USE ROLE ACCOUNTADMIN;
CREATE DATABASE IF NOT EXISTS governance;
CREATE SCHEMA IF NOT EXISTS governance.finops;
CREATE TAG IF NOT EXISTS governance.finops.cost_center
  ALLOWED_VALUES 'sales', 'finance', 'marketing', 'data-science', 'data-platform';
CREATE ROLE IF NOT EXISTS finops_admin;                         -- runs the chargeback; not an administrator
GRANT ROLE finops_admin TO ROLE SYSADMIN;
GRANT DATABASE ROLE SNOWFLAKE.USAGE_VIEWER TO ROLE finops_admin;       -- WAREHOUSE_METERING_HISTORY, QUERY_ATTRIBUTION_HISTORY
GRANT DATABASE ROLE SNOWFLAKE.GOVERNANCE_VIEWER TO ROLE finops_admin;  -- TAG_REFERENCES
GRANT USAGE ON DATABASE governance TO ROLE finops_admin;
GRANT ALL ON SCHEMA governance.finops TO ROLE finops_admin;
GRANT EXECUTE TASK ON ACCOUNT TO ROLE finops_admin;

-- 2. warehouses by workload, each with an owner or a shared purpose
CREATE WAREHOUSE IF NOT EXISTS transform_wh WAREHOUSE_SIZE = 'MEDIUM' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE WITH TAG (governance.finops.cost_center = 'data-platform');
CREATE WAREHOUSE IF NOT EXISTS sales_bi_wh WAREHOUSE_SIZE = 'SMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE
  MIN_CLUSTER_COUNT = 1 MAX_CLUSTER_COUNT = 3 SCALING_POLICY = 'STANDARD'
  INITIALLY_SUSPENDED = TRUE WITH TAG (governance.finops.cost_center = 'sales');
CREATE WAREHOUSE IF NOT EXISTS shared_wh WAREHOUSE_SIZE = 'SMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE COMMENT = 'Shared: charged by QUERY_TAG team=<cost center>';
CREATE WAREHOUSE IF NOT EXISTS sandbox_wh WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;

-- shared-warehouse users and services identify their team by default
ALTER USER etl_service SET QUERY_TAG = 'team=data-platform';
ALTER USER ana SET QUERY_TAG = 'team=sales';

-- 3. circuit breakers (warehouses only) and forecasts (everything)
CREATE OR REPLACE RESOURCE MONITOR sandbox_daily WITH CREDIT_QUOTA = 5 FREQUENCY = DAILY START_TIMESTAMP = IMMEDIATELY
  TRIGGERS ON 80 PERCENT DO NOTIFY ON 100 PERCENT DO SUSPEND_IMMEDIATE;
ALTER WAREHOUSE sandbox_wh SET RESOURCE_MONITOR = sandbox_daily;
CREATE OR REPLACE RESOURCE MONITOR shared_monthly WITH CREDIT_QUOTA = 300 FREQUENCY = MONTHLY START_TIMESTAMP = IMMEDIATELY
  TRIGGERS ON 75 PERCENT DO NOTIFY ON 90 PERCENT DO NOTIFY ON 110 PERCENT DO SUSPEND;
ALTER WAREHOUSE shared_wh SET RESOURCE_MONITOR = shared_monthly;

CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!ACTIVATE();
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!SET_SPENDING_LIMIT(2000);
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!SET_EMAIL_NOTIFICATIONS('finops@example.com');

-- 4. the monthly chargeback, materialised on the 2nd (attribution data can lag by up to eight hours)
USE ROLE SYSADMIN;
GRANT USAGE ON WAREHOUSE transform_wh TO ROLE finops_admin;
USE ROLE finops_admin;
CREATE TABLE IF NOT EXISTS governance.finops.chargeback (
  month DATE, warehouse_name VARCHAR, cost_center VARCHAR, credits NUMBER(38, 6), basis VARCHAR);

CREATE OR REPLACE TASK governance.finops.monthly_chargeback
  WAREHOUSE = transform_wh
  SCHEDULE = 'USING CRON 0 6 2 * * UTC'
AS
INSERT INTO governance.finops.chargeback
with metered as (
    select warehouse_name,
           sum(credits_used_compute) as compute,
           sum(credits_attributed_compute_queries) as attributed
    from SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
    where start_time >= DATE_TRUNC(month, DATEADD(month, -1, CURRENT_DATE())) and start_time < DATE_TRUNC(month, CURRENT_DATE())
    group by warehouse_name
),
owner as (                                            -- dedicated warehouses carry a cost_center tag
    select object_name as warehouse_name, tag_value as cost_center
    from SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
    where domain = 'WAREHOUSE' and tag_name = 'COST_CENTER' and tag_database = 'GOVERNANCE'
),
by_team as (                                          -- shared warehouses: queries carry 'team=<name>;...'
    select warehouse_name,
           coalesce(nullif(split_part(split_part(query_tag, 'team=', 2), ';', 1), ''), 'untagged') as cost_center,
           sum(credits_attributed_compute) as team_attributed
    from SNOWFLAKE.ACCOUNT_USAGE.QUERY_ATTRIBUTION_HISTORY
    where start_time >= DATE_TRUNC(month, DATEADD(month, -1, CURRENT_DATE())) and start_time < DATE_TRUNC(month, CURRENT_DATE())
    group by 1, 2
),
team_share as (
    select warehouse_name, cost_center,
           team_attributed / sum(team_attributed) over (partition by warehouse_name) as share
    from by_team
)
select DATE_TRUNC(month, DATEADD(month, -1, CURRENT_DATE())), m.warehouse_name, o.cost_center, m.compute as credits, 'dedicated' as basis
from metered m join owner o using (warehouse_name)
union all
select DATE_TRUNC(month, DATEADD(month, -1, CURRENT_DATE())), m.warehouse_name, s.cost_center, m.compute * s.share, 'shared: usage plus proportional idle'
from metered m join team_share s using (warehouse_name)
where m.warehouse_name not in (select warehouse_name from owner)
union all
select DATE_TRUNC(month, DATEADD(month, -1, CURRENT_DATE())), m.warehouse_name, 'unallocated', m.compute, 'no owner tag and no attributed queries'
from metered m
where m.warehouse_name not in (select warehouse_name from owner)
  and m.warehouse_name not in (select warehouse_name from team_share);

ALTER TASK governance.finops.monthly_chargeback RESUME;

-- 5. a month is healthy when almost nothing is unowned
SELECT month,
       SUM(IFF(cost_center IN ('untagged', 'unallocated'), credits, 0)) / SUM(credits) AS unowned_share
FROM governance.finops.chargeback
GROUP BY month
ORDER BY month DESC;
```

Design choices, with the lessons behind them:

- **Warehouses by workload, not by person** (lessons 05–06): transformation, BI (multi-cluster `STANDARD` for Monday-morning
  bursts), a shared warehouse for small teams, and a capped sandbox. All have `AUTO_SUSPEND = 60`, because idle time was 86% of
  the bill in lesson 05's model with a 10-minute setting.
- **Two ownership mechanisms**: an object tag on dedicated warehouses, and `QUERY_TAG` defaults on users for the shared one. Pipelines
  can override the tag per job.
- **Monitors where you must stop spend, the budget for everything else.** The sandbox monitor suspends immediately at its daily
  quota. The shared warehouse's notifies at 75% and 90% and suspends only at 110%, because stopping a shared warehouse stops several
  teams. The account budget covers serverless and AI services, which monitors can't.
- **A dedicated, non-admin role** runs the chargeback, with exactly the SNOWFLAKE database roles its views need (`USAGE_VIEWER`,
  `GOVERNANCE_VIEWER`), per the documentation.
- **The task runs on the 2nd**, because `QUERY_ATTRIBUTION_HISTORY` can lag by up to eight hours. A chargeback run at midnight on
  the 1st would compute shares from incomplete data.

## The chargeback, tested

The allocation logic as one SQL statement (`chargeback.sql`), written to run unchanged on DuckDB for testing, with parameters for
the month:

```sql
-- Monthly chargeback: every compute credit of every warehouse assigned to exactly one cost center.
-- Inputs mirror SNOWFLAKE.ACCOUNT_USAGE: WAREHOUSE_METERING_HISTORY, QUERY_ATTRIBUTION_HISTORY, TAG_REFERENCES.
with metered as (
    select warehouse_name,
           sum(credits_used_compute) as compute,
           sum(credits_attributed_compute_queries) as attributed
    from warehouse_metering_history
    where start_time >= :month_start and start_time < :month_end
    group by warehouse_name
),
owner as (                                            -- dedicated warehouses carry a cost_center tag
    select object_name as warehouse_name, tag_value as cost_center
    from tag_references
    where domain = 'WAREHOUSE' and tag_name = 'COST_CENTER'
),
by_team as (                                          -- shared warehouses: queries carry 'team=<name>;...'
    select warehouse_name,
           coalesce(nullif(split_part(split_part(query_tag, 'team=', 2), ';', 1), ''), 'untagged') as cost_center,
           sum(credits_attributed_compute) as team_attributed
    from query_attribution_history
    where start_time >= :month_start and start_time < :month_end
    group by 1, 2
),
team_share as (
    select warehouse_name, cost_center,
           team_attributed / sum(team_attributed) over (partition by warehouse_name) as share
    from by_team
)
select m.warehouse_name, o.cost_center, m.compute as credits, 'dedicated' as basis
from metered m join owner o using (warehouse_name)
union all
select m.warehouse_name, s.cost_center, m.compute * s.share, 'shared: usage plus proportional idle'
from metered m join team_share s using (warehouse_name)
where m.warehouse_name not in (select warehouse_name from owner)
union all
select m.warehouse_name, 'unallocated', m.compute, 'no owner tag and no attributed queries'
from metered m
where m.warehouse_name not in (select warehouse_name from owner)
  and m.warehouse_name not in (select warehouse_name from team_share)
```

The test data (`fixtures.py`) covers each case the logic must handle: a dedicated warehouse that also has tagged queries (it must
not be counted twice), a shared warehouse with three teams and one untagged query, a dedicated warehouse fully used, an idle sandbox
with no owner, and rows from the previous month that must be ignored:

```python
"""Synthetic October 2026 usage for four warehouses, as tables shaped like the ACCOUNT_USAGE views (DuckDB, local)."""
import duckdb

def load(con: duckdb.DuckDBPyConnection) -> None:
    con.execute("""create or replace table warehouse_metering_history as select * from (values
        ('SALES_BI_WH',  timestamp '2026-10-01 09:00', 30.0, 6.0),
        ('SALES_BI_WH',  timestamp '2026-10-15 09:00', 10.0, 4.0),
        ('SHARED_WH',    timestamp '2026-10-02 10:00', 40.0, 20.0),
        ('SHARED_WH',    timestamp '2026-10-20 10:00', 20.0, 10.0),
        ('SHARED_WH',    timestamp '2026-09-30 23:00', 99.0, 99.0),   -- previous month: must be ignored
        ('DS_WH',        timestamp '2026-10-03 14:00', 25.0, 25.0),
        ('SANDBOX_WH',   timestamp '2026-10-07 08:00',  3.0, 0.0)
        ) t(warehouse_name, start_time, credits_used_compute, credits_attributed_compute_queries)""")
    con.execute("""create or replace table query_attribution_history as select * from (values
        ('SHARED_WH',  timestamp '2026-10-02 10:05', 'team=finance;job=close',     12.0),
        ('SHARED_WH',  timestamp '2026-10-02 10:30', 'team=marketing',              6.0),
        ('SHARED_WH',  timestamp '2026-10-20 10:10', 'team=finance',                9.0),
        ('SHARED_WH',  timestamp '2026-10-20 10:40', '',                            3.0),
        ('SHARED_WH',  timestamp '2026-09-30 23:10', 'team=marketing',             99.0),
        ('SALES_BI_WH',timestamp '2026-10-01 09:10', 'team=finance',                6.0),
        ('DS_WH',      timestamp '2026-10-03 14:05', 'team=data-science;job=fit',  25.0)
        ) t(warehouse_name, start_time, query_tag, credits_attributed_compute)""")
    con.execute("""create or replace table tag_references as select * from (values
        ('WAREHOUSE', 'SALES_BI_WH', 'COST_CENTER', 'sales'),
        ('WAREHOUSE', 'DS_WH',       'COST_CENTER', 'data-science'),
        ('TABLE',     'ORDERS',      'COST_CENTER', 'sales')
        ) t(domain, object_name, tag_name, tag_value)""")
```

`test_chargeback.py` states the invariants:

```python
"""Invariants the chargeback must satisfy, run against chargeback.sql in DuckDB (local, not Snowflake)."""
import duckdb, pytest
from fixtures import load

SQL = open("chargeback.sql").read()

@pytest.fixture(scope="module")
def rows():
    con = duckdb.connect(); load(con)
    sql = SQL.replace(":month_start", "timestamp '2026-10-01'").replace(":month_end", "timestamp '2026-11-01'")
    return con.execute(sql).fetchall()      # (warehouse, cost_center, credits, basis)

def total(rows, **match):
    return sum(r[2] for r in rows if all(r[i] == v for i, v in ((0, match.get("wh")), (1, match.get("cc"))) if v))

def test_every_credit_is_allocated_exactly_once(rows):
    assert total(rows) == pytest.approx(30 + 10 + 40 + 20 + 25 + 3)        # October compute, nothing more

def test_each_warehouse_sums_to_its_meter(rows):
    for wh, metered in {"SALES_BI_WH": 40, "SHARED_WH": 60, "DS_WH": 25, "SANDBOX_WH": 3}.items():
        assert total(rows, wh=wh) == pytest.approx(metered)

def test_dedicated_warehouse_goes_to_its_owner_including_idle(rows):
    assert [(r[1], r[2]) for r in rows if r[0] == "SALES_BI_WH"] == [("sales", 40)]

def test_shared_idle_is_split_in_proportion_to_usage(rows):
    # October attributed on SHARED_WH: finance 21, marketing 6, untagged 3 (of 30); compute 60
    assert total(rows, wh="SHARED_WH", cc="finance") == pytest.approx(60 * 21 / 30)
    assert total(rows, wh="SHARED_WH", cc="marketing") == pytest.approx(60 * 6 / 30)

def test_queries_without_a_team_are_visible_not_hidden(rows):
    assert total(rows, wh="SHARED_WH", cc="untagged") == pytest.approx(60 * 3 / 30)

def test_idle_warehouse_without_owner_is_reported_as_unallocated(rows):
    assert [(r[1], r[2]) for r in rows if r[0] == "SANDBOX_WH"] == [("unallocated", 3)]

def test_no_negative_or_null_allocations(rows):
    assert all(r[1] is not None and r[2] is not None and r[2] >= 0 for r in rows)
```

```text
test_chargeback.py::test_every_credit_is_allocated_exactly_once PASSED
test_chargeback.py::test_each_warehouse_sums_to_its_meter PASSED
test_chargeback.py::test_dedicated_warehouse_goes_to_its_owner_including_idle PASSED
test_chargeback.py::test_shared_idle_is_split_in_proportion_to_usage PASSED
test_chargeback.py::test_queries_without_a_team_are_visible_not_hidden PASSED
test_chargeback.py::test_idle_warehouse_without_owner_is_reported_as_unallocated PASSED
test_chargeback.py::test_no_negative_or_null_allocations PASSED
============================== 7 passed in 0.15s ===============================
```

The result for October:

```text
DS_WH        data-science    25.0  dedicated
SALES_BI_WH  sales           40.0  dedicated
SANDBOX_WH   unallocated      3.0  no owner tag and no attributed queries
SHARED_WH    finance         42.0  shared: usage plus proportional idle
SHARED_WH    marketing       12.0  shared: usage plus proportional idle
SHARED_WH    untagged         6.0  shared: usage plus proportional idle
```

All 128 October credits are assigned. Finance pays 42 on the shared warehouse: 21 credits of its own queries plus its 70% share of
the 30 idle credits.

### Testing the tests

All seven tests passed on the first run, which proves less than it seems: tests that can't fail prove nothing. So the suite was run
against three deliberately wrong versions of the query, each a mistake a real chargeback could make (`mutants.py`):

```python
# Runs the test suite against deliberately wrong versions of chargeback.sql: does it catch each mistake?
import pathlib, shutil, subprocess, sys
MUTANTS = {
    "charge shared warehouses only for attributed usage (no idle)":
        ("m.compute * s.share", "m.attributed * s.share"),
    "forget the month filter on query attribution":
        ("from query_attribution_history\n    where start_time >= :month_start and start_time < :month_end",
         "from query_attribution_history\n    where true"),
    "drop queries without a team tag":
        ("coalesce(nullif(split_part(split_part(query_tag, 'team=', 2), ';', 1), ''), 'untagged')",
         "nullif(split_part(split_part(query_tag, 'team=', 2), ';', 1), '')"),
}
original = pathlib.Path("chargeback.sql").read_text()
shutil.copy("chargeback.sql", "chargeback.sql.orig")
try:
    for name, (old, new) in MUTANTS.items():
        assert old in original, name
        pathlib.Path("chargeback.sql").write_text(original.replace(old, new))
        r = subprocess.run([sys.executable, "-m", "pytest", "-q", "-p", "no:cacheprovider", "test_chargeback.py"],
                           capture_output=True, text=True)
        failed = [l.split("::")[1].split(" ")[0] for l in r.stdout.splitlines() if l.startswith("FAILED")]
        print(f"{name}:\n  {'CAUGHT by ' + ', '.join(failed) if failed else 'NOT CAUGHT'}")
finally:
    shutil.move("chargeback.sql.orig", "chargeback.sql")
```

```text
charge shared warehouses only for attributed usage (no idle):
  CAUGHT by test_every_credit_is_allocated_exactly_once, test_each_warehouse_sums_to_its_meter, test_shared_idle_is_split_in_proportion_to_usage, test_queries_without_a_team_are_visible_not_hidden
forget the month filter on query attribution:
  CAUGHT by test_shared_idle_is_split_in_proportion_to_usage, test_queries_without_a_team_are_visible_not_hidden
drop queries without a team tag:
  CAUGHT by test_queries_without_a_team_are_visible_not_hidden, test_no_negative_or_null_allocations
```

Each mistake is caught, and the second result is the instructive one. Forgetting the month filter on attribution still allocates
exactly the metered total, so **both "sums to the meter" tests pass**. It's caught only because a test checks the **proportions**,
which September's queries distort. A suite of invariants alone would have shipped that bug. Exact expected values for a hand-checked
case are what caught it.

### From tested SQL to the deployed task

The task in `platform.sql` was generated from `chargeback.sql` by four mechanical substitutions: the three table names become the
`SNOWFLAKE.ACCOUNT_USAGE` views, the month parameters become "last calendar month", the tag lookup is restricted to the governance
database's tag, and the month is added as the first column. Generating it, rather than editing a copy, keeps the deployed logic
identical to the tested logic.

## Running the platform

A monthly FinOps review, an hour a month:

1. **Unowned share** (the last query in `platform.sql`). Above a few percent, fix tags and default query tags first, because every
   other number depends on ownership.
2. **Idle share per warehouse** (lesson 05's query). High idle points to `AUTO_SUSPEND`, refresh schedules or consolidation.
3. **Top repeated queries** by `query_parameterized_hash` in `QUERY_ATTRIBUTION_HISTORY`. One dashboard query run 10,000 times a month
   is usually the cheapest big win: cache it, materialize it, or refresh it less often.
4. **Service-type lines** in `METERING_DAILY_HISTORY` (serverless, AI services, replication), against the budget forecast.
5. **Queuing vs spilling** per warehouse (lesson 06), to choose scale-out or scale-up, or neither.

## How It Actually Works

The platform works because each piece reads the same underlying meter at a different speed and for a different purpose. Resource
monitors watch warehouse credits close to real time and can act, so they're the brakes. Budgets read consolidated metering for all
services every few hours and project to month end, so they're the early warning. Account Usage views are the complete, lagging
record, so the chargeback reads them once the month is settled. Tags and query tags are the join keys that turn that record into
owners. The chargeback is a closed allocation: for each warehouse, metered compute is split among owners by shares that sum to
one. That's why "every credit allocated exactly once" holds whatever the shares are, and why correctness lives in the shares, which is
where the tests have to look.

## Common mistakes

- **Chargeback from attribution alone**, leaving idle time (often most of the bill) on nobody's books.
- **Running the chargeback before the data settles.** Wait for the documented latency.
- **Monitors that suspend shared warehouses at 100%**, stopping several teams for one team's spike.
- **Untagged warehouses and missing default query tags.** Ownership has to be set up first.
- **Tests that check only totals.** Totals hide shifted proportions. Pin hand-checked values too.
- **Editing a copy of tested SQL for production.** Generate the deployed version from the tested one.

## Exercise

1. Run the tests locally (`pip install duckdb pytest`, then `pytest test_chargeback.py`), then `python mutants.py`. Add a fourth
   mutant (for example, splitting idle **equally** between teams instead of by usage) and make sure a test catches it.
2. Extend the chargeback to serverless costs: assign `SERVERLESS_TASK` credits from `METERING_HISTORY` to the task's database owner
   tag. Write the tests first.
3. In a trial account, run `platform.sql` (adjusting users and emails), run some tagged and untagged queries on `shared_wh`, and two
   days later run the chargeback query by hand for the current month. What's your unowned share?
4. Write the one-page FinOps review for that month, using the five checks above, with one recommended change and its expected saving.
