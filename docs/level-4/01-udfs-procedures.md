---
description: "User-defined functions vs stored procedures in Snowflake: SQL UDFs and table functions, Snowflake Scripting, transactions, and owner's vs caller's rights."
---

# 01 · UDFs & Stored Procedures

Sooner or later, the same expression appears in twenty queries, or a job needs several statements to run as one unit.
Snowflake has two tools for that, and choosing between them is the first decision. A **user-defined function (UDF)**
computes a value (or a table) **inside** a SQL statement. A **stored procedure** **runs** statements: it's invoked with
`CALL`, and can create, insert, delete and control flow. This lesson covers both in SQL, and the security question every
procedure raises: whose privileges does it run with?

## Function or procedure?

| | UDF | Stored procedure |
|---|---|---|
| Called | inside a statement: `SELECT net_amount(amount, 10) …` | on its own: `CALL archive_old_orders(90)` |
| Returns | one value per row (scalar) or a set of rows (table function) | one value (or a table), once |
| May run DML/DDL | no | yes |
| Typical use | reusable calculations, parsing, business rules | admin jobs, multi-step loads, anything with control flow |

Both can be written in SQL, Python, Java, Scala or JavaScript, per Snowflake's documentation (lesson 03 covers Python).

## SQL UDFs

A scalar SQL UDF is an expression with a name:

```sql
CREATE OR REPLACE FUNCTION sales.mart.net_amount(amount NUMBER(12,2), discount_pct NUMBER(5,2))
RETURNS NUMBER(12,2)
COMMENT = 'Order amount after a percentage discount'
AS
$$
  amount * (1 - discount_pct / 100)
$$;

SELECT order_id, amount, sales.mart.net_amount(amount, 10) AS net FROM sales.mart.orders LIMIT 5;
```

A **table function (UDTF)** returns rows, and is used in `FROM` with `TABLE(…)`:

```sql
CREATE OR REPLACE FUNCTION sales.mart.orders_for(cust NUMBER)
RETURNS TABLE (order_id NUMBER, order_date DATE, amount NUMBER(12,2))
AS
$$
  SELECT order_id, order_date, amount FROM sales.mart.orders WHERE customer_id = cust
$$;

SELECT * FROM TABLE(sales.mart.orders_for(101)) ORDER BY order_date;

-- and per row of another table, as a lateral join
SELECT c.customer_id, o.order_id, o.amount
FROM sales.mart.customers c, TABLE(sales.mart.orders_for(c.customer_id)) o;
```

Functions are **overloaded** by argument types: `net_amount(NUMBER, NUMBER)` and `net_amount(NUMBER)` are two different
functions, and `DROP FUNCTION` and `GRANT … ON FUNCTION` need the signature (`net_amount(NUMBER, NUMBER)`). To call a
function, a role needs `USAGE` on it (and on its database and schema).

## Stored procedures in Snowflake Scripting

**Snowflake Scripting** is Snowflake's procedural SQL: variables, `IF`, loops, cursors and exception handlers. A procedure that
moves orders older than a cutoff into an archive table must do two statements, and they must succeed or fail together:

```sql
CREATE OR REPLACE PROCEDURE ops.archive_old_orders(keep_days NUMBER)
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS OWNER
AS
$$
DECLARE
  cutoff DATE DEFAULT DATEADD(day, -keep_days, CURRENT_DATE());
  n_archived NUMBER;
  n_deleted NUMBER;
BEGIN
  BEGIN TRANSACTION;
  INSERT INTO sales.mart.orders_archive SELECT * FROM sales.mart.orders WHERE order_date < :cutoff;
  n_archived := SQLROWCOUNT;
  DELETE FROM sales.mart.orders WHERE order_date < :cutoff;
  n_deleted := SQLROWCOUNT;
  COMMIT;
  RETURN 'archived ' || n_archived || ', deleted ' || n_deleted;
EXCEPTION
  WHEN OTHER THEN
    ROLLBACK;
    RAISE;
END;
$$;

CALL ops.archive_old_orders(90);
```

Three details make it correct:

- **One cutoff, computed once.** Both statements use the same variable, so a row can't be copied by one and missed by
  the other because the date changed between them. (Inside SQL statements, variables are referenced with a colon: `:cutoff`.)
- **An explicit transaction.** Without it, a failure after the `INSERT` would leave rows in both tables. The handler rolls
  back and **re-raises**, so the caller (a task, an orchestrator) sees the failure instead of a cheerful return value.
- **It's safe to run twice.** The second call finds nothing older than the cutoff.

The same logic, written as a Python procedure and run with Snowpark's **local testing mode** (an in-memory emulation that
ships with the Snowpark library, no account involved), on ten orders that are 20, 40, … 200 days old:

```python
# Snowpark local testing (in-memory, no Snowflake account): the archive procedure's logic, called twice.
import datetime as dt
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col, lit, sproc, current_date, dateadd
from snowflake.snowpark.types import IntegerType, StringType

session = Session.builder.config("local_testing", True).create()
today = dt.date(2026, 10, 14)
rows = [(i, today - dt.timedelta(days=i * 20), 10.0 * i) for i in range(1, 11)]   # 20, 40, ... 200 days old
session.create_dataframe(rows, schema=["order_id", "order_date", "amount"]).write.save_as_table("orders")
session.create_dataframe([], schema=session.table("orders").schema).write.save_as_table("orders_archive")

@sproc(name="archive_old_orders", input_types=[IntegerType()], return_type=StringType(), session=session, replace=True)
def archive_old_orders(session, keep_days):
    cutoff = today - dt.timedelta(days=keep_days)
    orders = session.table("orders")
    old = orders.filter(col("order_date") < lit(cutoff))
    n = old.count()
    old.write.save_as_table("orders_archive", mode="append")        # 1. copy
    deleted = orders.delete(orders["order_date"] < lit(cutoff))     # 2. delete the same predicate
    return f"archived {n}, deleted {deleted.rows_deleted}"

print("1st call:", session.call("archive_old_orders", 90))
print("2nd call:", session.call("archive_old_orders", 90))
print("orders left:", session.table("orders").count(), "| archived:", session.table("orders_archive").count())
```

```text
1st call: archived 6, deleted 6
2nd call: archived 0, deleted 0
orders left: 4 | archived: 6
```

Six orders are older than 90 days. The second call archives nothing, which is the idempotence you want from anything a
scheduler might retry. Local testing has documented limits: it doesn't run raw SQL strings, and it ignores registration
options such as `execute_as`, so it can test your **logic**, but not rights or transactions. Those need a real account.

## Owner's rights vs caller's rights

This is the part of procedures that matters most for security. From Snowflake's documentation:

| | Owner's rights (`EXECUTE AS OWNER`, the **default**) | Caller's rights (`EXECUTE AS CALLER`) |
|---|---|---|
| Privileges used | the procedure **owner's** | the **caller's** |
| Database and schema | the ones the procedure was created in | the caller's current ones |
| Caller's session variables | can't read or set them | can read and set them |
| Caller's session parameters | only a documented subset can be used; can't set them | can read and set them |
| Warehouse | the caller's current warehouse | the caller's current warehouse |

Owner's rights are how you **delegate a narrow power safely**. The `archive_old_orders` procedure above can be owned by
a role that may delete from `orders`, while support engineers get only `USAGE` on the procedure. They can archive, with
the exact logic in the procedure, but they can't run an arbitrary `DELETE`. That only stays safe if the procedure can't
be steered into doing something else. An owner's rights procedure that builds SQL from its arguments
(`EXECUTE IMMEDIATE 'DELETE FROM ' || table_arg`) hands the owner's privileges to anyone who can call it. Bind values
(`:cutoff`) instead of concatenating them, and never take object names from callers without validating them against a
fixed list.

Caller's rights suit **utility** procedures that should do only what the caller could do anyway: a helper that runs
a sequence of statements in the caller's schema, or one that needs the caller's session variables. Snowflake also offers
**restricted caller's rights**, where an administrator limits which of the caller's privileges a caller's rights procedure
may use.

```sql
GRANT USAGE ON PROCEDURE ops.archive_old_orders(NUMBER) TO ROLE support_engineer;   -- note the signature
ALTER PROCEDURE ops.archive_old_orders(NUMBER) EXECUTE AS CALLER;                   -- switching modes later is possible
```

## How It Actually Works

A SQL UDF is stored as an expression (or a query, for a table function), and it's evaluated as part of the query that calls
it, row by row or as a row source. That's why a UDF can't change data or control a transaction: it's an expression, not a
program. A **secure** function (Level 3 · 06) additionally hides its body and limits the optimizations applied around it.

A stored procedure is a program. `CALL` starts it in the cloud services layer, which runs its statements one by one, each
as a normal query on the caller's current warehouse. Variables and control flow live in the procedure's execution
context. Each statement it issues is compiled with the privileges of the owner or the caller, depending on the mode, and
appears as a separate query in query history, next to the `CALL` itself. Transactions inside a procedure follow the usual
rules: without `BEGIN TRANSACTION`, each DML statement commits on its own (autocommit).

## Common mistakes

- **Procedures where a UDF would do** (and vice versa). If it computes a value inside a query, it's a function.
- **Multi-statement changes without a transaction**, leaving half-finished work after a failure.
- **Swallowing exceptions.** A handler that returns `'error'` makes a failed job look successful to the scheduler. Re-raise.
- **Building SQL by concatenating arguments in owner's rights procedures.** That's SQL injection with the owner's privileges.
- **Forgetting the signature** in `GRANT`, `DROP` and `ALTER` on overloaded functions and procedures.
- **Assuming the caller's context** in an owner's rights procedure: its database, schema and session variables aren't the
  caller's.

## Exercise

1. Create `net_amount` and `orders_for` in a trial account, then find them in `INFORMATION_SCHEMA.FUNCTIONS`. Use `orders_for`
   in a lateral join for five customers.
2. Create `archive_old_orders` and make it fail halfway (for example, drop the archive table's grant). Confirm that `orders`
   is unchanged afterwards and that the `CALL` reports an error.
3. Grant `USAGE` on the procedure to a role that can't delete from `orders`, and call it as that role. Then switch the
   procedure to caller's rights and call it again. Explain both results.
4. Extend `archive_local.py` into a pytest file with three tests: the cutoff boundary (an order exactly 90 days old), an
   empty table, and two consecutive calls.
