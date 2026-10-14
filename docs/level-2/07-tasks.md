---
description: "Snowflake tasks: schedules and CRON, stream-gated and triggered tasks, serverless vs warehouse compute, task graphs with finalizers, and failure handling."
---

# 07 · Tasks & Task Graphs

A **task** runs a SQL statement (or a call to a stored procedure) on a schedule or when new data arrives, inside
Snowflake, with no external scheduler. Combined with streams (lesson 06), tasks give you incremental pipelines: a
stream collects changes, and a task merges them downstream. This lesson covers how tasks are scheduled, where their
compute comes from, how to chain them into graphs, and how to keep them from failing silently or burning credits doing
nothing.

## A first task

```sql
CREATE TASK IF NOT EXISTS learn_db.raw.merge_orders
  WAREHOUSE = etl_wh
  SCHEDULE = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('learn_db.raw.orders_changes')
AS
  MERGE INTO learn_db.analytics.orders_clean AS t
  USING (SELECT * FROM learn_db.raw.orders_changes
         WHERE NOT (METADATA$ACTION = 'DELETE' AND METADATA$ISUPDATE)) AS s
  ON t.order_id = s.order_id
  WHEN MATCHED AND s.METADATA$ACTION = 'DELETE' THEN DELETE
  WHEN MATCHED THEN UPDATE SET t.status = s.status, t.quantity = s.quantity, t.unit_price = s.unit_price
  WHEN NOT MATCHED AND s.METADATA$ACTION = 'INSERT' THEN INSERT (order_id, customer_id, ordered_at, channel, sku, quantity, unit_price, status)
       VALUES (s.order_id, s.customer_id, s.ordered_at, s.channel, s.sku, s.quantity, s.unit_price, s.status);

ALTER TASK learn_db.raw.merge_orders RESUME;     -- tasks are created SUSPENDED
EXECUTE TASK learn_db.raw.merge_orders;          -- run once now, e.g. to test
```

Three details:

- **Tasks are created suspended.** Nothing runs until `ALTER TASK … RESUME`, and forgetting that is the most common reason
  a "scheduled" pipeline never ran.
- **`SCHEDULE`** is either an interval (`'5 MINUTE'`, `'1 HOUR'`, and in seconds where supported) or a cron expression
  with a time zone: `'USING CRON 0 6 * * MON-FRI Europe/Lisbon'`. Always give a time zone in cron schedules.
- **`WHEN SYSTEM$STREAM_HAS_DATA(…)`** is evaluated by cloud services **before** starting compute. If the stream is
  empty, the run is skipped and no warehouse resumes. Without it, this task would wake `etl_wh` 288 times a day. At the
  60-second minimum on an X-Small that's 288 × 60 s = 4.8 hours, or 4.8 credits a day, to merge nothing most of the time.

## Triggered tasks: run when data arrives

Leave out the `SCHEDULE` and keep the `WHEN SYSTEM$STREAM_HAS_DATA(…)` condition, and the task becomes a **triggered task**
that runs when the stream has data, instead of polling on a timer:

```sql
CREATE TASK learn_db.raw.merge_orders_triggered
  TARGET_COMPLETION_INTERVAL = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('learn_db.raw.orders_changes')
AS
  INSERT INTO learn_db.analytics.order_events (order_id, action, seen_at)
  SELECT order_id, METADATA$ACTION, CURRENT_TIMESTAMP() FROM learn_db.raw.orders_changes;
```

This one has no `WAREHOUSE`, so it's also **serverless**, and for serverless triggered tasks Snowflake requires a
`TARGET_COMPLETION_INTERVAL`. `USER_TASK_MINIMUM_TRIGGER_INTERVAL_IN_SECONDS` limits how often a triggered task may fire.

## Where the compute comes from

| | User-managed warehouse | Serverless |
|---|---|---|
| Declared by | `WAREHOUSE = etl_wh` | omitting `WAREHOUSE` (optionally `USER_TASK_MANAGED_INITIAL_WAREHOUSE_SIZE`) |
| Sizing | you choose | Snowflake predicts from recent runs, up to the equivalent of XXLARGE |
| Billing | warehouse time, shared with whatever else runs there | per-second serverless compute at the serverless-task rate |
| Privilege needed | `USAGE` on the warehouse | the account-level `EXECUTE MANAGED TASK` |

Serverless suits short, frequent tasks, because it avoids a warehouse's 60-second minimum and idle tail.
`TARGET_COMPLETION_INTERVAL` (and `SERVERLESS_TASK_MIN/MAX_STATEMENT_SIZE`) let you tell Snowflake how fast it needs to
finish. Warehouse tasks suit heavy jobs that already have a warehouse running, or that need predictable sizing.

## Task graphs

Tasks can depend on other tasks with `AFTER`, forming a **graph** (DAG) with one **root** task that has the schedule:

```sql
CREATE TASK learn_db.raw.load_root
  WAREHOUSE = etl_wh
  SCHEDULE = 'USING CRON 0 * * * * UTC'
AS
  COPY INTO learn_db.raw.orders FROM @learn_db.raw.landing/orders/
  FILE_FORMAT = (FORMAT_NAME = 'learn_db.raw.csv_gz') ON_ERROR = 'CONTINUE';

CREATE TASK learn_db.raw.build_daily  WAREHOUSE = etl_wh AFTER learn_db.raw.load_root
AS
  INSERT OVERWRITE INTO learn_db.analytics.daily_revenue
  SELECT TO_DATE(ordered_at), channel, SUM(quantity * unit_price)
  FROM learn_db.raw.orders WHERE status = 'paid' GROUP BY ALL;

CREATE TASK learn_db.raw.build_customers WAREHOUSE = etl_wh AFTER learn_db.raw.load_root
AS
  INSERT OVERWRITE INTO learn_db.analytics.customer_totals
  SELECT customer_id, COUNT(*), SUM(quantity * unit_price)
  FROM learn_db.raw.orders WHERE status = 'paid' GROUP BY customer_id;

CREATE TASK learn_db.raw.notify_done WAREHOUSE = etl_wh
  FINALIZE = learn_db.raw.load_root
AS
  CALL SYSTEM$SEND_EMAIL('my_email_int', 'data-team@example.com', 'Hourly load finished', 'See TASK_HISTORY for details.');

-- resume the root and all its dependents in one go
SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('learn_db.raw.load_root');
```

- `build_daily` and `build_customers` both run **after** the root, in parallel. A task can also list several
  predecessors (`AFTER a, b`), and then it waits for all of them.
- A **finalizer** task (`FINALIZE = <root>`) runs once at the end of every graph run, **whether or not** the other
  tasks succeeded. Use it for notifications and cleanup.
- Child tasks can't have their own schedule. Only the root is scheduled.
- `SYSTEM$SEND_EMAIL` needs a notification integration (`my_email_int` here) that an administrator has created.

## Failures, retries, overlap

```sql
ALTER TASK learn_db.raw.load_root SET
  SUSPEND_TASK_AFTER_NUM_FAILURES = 3      -- stop retrying a broken graph forever
  TASK_AUTO_RETRY_ATTEMPTS = 2             -- retry a failed graph run automatically
  USER_TASK_TIMEOUT_MS = 1800000           -- 30-minute cap per run
  OVERLAP_POLICY = NO_OVERLAP;             -- don't start a run while the previous one is still running
```

`OVERLAP_POLICY` (`NO_OVERLAP`, `ALLOW_CHILD_OVERLAP`, `ALLOW_ALL_OVERLAP`) replaces the deprecated
`ALLOW_OVERLAPPING_EXECUTION`. `NO_OVERLAP` matters when a slow hourly run would otherwise race the next one. An
`ERROR_INTEGRATION` sends failure notifications to a cloud messaging service.

Check what happened:

```sql
SELECT name, state, scheduled_time, completed_time, error_code, error_message, query_id
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
       SCHEDULED_TIME_RANGE_START => DATEADD('day', -1, CURRENT_TIMESTAMP()),
       TASK_NAME => 'MERGE_ORDERS'))
ORDER BY scheduled_time DESC;
```

`state` values include `SUCCEEDED`, `FAILED`, `SKIPPED` (the `WHEN` condition was false, or a run was still going under
`NO_OVERLAP`) and `CANCELLED`. A task graph that shows only `SKIPPED` for days may be correctly idle, or fed by a source
that broke upstream. Alert on `FAILED` and on "no `SUCCEEDED` in N hours".

## How It Actually Works

The task scheduler lives in cloud services. At each scheduled time (or, for triggered tasks, when change tracking reports
new data in the stream), it evaluates the task's `WHEN` condition. `SYSTEM$STREAM_HAS_DATA` is answered from metadata,
without a warehouse. If the condition holds, the scheduler starts a **task run**: it resumes the task's warehouse (or
allocates serverless compute sized from the history of previous runs) and executes the body **as the task's owner
role** (or as a specified user with `EXECUTE AS USER`). That's why privileges belong to the role that owns the task,
not to whoever created or resumed it.

A graph run is identified by one root run. When the root succeeds, each child whose predecessors have all succeeded is
queued. Children of a failed task don't run. The finalizer runs after everything else ends. Each run uses the task
definitions as they were when the run started, so changing a task mid-run affects the next run. A stream consumed by a
task's DML is advanced only if that run's transaction commits, so a failed run leaves the changes in the stream for the
next attempt. This is what makes stream-plus-task pipelines safe to retry.

## Common mistakes

- **Never resuming the task** (or resuming only the root without the children). Use `SYSTEM$TASK_DEPENDENTS_ENABLE`.
- **No `WHEN` condition** on stream-driven tasks. The warehouse wakes to do nothing hundreds of times a day.
- **Cron schedules without a time zone**, which shift by an hour at daylight-saving changes.
- **No failure alerting.** A failed task just sits in `TASK_HISTORY`.
- **Overlapping runs** of slow tasks. Set `OVERLAP_POLICY = NO_OVERLAP`.
- **Giving the task owner more privileges than the job needs.** Tasks run with the owner's role.
- **Long-running heavy work as a serverless task without a size cap.** Set `SERVERLESS_TASK_MAX_STATEMENT_SIZE`.

## Exercise

1. Create the `merge_orders` task, resume it, insert some orders, and use `TASK_HISTORY` to find a `SKIPPED` run and a
   `SUCCEEDED` one.
2. Rebuild it as a serverless triggered task with a 5-minute target completion interval. Compare the credits used per day
   (`SERVERLESS_TASK_HISTORY` in `ACCOUNT_USAGE`) with the warehouse version.
3. Build the three-task graph plus a finalizer, then make `build_daily` fail (reference a table that doesn't exist). What
   runs, what doesn't, and what does the finalizer see?
4. Write the alert query: every task with no successful run in the last 6 hours, but at least one run attempt.
