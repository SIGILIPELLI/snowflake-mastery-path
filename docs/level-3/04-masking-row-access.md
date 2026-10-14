---
description: "Snowflake masking and row access policies: CURRENT_ROLE vs IS_ROLE_IN_SESSION modelled, admins who inherit PII, mapping tables and policy testing."
---

# 04 · Masking & Row Access Policies

Role-based access control (lesson 01) works at the level of whole objects: a role can read a table or it can't. Real
requirements are finer. Support staff may see customers' emails but analysts see them masked; the EU team sees EU
customers and nobody else's; a fraud investigator needs full card numbers that everyone else sees as `****1234`.
Snowflake implements these with **policies** that are attached to tables and evaluated on every query: **masking
policies** for columns and **row access policies** for rows. Both require **Enterprise Edition** or higher.

## A masking policy

A masking policy is a function from the column value (and, optionally, other columns) to what the current session may
see:

```sql
USE ROLE SECURITYADMIN;
CREATE SCHEMA IF NOT EXISTS security_db.policies;

CREATE OR REPLACE MASKING POLICY security_db.policies.mask_email
  AS (val VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('PII_READER') THEN val
    WHEN val IS NULL THEN NULL
    ELSE '***@' || SPLIT_PART(val, '@', 2)
  END;

ALTER TABLE sales.mart.customers MODIFY COLUMN email SET MASKING POLICY security_db.policies.mask_email;
```

- The policy's input and output types must match the column's type.
- A column has at most one masking policy, but one policy can protect any number of columns.
- The policy applies **wherever the column is read**: in `SELECT`s, through views, in joins, in `CREATE TABLE AS`. A
  user without `PII_READER` who copies the table copies masked values.
- Keep policies in a dedicated schema owned by a security role, so table owners can't drop or change them.

## CURRENT_ROLE() or IS_ROLE_IN_SESSION()?

The condition in the policy decides who sees what, and the obvious way to write it is subtly wrong. `CURRENT_ROLE() =
'PII_READER'` is true only when that exact role is the session's **primary** role. `IS_ROLE_IN_SESSION('PII_READER')` is
true when the role is the primary role, **inherited** through the hierarchy, or active as a secondary role. Here's the
difference for a typical hierarchy, computed with the RBAC model from lesson 01 (Python, locally; the model ignores
secondary roles):

```python
"""Which masking-policy condition shows PII to whom? Compares CURRENT_ROLE() = 'PII_READER'
with IS_ROLE_IN_SESSION('PII_READER') (role hierarchy aware), using the RBAC model from lesson 01."""
from rbac import Account
acct = Account()
acct.run("""
CREATE ROLE pii_reader; CREATE ROLE analyst; CREATE ROLE data_engineer; CREATE ROLE support;
GRANT ROLE pii_reader TO ROLE support;
GRANT ROLE analyst TO ROLE data_engineer;
GRANT ROLE pii_reader TO ROLE data_engineer;
GRANT ROLE analyst TO ROLE sysadmin; GRANT ROLE data_engineer TO ROLE sysadmin; GRANT ROLE support TO ROLE sysadmin;
""")
def mask(email): return "***@" + email.split("@")[1]
email = "ana.lima@example.com"
print(f"{'session primary role':22} {'CURRENT_ROLE() check':24} {'IS_ROLE_IN_SESSION check':24}")
for role in ["PII_READER", "SUPPORT", "DATA_ENGINEER", "ANALYST", "SYSADMIN", "ACCOUNTADMIN"]:
    current = email if role == "PII_READER" else mask(email)
    in_session = email if "PII_READER" in acct.inherited(role) else mask(email)
    print(f"{role:22} {current:24} {in_session:24}")
```

```text
session primary role   CURRENT_ROLE() check     IS_ROLE_IN_SESSION check
PII_READER             ana.lima@example.com     ana.lima@example.com    
SUPPORT                ***@example.com          ana.lima@example.com    
DATA_ENGINEER          ***@example.com          ana.lima@example.com    
ANALYST                ***@example.com          ***@example.com         
SYSADMIN               ***@example.com          ana.lima@example.com    
ACCOUNTADMIN           ***@example.com          ana.lima@example.com
```

- With `CURRENT_ROLE()`, **only** a session using `PII_READER` as its primary role sees emails. `SUPPORT`, which was
  granted `PII_READER` precisely so support staff could see them, gets masked values. That's the most common masking-policy
  bug report.
- With `IS_ROLE_IN_SESSION()`, inheritance works as intended for `SUPPORT` and `DATA_ENGINEER`, and `ANALYST` stays masked.
  But **`SYSADMIN` and `ACCOUNTADMIN` see the raw data too**, because every custom role rolls up to them (lesson 01's
  recommended hierarchy).

The second point is a design decision, not a bug. If administrators must **not** see PII, express that explicitly, for example by
granting PII roles outside the `SYSADMIN` branch and managing them from a dedicated security role. Whatever you choose,
write it down and test it.

## Conditional masking: using another column

Policies can take extra arguments from other columns in the same row:

```sql
CREATE OR REPLACE MASKING POLICY security_db.policies.mask_email_unless_consented
  AS (val VARCHAR, marketing_consent BOOLEAN) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('PII_READER') THEN val
    WHEN IS_ROLE_IN_SESSION('MARKETING') AND marketing_consent THEN val
    ELSE '***@' || SPLIT_PART(val, '@', 2)
  END;

ALTER TABLE sales.mart.customers MODIFY COLUMN email
  SET MASKING POLICY security_db.policies.mask_email_unless_consented USING (email, marketing_consent);
```

## Row access policies

A row access policy returns `TRUE` for rows the session may see. Scaling it beyond a couple of roles needs a **mapping
table** that says which roles see which regions:

```sql
CREATE TABLE IF NOT EXISTS security_db.policies.region_access (role_name VARCHAR, region VARCHAR);
INSERT INTO security_db.policies.region_access VALUES
  ('SALES_EU', 'EU'), ('SALES_US', 'US'), ('SALES_GLOBAL', 'EU'), ('SALES_GLOBAL', 'US'), ('SALES_GLOBAL', 'APAC');

CREATE OR REPLACE ROW ACCESS POLICY security_db.policies.region_rows
  AS (row_region VARCHAR) RETURNS BOOLEAN ->
  IS_ROLE_IN_SESSION('DATA_ENGINEER')
  OR EXISTS (
       SELECT 1 FROM security_db.policies.region_access m
       WHERE IS_ROLE_IN_SESSION(m.role_name) AND m.region = row_region
     );

ALTER TABLE sales.mart.orders ADD ROW ACCESS POLICY security_db.policies.region_rows ON (region);
```

- Granting a new team access is an `INSERT` into the mapping table, with no policy change.
- A table can have one row access policy, but the policy can take several columns.
- Snowflake's documentation recommends keeping the mapping table in the same database as the protected table, and for
  performance, considering a **memoizable function** in place of the mapping table lookup, and testing with realistic
  workloads. The policy runs as part of every query.
- Rows a session can't see behave as if they don't exist: `COUNT(*)`, `SUM` and joins all see the filtered table. Two
  analysts with different roles get different, consistent answers to the same query.

## Testing policies without switching users

`EXECUTE USING POLICY_CONTEXT` runs a query **as if** the session had a given role or set of roles, so you can check
policies from one administrative session:

```sql
EXECUTE USING POLICY_CONTEXT(CURRENT_ROLE => 'ANALYST')
  AS SELECT email FROM sales.mart.customers LIMIT 5;

EXECUTE USING POLICY_CONTEXT(SNOWFLAKE$SESSION_ACTIVATED_ROLES => ('SUPPORT', 'PII_READER', 'PUBLIC'))
  AS SELECT email FROM sales.mart.customers LIMIT 5;
```

Make this part of deployment: for each policy, a small set of `POLICY_CONTEXT` queries with the expected visible or masked
result per role. It's the SQL counterpart of the Python model above.

## Where policies are attached

```sql
SELECT * FROM TABLE(INFORMATION_SCHEMA.POLICY_REFERENCES(POLICY_NAME => 'security_db.policies.mask_email'));
SELECT * FROM TABLE(INFORMATION_SCHEMA.POLICY_REFERENCES(REF_ENTITY_NAME => 'sales.mart.customers', REF_ENTITY_DOMAIN => 'table'));
```

Attaching policies column by column doesn't scale to hundreds of tables. The next lesson attaches them to **tags**, so
every column tagged `pii=email` is masked automatically.

## How It Actually Works

Policies are schema-level objects evaluated by the query compiler. When a query references a protected column, the compiler
**rewrites** the query so that the column's every occurrence is wrapped in the policy expression, with the session's
context (current role, activated roles, user) available to functions like `IS_ROLE_IN_SESSION`. A row access policy
becomes an extra predicate applied to the table before anything else touches it. Because the rewrite happens below views
and joins, there's no way to read around a policy by querying through a view or copying the data. The masked values are
what the warehouse ever computes with.

This also explains the performance advice. The policy body runs for every row (or is joined into the plan). A mapping
table lookup per row is a join, so policies that call subqueries can slow large scans, and memoizable functions let
Snowflake cache the lookup result per query.

## Common mistakes

- **`CURRENT_ROLE() = 'X'`** when inherited or secondary roles should count. Use `IS_ROLE_IN_SESSION`.
- **Forgetting that administrators inherit everything** under the `SYSADMIN` hierarchy.
- **Policy owned by the same role that owns the table.** The table owner could detach it. Separate duties.
- **Policy type mismatches** (a `VARCHAR` policy on a `VARIANT` column).
- **Mapping tables nobody owns.** They become the real access-control list, so treat them that way.
- **Never testing** with `POLICY_CONTEXT`.
- **Assuming masked data is safe to copy into an unprotected table.** It is, because it's masked, but now the team can never
  get the real values from that copy.

## Exercise

1. Implement `mask_email` with `CURRENT_ROLE()` first, then with `IS_ROLE_IN_SESSION()`, and run the same
   `POLICY_CONTEXT` queries for `SUPPORT` and `DATA_ENGINEER`. Reproduce the model's table in your account.
2. Write a masking policy for a card-number column that shows the last four digits to `SUPPORT` and the full number only to
   `FRAUD_INVESTIGATOR`.
3. Add the region row access policy to the course's orders table (add a `region` column derived from country), and prove with
   `POLICY_CONTEXT` that `SALES_EU` sees only EU rows and `SALES_GLOBAL` sees all.
4. Decide whether `SYSADMIN` should see PII in your organization. Then change the role design so the model's output matches
   that decision.
