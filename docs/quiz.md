---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Snowflake Mastery Path with a 20-question final quiz."
---

# Final Quiz

20 questions drawn from all four levels of this course. You need **70% (14 out of 20)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this return?\n```sql\ncreate table t(id int, grp text, v int); insert into t values (1,'a',10),(2,'a',20),(3,'b',5);\n\nselect grp, sum(v), avg(v)::int, count(*) from t group by grp order by grp;\n```\n",
    "options": [
      "a,30,15,3; b,5,5,1",
      "a,10,15,2; b,5,5,1",
      "a,30,20,2; b,5,5,1",
      "a,30,15,2; b,5,5,1"
    ],
    "correctIndex": 3,
    "explanation": "Grouped aggregates; the same in Snowflake. Verified by running the standard-SQL part on PostgreSQL 18 (rows comma-separated, separated by semicolons); Snowflake itself was not available, so Snowflake-only syntax is not run here."
  },
  {
    "question": "What does this window query return?\n```sql\ncreate table t(d int, v int); insert into t values (1,10),(2,20),(3,30);\n\nselect d, sum(v) over (order by d) running, lag(v) over (order by d) prev from t order by d;\n```\n",
    "options": [
      "1,10,; 2,20,10; 3,30,20",
      "1,10,; 2,30,10; 3,60,20",
      "1,10,10; 2,30,10; 3,60,20",
      "1,10,; 2,30,10; 3,60,"
    ],
    "correctIndex": 1,
    "explanation": "A running sum and the previous row's value (NULL first). Verified by running the standard-SQL part on PostgreSQL 18 (rows comma-separated, separated by semicolons); Snowflake itself was not available, so Snowflake-only syntax is not run here."
  },
  {
    "question": "What does this NULL-handling query return?\n```sql\ncreate table t(x int); insert into t values (1),(null),(3);\n\nselect count(*), count(x), sum(x), coalesce(max(x),0), avg(x)::numeric(3,1) from t;\n```\n",
    "options": [
      "2,2,4,3,2.0",
      "3,3,4,3,1.3",
      "3,2,4,3,1.3",
      "3,2,4,3,2.0"
    ],
    "correctIndex": 3,
    "explanation": "NULLs are ignored by aggregates; count(*) counts rows. Verified by running the standard-SQL part on PostgreSQL 18 (rows comma-separated, separated by semicolons); Snowflake itself was not available, so Snowflake-only syntax is not run here."
  },
  {
    "question": "A table is 500 micro-partitions and your filter touches 20. What does pruning mean for cost?",
    "options": [
      "All 500 are scanned",
      "Only the matching partitions are scanned, so the query reads far less data",
      "Nothing is scanned",
      "It cannot be known"
    ],
    "correctIndex": 1,
    "explanation": "Min/max metadata excludes the rest."
  },
  {
    "question": "Why is auto-suspend a main cost control?",
    "options": [
      "Queries speed up",
      "Storage stops",
      "Data is deleted",
      "Compute credits accrue only while a warehouse runs, so idle warehouses should suspend"
    ],
    "correctIndex": 3,
    "explanation": "Per-second billing with a 60-second minimum on resume."
  },
  {
    "question": "An XL warehouse (16 credits/hour) runs 15 minutes. How many credits (per-second billing)?\n```python\nprint(16*0.25)\n```\n",
    "options": [
      "4.0",
      "16.0",
      "8.0",
      "1.0"
    ],
    "correctIndex": 0,
    "explanation": "Rate x hours. Verified by running it."
  },
  {
    "question": "Which cache can answer an identical repeated query without using a warehouse?",
    "options": [
      "Metadata cache only",
      "Local disk cache",
      "The result cache (valid for 24 hours if data is unchanged)",
      "Stage cache"
    ],
    "correctIndex": 2,
    "explanation": "Cloud services return the stored result."
  },
  {
    "question": "What is the difference between Time Travel and Fail-safe?",
    "options": [
      "Time Travel is user-accessible recovery within the retention period; Fail-safe is a 7-day last-resort period handled by Snowflake support",
      "Fail-safe is self-service",
      "No difference",
      "Time Travel is 7 days always"
    ],
    "correctIndex": 0,
    "explanation": "Fail-safe is not configurable or for routine recovery."
  },
  {
    "question": "Why is a zero-copy clone cheap?",
    "options": [
      "It compresses data",
      "It copies lazily on read",
      "It is a view",
      "It shares existing micro-partitions and stores only changes made afterwards"
    ],
    "correctIndex": 3,
    "explanation": "Ideal for dev/test."
  },
  {
    "question": "A stream shows an updated row as which pair of records?",
    "options": [
      "One UPDATE row",
      "A DELETE and an INSERT with METADATA$ISUPDATE = TRUE",
      "Two inserts",
      "One insert"
    ],
    "correctIndex": 1,
    "explanation": "Consume streams in DML to advance the offset."
  },
  {
    "question": "Which object keeps a result continuously refreshed within a freshness goal?",
    "options": [
      "A clone",
      "A dynamic table with TARGET_LAG",
      "A view",
      "A stage"
    ],
    "correctIndex": 1,
    "explanation": "Declarative pipelines."
  },
  {
    "question": "Why prefer role-based masking policies over duplicate masked tables?",
    "options": [
      "Required by SQL",
      "Cheaper compute",
      "Faster loads",
      "One governed table avoids drift and extra storage while each role sees what it is allowed to"
    ],
    "correctIndex": 3,
    "explanation": "Policies attach to columns."
  },
  {
    "question": "Why are secure views used for data sharing?",
    "options": [
      "They hide definitions and prevent optimiser side-channel leakage of data consumers shouldn't see",
      "Faster",
      "Cheaper",
      "Encrypted at rest only"
    ],
    "correctIndex": 0,
    "explanation": "A requirement for shares."
  },
  {
    "question": "What should a service account use for authentication?",
    "options": [
      "Key-pair (or OAuth) with a network policy, not a shared password",
      "None",
      "Shared password",
      "Admin role"
    ],
    "correctIndex": 0,
    "explanation": "Rotate keys."
  },
  {
    "question": "Where do you audit which columns were accessed by whom?",
    "options": [
      "Roles",
      "ACCOUNT_USAGE.ACCESS_HISTORY (with QUERY_HISTORY)",
      "Tasks",
      "Stages"
    ],
    "correctIndex": 1,
    "explanation": "Views have some latency."
  },
  {
    "question": "When should you scale a warehouse out (multi-cluster) instead of up?",
    "options": [
      "For loads",
      "Always",
      "Never",
      "When many concurrent queries queue, not when a single query is slow"
    ],
    "correctIndex": 3,
    "explanation": "Scale up for single heavy queries."
  },
  {
    "question": "What does Snowpark pushdown mean for large DataFrames?",
    "options": [
      "Operations are translated to SQL and run in Snowflake instead of pulling data to the client",
      "Data is copied locally",
      "Nothing is pushed",
      "Queries run in the browser"
    ],
    "correctIndex": 0,
    "explanation": "Avoid collect() on big tables."
  },
  {
    "question": "How do you make an incremental load safe to re-run?",
    "options": [
      "Run it once",
      "Ignore errors",
      "Make it idempotent with MERGE on a key so duplicates and re-processing change nothing",
      "Truncate daily"
    ],
    "correctIndex": 2,
    "explanation": "Verify with a second run."
  },
  {
    "question": "What is the right use of Cortex or LLM functions in a pipeline?",
    "options": [
      "With governance, cost limits and validation of outputs, since results can be wrong",
      "Trust blindly",
      "Skip review",
      "Run on all rows"
    ],
    "correctIndex": 0,
    "explanation": "Sample and monitor."
  },
  {
    "question": "What is the safest migration cutover approach?",
    "options": [
      "Big bang",
      "Delete old first",
      "Skip tests",
      "Run old and new in parallel, reconcile results and cut over workload by workload"
    ],
    "correctIndex": 3,
    "explanation": "Validate with row counts and checksums."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Sales analytics with window functions

Write three analytics queries over a small sales table: monthly revenue with a running total and share of region, the best rep per region and the month-over-month change. These run on PostgreSQL here because Snowflake is not available to me; the SQL is standard, and where Snowflake offers QUALIFY the comments say so.

**Starter code**

```sql
create table sales(id int, region text, rep text, sold_on date, amount numeric);
insert into sales values
 (1,'north','ana','2026-01-05',120),(2,'north','bo','2026-01-09',80),(3,'north','ana','2026-02-02',200),
 (4,'south','cy','2026-01-07',150),(5,'south','di','2026-01-20',90),(6,'south','cy','2026-02-11',60),
 (7,'north','bo','2026-02-15',140),(8,'south','di','2026-02-28',300);

-- TODO 1: monthly revenue per region (to_char(sold_on,'YYYY-MM')) with a running total per region and each month's % of the region total (1 decimal); order by region, month
-- TODO 2: the best rep per region by total amount (ties by rep name) - Snowflake would use QUALIFY; here use a subquery with row_number()
-- TODO 3: month-over-month change in revenue per region using lag(); the first month's change is NULL (printed as empty)
```

**Expected output**

```text
north | 2026-01 | 200 | 200 | 37.0
north | 2026-02 | 340 | 540 | 63.0
south | 2026-01 | 240 | 240 | 40.0
south | 2026-02 | 360 | 600 | 60.0
north | ana | 320
south | di | 390
north | 2026-01 | 200 | 
north | 2026-02 | 340 | 140
south | 2026-01 | 240 | 
south | 2026-02 | 360 | 120
```

<details><summary>Show solution and explanation</summary>

<pre><code>create table sales(id int, region text, rep text, sold_on date, amount numeric);
insert into sales values
 (1,&#x27;north&#x27;,&#x27;ana&#x27;,&#x27;2026-01-05&#x27;,120),(2,&#x27;north&#x27;,&#x27;bo&#x27;,&#x27;2026-01-09&#x27;,80),(3,&#x27;north&#x27;,&#x27;ana&#x27;,&#x27;2026-02-02&#x27;,200),
 (4,&#x27;south&#x27;,&#x27;cy&#x27;,&#x27;2026-01-07&#x27;,150),(5,&#x27;south&#x27;,&#x27;di&#x27;,&#x27;2026-01-20&#x27;,90),(6,&#x27;south&#x27;,&#x27;cy&#x27;,&#x27;2026-02-11&#x27;,60),
 (7,&#x27;north&#x27;,&#x27;bo&#x27;,&#x27;2026-02-15&#x27;,140),(8,&#x27;south&#x27;,&#x27;di&#x27;,&#x27;2026-02-28&#x27;,300);

-- 1. monthly revenue per region with running total and share of the region&#x27;s total
select region, to_char(sold_on,&#x27;YYYY-MM&#x27;) as month, sum(amount) as revenue,
       sum(sum(amount)) over (partition by region order by to_char(sold_on,&#x27;YYYY-MM&#x27;)) as running,
       round(100 * sum(amount) / sum(sum(amount)) over (partition by region), 1) as pct_of_region
from sales group by region, to_char(sold_on,&#x27;YYYY-MM&#x27;)
order by region, month;

-- 2. best rep per region (the Snowflake way would be QUALIFY row_number() = 1)
select region, rep, total from (
  select region, rep, sum(amount) as total,
         row_number() over (partition by region order by sum(amount) desc, rep) as rn
  from sales group by region, rep
) x where rn = 1 order by region;

-- 3. month-over-month change for each region
select region, month, revenue, revenue - lag(revenue) over (partition by region order by month) as change from (
  select region, to_char(sold_on,&#x27;YYYY-MM&#x27;) as month, sum(amount) as revenue from sales group by 1, 2
) m order by region, month;
</code></pre>

<p>Window functions can wrap aggregates (sum(sum(amount)) over ...), and row_number() in a subquery is exactly what QUALIFY expresses in Snowflake. lag() gives the previous month, NULL for the first. Verified by running it on PostgreSQL 18; not run on Snowflake.</p>
</details>

### An idempotent change-capture upsert

Simulate a stream plus MERGE: find the new and changed staging rows, apply them to the target only when values really differ and prove a second run finds nothing left to do. PostgreSQL's ON CONFLICT stands in for Snowflake's MERGE; Snowflake itself was not used.

**Starter code**

```sql
create table target(id int primary key, name text, plan text, updated date);
insert into target values (1,'ana','free','2026-01-01'),(2,'bo','pro','2026-01-01'),(3,'cy','free','2026-01-01');
create table staging(id int, name text, plan text, updated date);
insert into staging values (2,'bo','free','2026-02-01'),(3,'cy','free','2026-02-01'),(4,'di','pro','2026-02-01');

-- TODO 1: list the changes a stream would report: staging rows that are new ("insert") or whose (name, plan) differ from target ("update"), ordered by id
-- TODO 2: apply them idempotently with INSERT ... ON CONFLICT (Snowflake: MERGE), changing a row only when (name, plan) really differ, then show target ordered by id
-- TODO 3: count staging rows that still differ from target (should be 0 on a re-run)
```

**Expected output**

```text
2 | update
4 | insert
1 | ana | free | 2026-01-01
2 | bo | free | 2026-02-01
3 | cy | free | 2026-01-01
4 | di | pro | 2026-02-01
0
```

<details><summary>Show solution and explanation</summary>

<pre><code>create table target(id int primary key, name text, plan text, updated date);
insert into target values (1,&#x27;ana&#x27;,&#x27;free&#x27;,&#x27;2026-01-01&#x27;),(2,&#x27;bo&#x27;,&#x27;pro&#x27;,&#x27;2026-01-01&#x27;),(3,&#x27;cy&#x27;,&#x27;free&#x27;,&#x27;2026-01-01&#x27;);
create table staging(id int, name text, plan text, updated date);
insert into staging values (2,&#x27;bo&#x27;,&#x27;free&#x27;,&#x27;2026-02-01&#x27;),(3,&#x27;cy&#x27;,&#x27;free&#x27;,&#x27;2026-02-01&#x27;),(4,&#x27;di&#x27;,&#x27;pro&#x27;,&#x27;2026-02-01&#x27;);

-- 1. the changes a stream would report: new rows and rows whose values differ
select s.id, case when t.id is null then &#x27;insert&#x27; else &#x27;update&#x27; end as action
from staging s left join target t on t.id = s.id
where t.id is null or (t.name, t.plan) is distinct from (s.name, s.plan)
order by s.id;

-- 2. apply them idempotently (Snowflake: MERGE INTO ... WHEN MATCHED / NOT MATCHED)
insert into target select * from staging
on conflict (id) do update set name = excluded.name, plan = excluded.plan, updated = excluded.updated
where (target.name, target.plan) is distinct from (excluded.name, excluded.plan);

select * from target order by id;

-- 3. running it again changes nothing: the change set is now empty
select count(*) as pending from staging s join target t on t.id = s.id
where (t.name, t.plan) is distinct from (s.name, s.plan);
</code></pre>

<p>IS DISTINCT FROM compares row values null-safely, so unchanged rows are not rewritten and the pending count drops to zero. In Snowflake you would write MERGE INTO ... WHEN MATCHED AND (...) THEN UPDATE WHEN NOT MATCHED THEN INSERT. Verified by running it on PostgreSQL 18; not run on Snowflake.</p>
</details>
