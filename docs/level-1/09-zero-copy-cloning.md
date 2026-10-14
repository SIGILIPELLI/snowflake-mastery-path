---
description: "Zero-copy cloning of tables, schemas and databases for dev and test, what is and isn't cloned, and when a clone stops being free: scattered updates."
---

# 09 · Zero-Copy Cloning & Environments

`CREATE … CLONE` makes a full, independent, writable copy of a table, schema or entire database in seconds, regardless of
size, without copying any data. It's one of the features that most changes how teams work: every developer can have a
copy of production to test against, every CI run can get a fresh database, and every risky migration can be rehearsed
on a clone first. This lesson covers how to use it, the rules about what gets cloned, and the case where a clone
quietly stops being free.

## Cloning

```sql
-- a table
CREATE TABLE learn_db.raw.orders_dev CLONE learn_db.raw.orders;

-- a whole database, for a developer
CREATE DATABASE learn_db_dev_ana CLONE learn_db;

-- a database as it was yesterday at 06:00 (Time Travel + clone)
CREATE DATABASE learn_db_audit CLONE learn_db
  AT(TIMESTAMP => '2026-10-12 06:00:00'::TIMESTAMP_LTZ);

-- keep the grants of the source table (all except OWNERSHIP)
CREATE TABLE learn_db.raw.orders_copy CLONE learn_db.raw.orders COPY GRANTS;
```

The clone is immediately usable and fully independent: changes to it don't affect the source, and changes to the
source after the clone don't appear in it. Combining `CLONE` with `AT`/`BEFORE` gives you a point-in-time copy, which
is the standard way to rehearse a recovery (lesson 08) or reproduce yesterday's report exactly.

## What comes along, and what doesn't

From Snowflake's cloning documentation:

| Object in a cloned database/schema | Behaviour |
|---|---|
| Tables, views, sequences, file formats | cloned |
| Grants on child objects | cloned; grants on the **database/schema container itself are not** |
| External named stages | cloned (they point at the same bucket) |
| Internal named stages | cloned only with `INCLUDE INTERNAL STAGES`; table stages are cloned **empty** |
| Pipes | pipes on external stages are cloned; pipes on internal stages are not |
| Streams | cloned, but **unconsumed records in them are inaccessible** in the clone |
| Tasks, alerts | cloned **suspended** |
| Tables with automatic clustering | clustering is suspended on the clone (`ALTER TABLE … RESUME RECLUSTER`) |

Two of these matter in practice:

- **Tasks start suspended**, which is what you want. A cloned dev database must not start running production jobs. But it
  also means a cloned pipeline doesn't "just run": resume what you need deliberately.
- **External stages and pipes still point at production storage.** A cloned pipe on an external stage, if resumed, loads
  production's files into your dev tables. Review and repoint these before resuming anything.

## A dev/test workflow

```sql
-- every morning (or per pull request), a fresh copy of production for development
CREATE OR REPLACE DATABASE analytics_dev CLONE analytics_prod;
GRANT USAGE ON DATABASE analytics_dev TO ROLE developer;           -- container grants aren't cloned

-- a migration rehearsal
CREATE OR REPLACE DATABASE analytics_migration_test CLONE analytics_prod;
-- run the migration scripts against analytics_migration_test, then run the tests, then:
DROP DATABASE analytics_migration_test;
```

Pair it with masking policies (Level 3 · 04) so developers on a clone of production don't see personal data they
shouldn't. Policies are attached to columns and travel with the clone.

## When a clone stops being free

A clone shares the source's micro-partitions. Storage is only consumed for micro-partitions that one side **rewrites**
after the clone. Since micro-partitions are immutable, changing even one row in a micro-partition rewrites the whole
micro-partition. So the question is how many micro-partitions a change touches, and that depends on how the changed rows
are **distributed**, not on how many there are.

This model (Python, run locally, not a Snowflake measurement) takes a ~1 TB table of 1,000,000,000 rows in 20,000
micro-partitions of 50,000 rows, and computes how much of it a single `UPDATE` in the clone rewrites:

```python
P, ROWS_PER_P, GB_PER_P = 20_000, 50_000, 0.05
def partitions_touched(rows_updated, scattered):
    if scattered:      # updated rows spread uniformly at random over the table
        return P * (1 - (1 - 1 / P) ** rows_updated)
    return min(P, -(-rows_updated // ROWS_PER_P))     # updated rows packed together (e.g. one recent date)
```

```text
  rows updated % of table | clustered: parts      GB | scattered: parts      GB
        10,000     0.001% |                1     0.1 |            7,870   393.5
       100,000     0.010% |                2     0.1 |           19,865   993.3
     1,000,000     0.100% |               20     1.0 |           20,000  1000.0
    10,000,000     1.000% |              200    10.0 |           20,000  1000.0
```

Updating **0.001%** of rows, if they're scattered across the table (say "set `is_test = TRUE` for these 10,000 customer
IDs"), rewrites about **39% of the table**. Updating 0.01% rewrites essentially all of it. The clone now owns close to a
full copy, and so does the source's Time Travel history. The same 10,000 rows packed together (one day's orders, when
the table is ordered by date) touch a single micro-partition.

The same arithmetic governs every `UPDATE`, `DELETE` and `MERGE` in Snowflake, clone or not. It's the cost of the change,
the Time Travel storage it generates, and the work automatic clustering has to redo. Level 2 · 01–02 builds on it:
keeping related rows together in micro-partitions is what makes both queries and changes cheap.

## How It Actually Works

A table's identity in the metadata is, essentially, a list of micro-partition files. `CLONE` creates a new metadata
entry whose list points at **the same files**, which is why it takes the same time for 1 KB and 100 TB. From then on
the two tables evolve independently. A write to either one creates new files referenced only by that table, and the
retired files remain referenced by the other table's current version (and by Time Travel). A file is deleted only when
nothing references it anymore. That includes no table version within its retention, and no clone.

That's also why storage accounting can surprise you. If you drop the *source* table while a clone still uses its files,
those files don't disappear. They remain, now billed to the table that still needs them. `TABLE_STORAGE_METRICS` reports
this through its `RETAINED_FOR_CLONE_BYTES` column.

Cloning a database or schema clones each child object the same way, plus the container metadata. Objects that point
outside Snowflake (external stages, pipes on them) are cloned as pointers to the same external resources, which is why
they need review.

## Common mistakes

- **Forgetting container grants.** The cloned database has none of the source database's grants.
- **Resuming cloned pipes or tasks without repointing them.** Dev loads production files, or dev jobs write somewhere
  shared.
- **Scattered updates on a clone** (anonymizing a column in place, flagging random rows), which turns a free clone into a
  full copy. Prefer views, or masking policies for anonymization.
- **Keeping old clones forever.** As the source changes, the clone holds onto partitions the source has retired. Drop
  clones you no longer need.
- **Expecting stream contents to carry over.** Unconsumed stream records aren't accessible in the clone.

## Exercise

1. Clone `learn_db` to `learn_db_dev`, then compare `SHOW GRANTS ON DATABASE` for both, and `SHOW GRANTS ON TABLE` for a
   table in each.
2. Create a clone of `raw.orders`, update 1,000 random rows in it (`WHERE UNIFORM(0, 199, RANDOM()) = 0` approximates
   0.5%), and check the clone's `ACTIVE_BYTES` in `TABLE_STORAGE_METRICS` the next day. With only four small files'
   worth of data, how many micro-partitions does the table even have? (`SYSTEM$CLUSTERING_INFORMATION` reports the
   total.)
3. Extend the model: the 10,000 scattered rows are updated **every day for 30 days** in the source table (not a clone),
   with 30-day Time Travel. How much Time Travel storage results?
4. Write a script (SQL or Snowflake CLI) that recreates a developer database from production every morning and grants it
   to the developer role, and run it twice. Does the second run disturb anything?
