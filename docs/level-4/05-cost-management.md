---
description: "Snowflake cost management: where credits go, per-query attribution and its idle gap, chargeback with tags, resource monitors, budgets and their limits."
---

# 05 · Cost Management: Monitors, Budgets & Attribution

Level 1 explained how Snowflake bills: credits per second while warehouses run, plus serverless features, cloud services
and storage. This lesson is about **managing** that bill: seeing where credits go, charging them back to the teams that
spent them, and stopping runaway spend before it happens. Each tool has a documented blind spot, and knowing those blind
spots is the difference between a cost dashboard and a cost dashboard you can trust.

## Where the credits go

`METERING_DAILY_HISTORY` breaks credits down by **service type**. Per the documentation, that includes warehouses, serverless
tasks, Snowpipe Streaming, automatic clustering, materialized views, search optimization, query acceleration, replication, AI
services and more:

```sql
SELECT service_type,
       SUM(credits_used_compute)              AS compute,
       SUM(credits_used_cloud_services)       AS cloud_services,
       SUM(credits_adjustment_cloud_services) AS cs_adjustment,
       SUM(credits_billed)                    AS billed
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATEADD(day, -30, CURRENT_DATE())
GROUP BY service_type
ORDER BY billed DESC;
```

`credits_billed` already includes the cloud services **adjustment**: cloud services usage is billed only above 10% of the
day's warehouse compute (Level 1 · 04). Start every cost review here. A team obsessing over warehouse sizes is wasting effort
if search optimization or a forgotten serverless task is the biggest line.

## Attribution: which query spent it?

`QUERY_ATTRIBUTION_HISTORY` gives the warehouse credits attributed to each query (`credits_attributed_compute`), with its
user, warehouse, query tag and a parameterized hash for grouping repeated queries. Its documented limits:

- It **excludes warehouse idle time**: credits for a warehouse running with nothing to do aren't attributed to any query.
- Very short queries (about 100 ms or less) aren't included.
- Latency is **up to eight hours**.

The first limit matters most, because idle time can be most of a bill. A model of one shared X-Small warehouse for a day,
with BI dashboards refreshing every 15 minutes, one nightly job and two ad-hoc sessions:

```python
# Model (not Snowflake output): one shared X-Small warehouse (1 credit/hour) for a day, billed per second with a 60 s
# minimum per resume. Assumptions: the warehouse resumes at a query's start and suspends AUTO_SUSPEND seconds after the
# last query ends; a second shared by n running queries is attributed 1/n to each (Snowflake's exact split isn't modelled).
import random
random.seed(42)
queries = []                                                    # (team, start_s, duration_s)
for minute in range(8 * 60, 18 * 60, 15):                       # BI dashboard refreshes every 15 min, 08:00-18:00
    for _ in range(random.randint(1, 4)):
        queries.append(("bi", minute * 60 + random.randint(0, 30), random.randint(2, 20)))
queries.append(("etl", 2 * 3600, 40 * 60))                       # nightly ELT: one 40-minute job at 02:00
for h in (10, 14):                                              # ad-hoc data science twice a day
    queries.append(("ds", h * 3600 + 600, random.randint(300, 900)))

def day(auto_suspend):
    active = {}                                                 # second -> teams running
    for team, s, d in queries:
        for t in range(s, s + d):
            active.setdefault(t, []).append(team)
    running, t_end = set(), None
    for t in sorted(active):                                    # build running intervals
        if t_end is None or t > t_end + auto_suspend:
            if t_end is not None:
                running |= set(range(start, max(t_end + auto_suspend, start + 60)))
            start = t
        t_end = t + 1
    running |= set(range(start, max(t_end + auto_suspend, start + 60)))
    rate = 1 / 3600
    metered = len(running) * rate
    attributed = {}
    for t, teams in active.items():
        for team in teams:
            attributed[team] = attributed.get(team, 0) + rate / len(teams)
    return metered, attributed

print(f"queries: {len(queries)}")
for s in (600, 60):
    metered, attr = day(s)
    total = sum(attr.values())
    print(f"\nAUTO_SUSPEND = {s}")
    print(f"  metered (WAREHOUSE_METERING_HISTORY-like): {metered:6.3f} credits")
    for team in sorted(attr):
        print(f"  attributed to {team:<4}{attr[team]:24.3f} credits")
    print(f"  idle, attributed to nobody: {metered - total:15.3f} credits ({(metered - total) / metered:.0%} of the bill)")
```

```text
queries: 96

AUTO_SUSPEND = 600
  metered (WAREHOUSE_METERING_HISTORY-like):  7.967 credits
  attributed to bi                     0.209 credits
  attributed to ds                     0.261 credits
  attributed to etl                    0.667 credits
  idle, attributed to nobody:           6.829 credits (86% of the bill)

AUTO_SUSPEND = 60
  metered (WAREHOUSE_METERING_HISTORY-like):  1.850 credits
  attributed to bi                     0.209 credits
  attributed to ds                     0.261 credits
  attributed to etl                    0.667 credits
  idle, attributed to nobody:           0.712 credits (39% of the bill)
```

The teams' **attributed** credits are identical in both runs, because the queries didn't change. The **bill** differs by a
factor of four, and with a 10-minute auto-suspend, 86% of it is attributed to nobody. A chargeback report that sums
`credits_attributed_compute` by team would show about 1.1 credits for a warehouse that cost 8. The numbers come from a model
with stated assumptions, but the shape doesn't: bursty, frequent, short queries with a long auto-suspend produce mostly idle
time.

Snowflake exposes the gap directly. `WAREHOUSE_METERING_HISTORY` has both the compute credits and the part attributed to
queries:

```sql
SELECT warehouse_name,
       SUM(credits_used_compute)                                            AS compute,
       SUM(credits_attributed_compute_queries)                              AS attributed,
       SUM(credits_used_compute) - SUM(credits_attributed_compute_queries)  AS idle,
       ROUND(100 * idle / NULLIF(compute, 0), 1)                            AS idle_pct
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY idle DESC;
```

A high `idle_pct` is a tuning signal (shorter `AUTO_SUSPEND`, fewer refreshes, consolidated schedules) and a chargeback
decision: idle credits must be **allocated** to someone, typically in proportion to each team's attributed usage on that
warehouse, which is also the approach in Snowflake's own cost-attribution examples. The capstone (lesson 10) builds that allocator with tests.

## Chargeback with tags

Attribution needs an owner for every credit. Two documented mechanisms:

- **Object tags** on warehouses (and other objects): with a dedicated warehouse per team, tag it with `cost_center`, and its
  whole metered cost (idle included) belongs to that team. Join `WAREHOUSE_METERING_HISTORY` to `TAG_REFERENCES`.
- **Query tags** for shared warehouses: set `QUERY_TAG` per session (tools and pipelines can set it), and group
  `QUERY_ATTRIBUTION_HISTORY` by it.

```sql
CREATE TAG IF NOT EXISTS governance.tags.cost_center ALLOWED_VALUES 'sales', 'hr', 'data-platform', 'data-science';
ALTER WAREHOUSE sales_bi_wh SET TAG governance.tags.cost_center = 'sales';
ALTER WAREHOUSE sales_etl_wh SET TAG governance.tags.cost_center = 'sales';

-- in a pipeline's session
ALTER SESSION SET QUERY_TAG = 'team=data-science;job=churn-features';

SELECT t.tag_value AS cost_center, SUM(m.credits_used) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY m
JOIN SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES t
  ON t.domain = 'WAREHOUSE' AND t.object_name = m.warehouse_name AND t.tag_name = 'COST_CENTER'
WHERE m.start_time >= DATE_TRUNC(month, CURRENT_DATE())
GROUP BY cost_center
ORDER BY credits DESC;
```

## Resource monitors: hard stops for warehouses

A resource monitor counts credits against a quota and acts at thresholds:

```sql
USE ROLE ACCOUNTADMIN;
CREATE OR REPLACE RESOURCE MONITOR sales_bi_monthly WITH
  CREDIT_QUOTA = 100
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS ON 75 PERCENT DO NOTIFY
           ON 90 PERCENT DO SUSPEND
           ON 100 PERCENT DO SUSPEND_IMMEDIATE;

ALTER WAREHOUSE sales_bi_wh SET RESOURCE_MONITOR = sales_bi_monthly;
```

From the documentation:

- They work for **warehouses only**. Serverless features, AI services and storage aren't covered.
- Only `ACCOUNTADMIN` can create them. A warehouse can belong to **one** monitor, and an account has at most one
  **account-level** monitor.
- `SUSPEND` lets running statements finish. `SUSPEND_IMMEDIATE` cancels them.
- Quotas count **all** cloud services credits for the warehouses, **ignoring** the 10% adjustment, so a monitor can trip
  before the bill shows that spend.
- Usage resets at 12:00 AM UTC on the schedule's day. For strict limits, Snowflake recommends **buffers** (act at 90%, not
  100%) and one warehouse per monitor, so one warehouse's spend can't suspend another.

Use them as **circuit breakers** for things that must never run away: reader-account warehouses (Level 3 · 07), sandbox
warehouses, and anything that runs unattended.

## Budgets: forecasts and alerts for everything

A **budget** sets a **monthly** spending limit and notifies when spend is **projected** to exceed it. It covers serverless
features and AI services as well as warehouses. There's one **account budget**, and **custom budgets** for groups of objects,
which can be selected by tag:

```sql
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!ACTIVATE();
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!SET_SPENDING_LIMIT(1000);
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!SET_EMAIL_NOTIFICATIONS('costadmin@example.com, finance@example.com');

CREATE SNOWFLAKE.CORE.BUDGET governance.budgets.sales_budget();
CALL governance.budgets.sales_budget!SET_SPENDING_LIMIT(300);
CALL governance.budgets.sales_budget!ADD_RESOURCE_TAG(
  SELECT SYSTEM$REFERENCE('TAG', 'governance.tags.cost_center', 'SESSION', 'applybudget'), 'sales');
```

Budgets differ from monitors in the ways that matter:

| | Resource monitor | Budget |
|---|---|---|
| Covers | warehouses only | warehouses, serverless features, AI services, … |
| Period | daily, weekly, monthly, yearly or never | calendar month |
| Acts | notifies, **suspends** warehouses | notifies (email, cloud queue, webhook); can call your stored procedures |
| Timing | thresholds on actual usage | projections, refreshed up to every 6.5 hours by default |

Per the documentation, a **low-latency** budget refreshes hourly, at **12 times** the budget's own compute cost. A custom budget
whose objects are added **individually** isn't backfilled for the current month, while objects added **by tag** are.
Prefer tags.

## How It Actually Works

Every credit Snowflake meters is recorded against a service and, where there is one, an object: warehouse seconds per
warehouse, serverless seconds per task, pipe or table, tokens per AI call. Account Usage views are periodically refreshed
copies of those records, which is why they lag. Attribution is a second calculation on top of metering. For each warehouse
interval, Snowflake attributes the credits to the queries running in it, by a weighted average of their resource consumption
(per the documentation; the model above simply splits equally), and what no query used is idle time. Resource
monitors read the warehouse meters frequently and act on warehouses directly, which is why they can stop spend. Budgets read the
consolidated metering data on a slower schedule and project to month end, which is why they can cover every service but only
warn (or call your procedure). A complete setup uses all three: metering for the truth, attribution plus allocation for
chargeback, monitors as circuit breakers, and budgets as the early warning.

## Common mistakes

- **Chargeback from `QUERY_ATTRIBUTION_HISTORY` alone.** It leaves out idle time, which can be most of the bill.
- **Relying on resource monitors to cap total spend.** They cover warehouses only.
- **Thresholds at exactly 100%** when the limit is a hard one. Monitors aren't instantaneous. Use buffers.
- **One monitor for many warehouses** when one team's burst shouldn't suspend another team's warehouse.
- **Untagged warehouses.** Every warehouse needs a cost owner from day one.
- **Custom budgets built object by object.** Use tags, so new objects are covered and history is backfilled.

## Exercise

1. Run the service-type query on a trial account. Which three lines are largest, and which of them could a resource monitor
   stop?
2. Run the idle query for your warehouses. Pick the one with the highest `idle_pct`, change its `AUTO_SUSPEND`, and compare a
   week before and after.
3. Change `attribution_gap.py` so the BI refreshes run every 5 minutes, then every 60. Predict the idle share first.
4. Create a monitor for a sandbox warehouse with a tiny quota (1 credit, daily), run queries until it fires, and record what
   happened to a running query at each trigger.
