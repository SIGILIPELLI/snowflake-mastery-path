---
description: "Auditing Snowflake with ACCOUNT_USAGE: latency, retention, viewer roles, login and grant audits, and ACCESS_HISTORY reads through views."
---

# 09 · Auditing with ACCOUNT_USAGE & Access History

Every control in this level, whether roles, authentication, network policies or masking, needs evidence that it works and
is used. Snowflake records the evidence in the **`SNOWFLAKE`** database, mainly in the **`ACCOUNT_USAGE`** schema: who
logged in from where, every query, every grant, and, with **Access History**, which columns each query actually read. This
lesson covers how to use those views, and the one detail that makes or breaks a data-access audit.

## ACCOUNT_USAGE vs INFORMATION_SCHEMA

From Snowflake's documentation:

| | `SNOWFLAKE.ACCOUNT_USAGE` | each database's `INFORMATION_SCHEMA` |
|---|---|---|
| Scope | the whole account | one database (plus account-level table functions) |
| Includes dropped objects | yes (with a `DELETED` timestamp) | no |
| Latency | **45 minutes to 3 hours**, varies by view | none |
| Retention of history | **1 year** | 7 days to 6 months, varies |

Use `INFORMATION_SCHEMA` for "what's happening right now" and `ACCOUNT_USAGE` for audits, trends and anything older than a
few days. For data you must keep longer than a year, copy the views into your own tables on a schedule (a task, Level 2 · 07).

## Who can read the audit views

Grant the `SNOWFLAKE` database's built-in **database roles**, rather than `ACCOUNTADMIN`, to auditors and platform teams:

| Database role | Gives visibility into |
|---|---|
| `SNOWFLAKE.OBJECT_VIEWER` | object metadata |
| `SNOWFLAKE.USAGE_VIEWER` | historical usage (queries, metering) |
| `SNOWFLAKE.GOVERNANCE_VIEWER` | governance data (policies, tags, access history) |
| `SNOWFLAKE.SECURITY_VIEWER` | security data (logins, users, grants) |

```sql
USE ROLE ACCOUNTADMIN;
CREATE ROLE IF NOT EXISTS auditor;
GRANT DATABASE ROLE SNOWFLAKE.GOVERNANCE_VIEWER TO ROLE auditor;
GRANT DATABASE ROLE SNOWFLAKE.SECURITY_VIEWER TO ROLE auditor;
GRANT USAGE ON WAREHOUSE bi_wh TO ROLE auditor;
```

## Sign-ins

```sql
SELECT event_timestamp, user_name, client_ip, reported_client_type,
       first_authentication_factor, second_authentication_factor, is_success, error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE event_timestamp > DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND is_success = 'NO'
ORDER BY event_timestamp DESC;
```

Useful audits from this view: failed sign-ins per user and IP (password spraying), successful password sign-ins without a
second factor (lesson 02's enforcement), and sign-ins from IPs outside your network rules (which shouldn't succeed if
network policies are in place).

## Grants and role changes

```sql
-- who holds ACCOUNTADMIN, directly
SELECT grantee_name AS user_name, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
WHERE role = 'ACCOUNTADMIN' AND deleted_on IS NULL;

-- privileges granted in the last 7 days, by whom
SELECT created_on, privilege, granted_on, name, grantee_name, granted_by
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE created_on > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY created_on DESC;
```

These are the inputs for an effective-access calculation like the `rbac.py` model in lesson 01: load them into its `Account`
instead of a hand-written script, and you get "who can read this table?" from the live account.

## Access History: who read which columns

**Access History** (Enterprise Edition) records, for each query, which objects and **columns** it read and wrote. The
`ACCESS_HISTORY` view has, among others, these columns per query: `query_id`, `query_start_time`, `user_name`,
`direct_objects_accessed`, `base_objects_accessed`, `objects_modified`, `policies_referenced`, and parent/root query IDs for
queries run from procedures.

The two "accessed" columns are the important distinction. From the documentation's view example:

- **`direct_objects_accessed`** lists the objects named **in the query**. For a query on a view, that's the view and its
  projected columns.
- **`base_objects_accessed`** lists the **underlying** tables and columns the data actually came from, including columns used
  only to filter.

An audit of "who read customers' emails?" that looks only at direct objects misses everyone who read them **through a view**.
To show the difference, here are four invented access-history records in the documented JSON shape, with the same audit
query run in DuckDB 1.5.6 locally. This is synthetic data, not Snowflake output:

```python
"""Synthetic ACCESS_HISTORY rows in the documented JSON shape (objectName / objectDomain / columns[columnName]),
and the 'who read customers.email?' audit, run in DuckDB. The rows are invented test data, not Snowflake output."""
import json, duckdb
def obj(name, cols, domain="Table"):
    return {"objectName": name, "objectDomain": domain, "columns": [{"columnName": c} for c in cols]}
rows = [
  ("q1", "2026-10-01 09:00", "ANA", [obj("SALES.MART.CUSTOMERS", ["CUSTOMER_ID", "COUNTRY"])],
                                    [obj("SALES.MART.CUSTOMERS", ["CUSTOMER_ID", "COUNTRY"])]),
  ("q2", "2026-10-02 10:30", "BEN", [obj("SALES.MART.CUSTOMERS", ["EMAIL"])],
                                    [obj("SALES.MART.CUSTOMERS", ["EMAIL"])]),
  # CY reads a VIEW that exposes email: the view is the direct object, the table column is the base object
  ("q3", "2026-10-03 14:00", "CY",  [obj("SALES.MART.CUSTOMER_CONTACTS", ["CONTACT_EMAIL"], "View")],
                                    [obj("SALES.MART.CUSTOMERS", ["EMAIL", "CUSTOMER_ID"])]),
  ("q4", "2026-10-04 08:15", "DEE", [obj("SALES.MART.ORDERS", ["ORDER_ID", "AMOUNT"])],
                                    [obj("SALES.MART.ORDERS", ["ORDER_ID", "AMOUNT"])]),
]
con = duckdb.connect()
con.sql("CREATE TABLE access_history (query_id VARCHAR, query_start_time TIMESTAMP, user_name VARCHAR, direct_objects_accessed JSON, base_objects_accessed JSON)")
con.executemany("INSERT INTO access_history VALUES (?,?,?,?,?)", [(q, t, u, json.dumps(d), json.dumps(b)) for q, t, u, d, b in rows])
for label, col in (("direct_objects_accessed", "direct_objects_accessed"), ("base_objects_accessed", "base_objects_accessed")):
    print(f"who read SALES.MART.CUSTOMERS.EMAIL, judged by {label}:")
    print(con.sql(f"""
      SELECT DISTINCT user_name, query_id
      FROM access_history,
           UNNEST(CAST(json_extract({col}, '$[*]') AS JSON[])) AS o(obj),
           UNNEST(CAST(json_extract(obj, '$.columns[*]') AS JSON[])) AS c(cj)
      WHERE (obj->>'objectName') = 'SALES.MART.CUSTOMERS' AND (cj->>'columnName') = 'EMAIL'
      ORDER BY query_id""").fetchall())
```

```text
who read SALES.MART.CUSTOMERS.EMAIL, judged by direct_objects_accessed:
[('BEN', 'q2')]
who read SALES.MART.CUSTOMERS.EMAIL, judged by base_objects_accessed:
[('BEN', 'q2'), ('CY', 'q3')]
```

Judged by `direct_objects_accessed`, only Ben read the email column. Judged by `base_objects_accessed`, **Cy** did too, by
querying `CUSTOMER_CONTACTS`, a view whose `CONTACT_EMAIL` column comes from `CUSTOMERS.EMAIL`. The Snowflake version of the
correct audit:

```sql
SELECT DISTINCT ah.user_name, ah.query_id, ah.query_start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY ah,
     LATERAL FLATTEN(input => ah.base_objects_accessed) obj,
     LATERAL FLATTEN(input => obj.value:"columns") col
WHERE obj.value:"objectName"::VARCHAR = 'SALES.MART.CUSTOMERS'
  AND col.value:"columnName"::VARCHAR = 'EMAIL'
  AND ah.query_start_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY ah.query_start_time DESC;
```

Note the double `FLATTEN` (objects, then columns) from Level 1 · 07. `policies_referenced` shows whether a masking or row
access policy was applied. Combine it with the query above to separate users who saw **masked** emails from those who saw raw
ones, which is usually the real question.

Access History also powers **lineage**: `objects_modified` records which columns a statement wrote and from which source
columns, so you can trace, say, where data derived from `EMAIL` ended up.

## Turning audits into alerts

Snowflake **alerts** run a condition on a schedule and act when it returns rows:

```sql
CREATE OR REPLACE ALERT security_db.network.too_many_failed_logins
  WAREHOUSE = bi_wh
  SCHEDULE = '60 MINUTE'
  IF (EXISTS (
    SELECT user_name FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
    WHERE is_success = 'NO' AND event_timestamp > DATEADD('hour', -2, CURRENT_TIMESTAMP())
    GROUP BY user_name HAVING COUNT(*) >= 10))
  THEN
    CALL SYSTEM$SEND_EMAIL('my_email_int', 'security@example.com',
                           'Snowflake: repeated failed sign-ins', 'Check LOGIN_HISTORY for the last 2 hours.');

ALTER ALERT security_db.network.too_many_failed_logins RESUME;
```

Because `ACCOUNT_USAGE` can lag by up to a few hours, set the alert's look-back window wider than its schedule (here 2
hours checked hourly), and accept that it's a detective control, not a real-time one.

## How It Actually Works

`ACCOUNT_USAGE` views are shared to every account from Snowflake through the `SNOWFLAKE` database, which is itself a share
(lesson 07). The data is produced by Snowflake's services as they run your account: the authentication service writes login
events, the query service writes query and access records, the metadata store emits grant and object changes. These records
are processed into the views asynchronously, which is the source of the latency. Access History's column-level detail comes
from the compiler: when it resolves a query, including expanding views and applying policies, it knows exactly which base
columns contribute to each output and filter, and it records that resolution alongside the query.

## Common mistakes

- **Auditing reads with `direct_objects_accessed` only.** It misses access through views.
- **Granting `ACCOUNTADMIN` to auditors.** Use the `SNOWFLAKE` database roles.
- **Real-time expectations** from `ACCOUNT_USAGE`. Latency is up to hours.
- **Relying on one year of retention** for multi-year obligations. Archive the views yourself.
- **Alerts whose look-back equals their schedule.** Late-arriving records are missed. Overlap the windows.
- **Not distinguishing masked from unmasked reads.** Use `policies_referenced`.

## Exercise

1. Create the `auditor` role with the two database roles and confirm it can query `LOGIN_HISTORY` and `ACCESS_HISTORY` but
   not your business tables.
2. Create a view over `customers` that exposes `email` under another name, query it as an analyst, wait for the latency, and
   find that read with the `base_objects_accessed` query.
3. Extend the audit to report, per user, whether the email values they saw were masked (`policies_referenced`).
4. Build a daily task that copies the previous day's `LOGIN_HISTORY` and `ACCESS_HISTORY` rows into an archive table, so you
   can keep them beyond one year.
