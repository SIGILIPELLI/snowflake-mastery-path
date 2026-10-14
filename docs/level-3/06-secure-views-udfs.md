---
description: "Snowflake secure views and functions: how a normal view can leak hidden rows via the optimizer, what SECURE changes and costs, and per-user rows."
---

# 06 · Secure Views & Secure Functions

A view is the classic way to expose part of a table: some columns, some rows. It looks like an access-control boundary,
but by default a view in Snowflake is **not** a security boundary. The optimizer is free to rearrange a query through a
view, and the view's SQL text is visible to people who can see the view. **Secure views** (and secure functions) close both
gaps, at a price in performance. They're also the default for sharing data with other accounts (next lesson).

## How a normal view can leak

A view that exposes only the sales team's salaries:

```sql
CREATE OR REPLACE VIEW hr.mart.sales_salaries AS
SELECT employee, salary FROM hr.raw.salaries WHERE team = 'sales';
```

A user who may query only this view wants to know whether *anyone* in the company earns over 400,000. They can't see
those rows, but they can try to make the database **evaluate their own expression on them**:

```sql
SELECT employee
FROM hr.mart.sales_salaries
WHERE 1 / IFF(salary > 400000, 0, 1) = 1;      -- divides by zero only for salaries above 400,000
```

If the optimizer evaluates the user's predicate **before** the view's `team = 'sales'` filter (pushing predicates down and
reordering them is a normal optimization), a hidden row with a salary over 400,000 triggers a division-by-zero error. The
error itself answers the question. Snowflake's documentation describes exactly this risk with a "widgets" example: for a
non-secure view, internal optimizations that need access to the base table's data "might allow data that is hidden from
users of the view to be exposed through user code, such as user-defined functions, or other programmatic methods."

Whether the leak happens depends on the plan the optimizer happens to choose. DuckDB 1.5.6 run locally (not Snowflake),
with the same table, view and a probe using `error()`:

```sql
-- DuckDB 1.5.6 (local), not Snowflake
SELECT employee FROM sales_salaries
WHERE CASE WHEN salary > 400000 THEN error('someone earns over 400k') ELSE TRUE END;
```

```text
[('ana',), ('ben',)]

SEQ_SCAN … Filters: team='sales'   →   FILTER: CASE WHEN ((salary > 400000)) THEN … error(…) …
```

Here the plan applied the view's filter inside the scan, before the probe, so nothing leaked. That's the point: **with a
normal view, safety is an accident of the plan**, and a different engine, version or data distribution can produce a
different order. A secure view makes the ordering a guarantee instead of an accident.

## Secure views

```sql
CREATE OR REPLACE SECURE VIEW hr.mart.sales_salaries AS
SELECT employee, salary FROM hr.raw.salaries WHERE team = 'sales';

ALTER VIEW hr.mart.some_old_view SET SECURE;
```

`SECURE` changes two things, per the documentation:

1. **No leaking optimizations.** Snowflake doesn't use the internal optimizations that could expose base-table data through
   user code. The view's own logic is applied before user predicates can see the rows.
2. **The definition is hidden** from users who aren't the owner, in `SHOW VIEWS`, `GET_DDL`, the information schema, and
   even the Query Profile. The internals of a secure view aren't shown there **even to its owner**, because non-owners might
   have access to the owner's Query Profile.

The cost: secure views "can execute more slowly than non-secure views", because the optimizer has fewer options. So use
`SECURE` where a view is an access boundary or exposes data to another account, and plain views where it's just a
convenience for people who could read the base tables anyway.

```sql
SELECT table_catalog, table_schema, table_name, is_secure
FROM hr.INFORMATION_SCHEMA.VIEWS
WHERE table_schema = 'MART';
```

## Filtering by who's asking

Secure views can use context functions to show each user or role only their rows. That's row-level security without a
row access policy, and common before policies existed:

```sql
CREATE OR REPLACE SECURE VIEW sales.mart.my_region_orders AS
SELECT o.*
FROM sales.mart.orders o
JOIN security_db.policies.region_access m
  ON m.region = o.region
 AND IS_ROLE_IN_SESSION(m.role_name);
```

For new designs, prefer a **row access policy** (lesson 04): it's attached to the table itself, so every access path is
filtered, not just this view. Secure views remain the tool for **shaping** data (joins, aggregates, renamed columns) for a
particular audience, especially another account.

## Secure functions

UDFs have the same two issues. A normal UDF's body is visible to anyone who can describe it, and the optimizer may evaluate
it in ways that expose inputs. `SECURE` fixes both:

```sql
CREATE OR REPLACE SECURE FUNCTION finance.mart.discount_for(customer_tier VARCHAR, order_total NUMBER(12,2))
RETURNS NUMBER(12,2)
AS
$$
  CASE customer_tier
    WHEN 'gold'     THEN order_total * 0.12          -- pricing logic you don't want to publish
    WHEN 'platinum' THEN order_total * 0.18
    ELSE 0
  END
$$;
```

Use it when the function's **logic** is confidential (pricing, scoring), or when it's shared with other accounts.

## How It Actually Works

A view is stored as a query that the compiler inlines into every query that uses it. Once inlined, a normal view is
just part of one big query, and the optimizer may push the user's filters down, reorder predicates, evaluate expressions
early, or use the view's base-table statistics. All of these are good for speed, and some can evaluate user-supplied
expressions on rows the view would have removed. For a secure view, the compiler keeps the view as an **opaque boundary**.
The view's filters and projections are applied first, and only their output reaches operators that evaluate user
expressions. That rules out the reordering and the optimizations that depend on it, which is the performance cost. Hiding
the definition is a separate, simpler rule: metadata commands and the Query Profile redact the view text and internals for
anyone but the owner (and not even the owner in the profile).

## Common mistakes

- **Treating a normal view as an access boundary.** Its safety depends on the plan. Use `SECURE` (or a policy on the table).
- **Making every view secure.** You pay in performance where no boundary exists.
- **Secure views that use `CURRENT_ROLE()`** where role inheritance should count. Use `IS_ROLE_IN_SESSION`, as in lesson 04.
- **Expecting to debug a secure view in the Query Profile.** Its internals are hidden there. Test the defining query separately.
- **Sharing non-secure views without thinking.** Shares accept only secure views unless created with
  `SECURE_OBJECTS_ONLY = FALSE`. That option exists for performance, and it hands the protections above to the consumer.

## Exercise

1. Create the salaries table and the non-secure view in a trial account, run the division-by-zero probe, and check the Query
   Profile for where the filters were applied. Repeat with a `SECURE` view.
2. As a role that doesn't own the view, run `SHOW VIEWS` and `SELECT GET_DDL('VIEW', …)` for both versions. What can you see?
3. Measure a heavier query (an aggregation over `SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS`) through a normal and a secure view
   with the result cache off. Is there a difference for your query?
4. Rewrite `my_region_orders` as a row access policy on the base table, and list the access paths the policy covers that the
   view didn't.
