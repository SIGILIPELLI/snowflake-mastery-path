---
description: "dbt and CI/CD for Snowflake: a tested dbt project, an incremental bug that passed every test, clones per pull request, and dbt Projects on Snowflake."
---

# 08 · dbt & CI/CD for Snowflake

Level 2's pipelines were SQL objects created by hand. In a team, transformation code needs what application code has: version
control, review, automated tests and repeatable deployment. **dbt** is the most common way to get that for SQL. Snowflake adds
pieces that make it cheaper than elsewhere: **zero-copy clones** give every pull request a full production-shaped environment for
free, and **dbt Projects on Snowflake** run dbt inside the account. This lesson builds a small dbt project, shows a bug that
survived every standard test, and wires it into CI.

!!! note "What was run"
    The dbt project was run for real with dbt-core 1.12.5 against **DuckDB 1.5.6 (dbt-duckdb 1.11.0) locally, not Snowflake**,
    through three batches of data. Its compiled SQL was checked to parse as Snowflake SQL, and `dbt parse` ran against the
    Snowflake targets (dbt-snowflake 1.12.2, which needs no connection to parse). The CI workflow and Snowflake CLI commands
    were checked for syntax and against `snow dbt --help` (Snowflake CLI 3.29.0), but not run.

## The project

A staging model that keeps the latest version of each order, and an **incremental** mart of daily revenue by the day each order was
created. `profiles.yml` has a local DuckDB target for development and Snowflake targets for CI and production:

```yaml
sales:
  target: local
  outputs:
    local:                                   # DuckDB file, for development without an account
      type: duckdb
      path: dev.duckdb
      threads: 4
    ci:                                      # a per-pull-request clone in Snowflake (key-pair auth from env vars)
      type: snowflake
      ...
```

`models/staging/stg_orders.sql`:

```sql+jinja
-- one row per order: the latest version the feed sent, plus when the order was first seen
select order_id, customer_id, region, amount, status,
       min(updated_at) over (partition by order_id) as created_at,
       updated_at
from {{ source('raw', 'orders') }}
qualify row_number() over (partition by order_id order by updated_at desc) = 1
```

`models/marts/fct_daily_revenue.sql`, first version. On incremental runs it reprocesses only days from the latest day already
loaded:

```sql+jinja
{{ config(materialized='incremental', unique_key=['order_date', 'region']) }}

select cast(created_at as date) as order_date, region,
       count(*) as orders, sum(amount) as revenue,
       max(updated_at) as last_change
from {{ ref('stg_orders') }}
where status <> 'cancelled'
{% if is_incremental() %}
  -- reprocess from the latest day already loaded
  and cast(created_at as date) >= (select max(order_date) from {{ this }})
{% endif %}
group by 1, 2
```

The tests: `unique`, `not_null` and `accepted_values` on staging, a uniqueness test on the mart's key, and a dbt **unit test**
that feeds four hand-written rows into the mart and checks the exact output:

```yaml
version: 2
models:
  - name: stg_orders
    columns:
      - name: order_id
        data_tests: [unique, not_null]
      - name: status
        data_tests:
          - accepted_values:
              arguments:
                values: ['placed', 'shipped', 'cancelled']
  - name: fct_daily_revenue
    data_tests:
      - unique_combination:
          arguments:
            columns: [order_date, region]

unit_tests:
  - name: latest_version_wins_and_cancelled_orders_drop_out
    model: fct_daily_revenue
    overrides:
      macros:
        is_incremental: false
    given:
      - input: ref('stg_orders')
        rows:
          - {order_id: 1, customer_id: 101, region: EU, amount: 120.0, status: placed,    created_at: "2026-10-14 09:00:00", updated_at: "2026-10-14 11:00:00"}
          - {order_id: 2, customer_id: 101, region: EU, amount: 30.0,  status: placed,    created_at: "2026-10-14 10:00:00", updated_at: "2026-10-14 10:00:00"}
          - {order_id: 3, customer_id: 102, region: EU, amount: 50.0,  status: cancelled, created_at: "2026-10-14 09:00:00", updated_at: "2026-10-14 12:00:00"}
          - {order_id: 4, customer_id: 201, region: US, amount: 80.0,  status: placed,    created_at: "2026-10-14 09:00:00", updated_at: "2026-10-14 09:00:00"}
    expect:
      rows:
        - {order_date: 2026-10-14, region: EU, orders: 2, revenue: 150.0, last_change: "2026-10-14 11:00:00"}
        - {order_date: 2026-10-14, region: US, orders: 1, revenue: 80.0, last_change: "2026-10-14 09:00:00"}
```

## A bug that passed every test

Three batches arrive. The third contains a **late cancellation**: order 3, created on 13 October, is cancelled on the 15th.
After each batch, `dbt build` ran the models and all tests, and then a script compared the incremental table with recomputing it
from scratch:

```text
$ python load_raw.py 1
batch 1 landed; raw.orders has 5 rows
$ dbt build --target local
Done. PASS=7 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=7
$ python load_raw.py 2
batch 2 landed; raw.orders has 7 rows
$ dbt build --target local
Done. PASS=7 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=7
$ python load_raw.py 3
batch 3 landed; raw.orders has 9 rows
$ dbt build --target local
Done. PASS=7 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=7
$ python compare.py
incremental: [('2026-10-13', 'EU', 3, 200.0), ('2026-10-13', 'US', 1, 80.0), ('2026-10-14', 'EU', 1, 40.0), ('2026-10-14', 'US', 1, 25.0), ('2026-10-15', 'EU', 1, 60.0)]
full build : [('2026-10-13', 'EU', 2, 150.0), ('2026-10-13', 'US', 1, 80.0), ('2026-10-14', 'EU', 1, 40.0), ('2026-10-14', 'US', 1, 25.0), ('2026-10-15', 'EU', 1, 60.0)]
MISMATCH: [(datetime.date(2026, 10, 13), 'EU', 2, 150.0), (datetime.date(2026, 10, 13), 'EU', 3, 200.0)]
```

Every test passed, and 13 October still shows three EU orders and 200 in revenue, including the cancelled order. The incremental
filter only reprocessed days from 14 October onwards, so a change to an older day never reached the mart. None of the standard
tests can see this: each row is unique, non-null and well-formed. It's simply stale. The unit test passed too, because it runs the
model in **full** mode (`is_incremental: false`), which is correct.

The test that catches it compares the incremental table with a rebuild, inside dbt.
`tests/assert_incremental_matches_full_rebuild.sql`:

```sql+jinja
-- fails (returns rows) if the incremental table differs from recomputing it from scratch
with rebuilt as (
    select cast(created_at as date) as order_date, region, count(*) as orders, sum(amount) as revenue
    from {{ ref('stg_orders') }}
    where status <> 'cancelled'
    group by 1, 2
),
loaded as (
    select order_date, region, orders, revenue from {{ ref('fct_daily_revenue') }}
)
select 'missing or wrong' as problem, * from (select * from rebuilt except select * from loaded)
union all
select 'unexpected' as problem, * from (select * from loaded except select * from rebuilt)
```

(The first version named its CTE `full`, which failed to parse: `FULL` is a reserved word in DuckDB and in Snowflake.) With it, the
same three batches fail where they should:

```text
$ python load_raw.py 3
batch 3 landed; raw.orders has 9 rows
$ dbt build --target local
7 of 8 FAIL 2 assert_incremental_matches_full_rebuild … [FAIL 2]
Done. PASS=7 WARN=0 ERROR=1 SKIP=0 NO-OP=0 REUSED=0 TOTAL=8
Got 2 results, configured to fail if != 0
[ERROR]: in test assert_incremental_matches_full_rebuild (tests/assert_incremental_matches_full_rebuild.sql)
```

```text
| problem          | order_date | region | orders | revenue |
| ---------------- | ---------- | ------ | ------ | ------- |
| missing or wrong | 2026-10-13 | EU     |      2 |     150 |
| unexpected       | 2026-10-13 | EU     |      3 |     200 |
```

The fix is to reprocess every day that has an order **changed** since the last load, however old the day is. The table records
`last_change` per row, so the model knows where it left off:

```sql+jinja
{{ config(materialized='incremental', unique_key=['order_date', 'region']) }}

select cast(created_at as date) as order_date, region,
       count(*) as orders, sum(amount) as revenue,
       max(updated_at) as last_change
from {{ ref('stg_orders') }}
where status <> 'cancelled'
{% if is_incremental() %}
  -- reprocess every day that has an order changed since the last load, however old the day is
  and cast(created_at as date) in (
      select distinct cast(created_at as date)
      from {{ ref('stg_orders') }}
      where updated_at > (select max(last_change) from {{ this }})
  )
{% endif %}
group by 1, 2
```

The same three batches from an empty database:

```text
$ python load_raw.py 1
batch 1 landed; raw.orders has 5 rows
$ dbt build --target local
Done. PASS=8 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=8
$ python load_raw.py 2
batch 2 landed; raw.orders has 7 rows
$ dbt build --target local
Done. PASS=8 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=8
$ python load_raw.py 3
batch 3 landed; raw.orders has 9 rows
$ dbt build --target local
Done. PASS=8 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=8
$ python compare.py
incremental: [('2026-10-13', 'EU', 2, 150.0), ('2026-10-13', 'US', 1, 80.0), ('2026-10-14', 'EU', 1, 40.0), ('2026-10-14', 'US', 1, 25.0), ('2026-10-15', 'EU', 1, 60.0)]
full build : [('2026-10-13', 'EU', 2, 150.0), ('2026-10-13', 'US', 1, 80.0), ('2026-10-14', 'EU', 1, 40.0), ('2026-10-14', 'US', 1, 25.0), ('2026-10-15', 'EU', 1, 60.0)]
MATCH
```

And the portability check for the Snowflake targets:

```text
7 compiled files (2 models, 5 data tests) parse as Snowflake SQL with sqlglot 30.22.0
```

Two lessons here. **Incremental models need a test against a full rebuild**, run in CI on realistic data, because their bugs are
staleness, which row-level tests can't see. And the incremental filter must be driven by **what changed** (a watermark on change
time), not by the business date you group by. One gap remains, deliberately left for the exercise: if every order of a day and region
is cancelled, the new run produces no row for it, and a merge on `unique_key` doesn't delete the old one.

## CI with zero-copy clones

The expensive part of testing data code is a realistic environment. In Snowflake, `CREATE DATABASE … CLONE` copies only metadata
(Level 1 · 09), so every pull request can get its own full copy of production in seconds, at no storage cost until it diverges. A
GitHub Actions workflow (`.github/workflows/dbt-ci.yml`):

```yaml
{{.github/workflows/dbt-ci.yml}}
```

The pull-request job authenticates with a **key pair** from repository secrets (a `SERVICE` user, Level 3 · 02), clones production,
runs `dbt build` in the clone (models, data tests, unit tests and the rebuild comparison), and **always** drops the clone, even when
the build fails. Two refinements for larger projects:

- **Slim CI.** `dbt build --select state:modified+ --defer --state <production artifacts>` builds only changed models and their
  dependents, and reads unchanged parents from production. Per Snowflake's documentation, dbt Projects on Snowflake can provide the
  production state directly.
- **A dedicated CI warehouse** with a resource monitor (lesson 05), so a runaway test can't spend more than you've decided.

## Deploying: dbt Projects on Snowflake

The deploy job uses **dbt Projects on Snowflake**: `snow dbt deploy` uploads the project as a **dbt project object** in the account,
and `snow dbt execute` runs a dbt command on it inside Snowflake, using a warehouse. Per the documentation, Snowflake provides managed
dbt Core (and dbt Fusion) runtimes, runs can be scheduled with **tasks**, and run history and logs are kept in the account.
Executions cost warehouse time, with no separate licence fee. Note the argument order in the workflow: everything after the
project name is passed to dbt itself, so Snowflake CLI options go before it.

## Beyond dbt: declarative DDL and Git

Not everything is a dbt model: roles, warehouses, policies, tasks. Two Snowflake features help manage those in Git too:

- **`CREATE OR ALTER`**: you state the full desired definition, and Snowflake works out the change, preserving data, grants, tags
  and policies. Re-running it is idempotent, which makes it safe in a deploy script. Per the documentation, a change that drops a
  column still drops its data (recoverable with Time Travel), so review the diff as carefully as any migration.
- **Git repository objects** and **`EXECUTE IMMEDIATE FROM`**: Snowflake can fetch a Git repository and run a SQL file from it (with
  optional Jinja templating), so the deploy step is `EXECUTE IMMEDIATE FROM @repo/branches/main/deploy/roles.sql`.

```sql
CREATE OR ALTER TABLE sales.mart.customers (
  customer_id NUMBER NOT NULL,
  email       VARCHAR,
  region      VARCHAR,
  created_at  TIMESTAMP_NTZ
);

CREATE OR ALTER WAREHOUSE transform_wh WAREHOUSE_SIZE = 'SMALL' AUTO_SUSPEND = 60;
```

## How It Actually Works

dbt is a compiler and a scheduler. It reads models (SQL with Jinja) and YAML, builds a dependency graph from `ref()` and `source()`,
renders each model into plain SQL for the target adapter, and runs them in dependency order, wrapping each in the DDL its
materialization needs: a view, `CREATE TABLE AS`, or for incremental models, a temporary result followed by a `MERGE` (Snowflake's
default) or delete-and-insert. Tests are just queries that must return zero rows. Unit tests render a model with fixed inputs in
place of its parents. Nothing in dbt knows whether your incremental filter is complete, which is why the rebuild comparison has to be
written by you.

The CI pattern works because of how Snowflake stores data. A clone is a new set of metadata pointing at the same immutable
micro-partitions, so it's instant and free. Writes in the clone create new partitions that only the clone references, and dropping
the clone discards them without touching production.

## Common mistakes

- **Incremental models tested only with row-level tests.** Compare with a full rebuild.
- **Filtering increments by business date** instead of change time. Late changes to old dates are lost.
- **CI against a shared dev database**, where runs collide and data drifts. Clone per pull request and drop afterwards.
- **Personal credentials in CI.** Use a `SERVICE` user with a key pair and a narrow role.
- **No resource monitor on the CI warehouse.**
- **Unreviewed `CREATE OR ALTER` diffs.** Dropping a column drops its data.

## Exercise

1. Run the project locally with dbt-duckdb (copy the files, run `load_raw.py` with batches 1–3, then `dbt build`). Then switch back
   to the first version of `fct_daily_revenue` and confirm the rebuild test fails.
2. Add batch 4: cancel order 6, the only EU order on 14 October. Does the fixed model leave a stale row? Change the model (for example,
   with `incremental_strategy` and a delete step, or by emitting zero rows for emptied days) until the rebuild test passes.
3. With a trial account, set up the pull-request job against a clone and check the time the clone took and its storage after the
   build in `TABLE_STORAGE_METRICS`.
4. Deploy the project with `snow dbt deploy` and schedule `snow dbt execute … build` as a Snowflake task. Where do you see the run's
   logs?
