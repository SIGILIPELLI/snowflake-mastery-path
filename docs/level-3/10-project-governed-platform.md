---
description: "Level 3 project: a governed two-team Snowflake platform with tested role grants, tag-based masking, region row access, network rules and a PII alert."
---

# 10 · Project — A Governed Multi-Team Platform

This project puts Level 3 together. Two teams (sales and HR), a support desk and an auditor share one account. The
requirements are the kind a security review writes down:

1. Analysts read curated marts only. Engineers also read raw data and write marts. Nobody crosses teams.
2. Customer emails and phones are masked for everyone except support agents. Salaries are visible only to HR analysts.
3. **Administrators don't see raw PII or salaries** just because they're administrators.
4. Each role sees order rows only for the regions it's entitled to.
5. Service users connect only from known networks.
6. An auditor can review access history and grants without being able to read business data.
7. If someone later grants a PII role to the wrong person, the security team hears about it the next morning.

What makes this a project rather than a pile of SQL is the last step: **the access design is tested**. Requirements 1–3 are
checked by 20 automated tests against the grant script, and those tests found three real mistakes in the design while it
was being written for this lesson.

!!! note "What was run"
    The grants were executed by the course's RBAC model (`rbac.py`, from lesson 01), which computes effective access through
    the role hierarchy, and checked with pytest. All runs shown are real output. The policy, network and alert SQL was
    syntax-checked and verified against Snowflake's documentation, but not executed on Snowflake. `policy_tests.sql` states
    what each query **must** return, for you to run in a trial account.

## The design

```text
                     ACCOUNTADMIN
                     /          \
              SECURITYADMIN    SYSADMIN                         PII_STEWARD   (priya; outside the admin tree)
                    |          /      \                          /        \
                 AUDITOR  SALES_ENG   HR_ENG               SUPPORT_AGENT   SALARY_READER
                   (otto)   (ben)    |                        (sam)           (hana, as a secondary role)
                            |      HR_ANALYST (hana)            |
                     SALES_ANALYST (ana)                     PII_READER
```

Functional roles (who you are) receive access roles (what you can touch: `sales_mart_read`, `sales_mart_write`,
`hr_mart_read`, and so on), as in lesson 01. Everything that can see raw PII hangs under `PII_STEWARD`, a role owned by a
named data-protection owner and **deliberately not granted to SYSADMIN**. That is the whole trick behind requirement 3:
in Snowflake every privilege flows upward through role grants, so the only way to keep a role's power away from administrators
is to keep the role out of their hierarchy.

## Step 1 — Roles and grants

`grants.sql`:

```sql
-- 1. access roles: (domain, layer, level)
USE ROLE USERADMIN;
CREATE ROLE IF NOT EXISTS sales_raw_read;   CREATE ROLE IF NOT EXISTS sales_mart_read;  CREATE ROLE IF NOT EXISTS sales_mart_write;
CREATE ROLE IF NOT EXISTS hr_mart_read;     CREATE ROLE IF NOT EXISTS hr_mart_write;
CREATE ROLE IF NOT EXISTS pii_reader;       CREATE ROLE IF NOT EXISTS salary_reader;
-- 2. functional roles
CREATE ROLE IF NOT EXISTS sales_analyst;    CREATE ROLE IF NOT EXISTS sales_engineer;   CREATE ROLE IF NOT EXISTS hr_engineer;
CREATE ROLE IF NOT EXISTS hr_analyst;       CREATE ROLE IF NOT EXISTS support_agent;  CREATE ROLE IF NOT EXISTS auditor;
CREATE ROLE IF NOT EXISTS pii_steward;      -- owns and manages every role that can see raw PII

USE ROLE SECURITYADMIN;
GRANT USAGE ON DATABASE sales TO ROLE sales_mart_read;
GRANT USAGE ON SCHEMA sales.mart TO ROLE sales_mart_read;
GRANT SELECT ON ALL TABLES IN SCHEMA sales.mart TO ROLE sales_mart_read;
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales.mart TO ROLE sales_mart_read;
GRANT INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA sales.mart TO ROLE sales_mart_write;
GRANT ROLE sales_mart_read TO ROLE sales_mart_write;
GRANT USAGE ON DATABASE sales TO ROLE sales_raw_read;
GRANT USAGE ON SCHEMA sales.raw TO ROLE sales_raw_read;
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales.raw TO ROLE sales_raw_read;
GRANT USAGE ON DATABASE hr TO ROLE hr_mart_read;
GRANT USAGE ON SCHEMA hr.mart TO ROLE hr_mart_read;
GRANT SELECT ON FUTURE TABLES IN SCHEMA hr.mart TO ROLE hr_mart_read;
GRANT INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA hr.mart TO ROLE hr_mart_write;
GRANT ROLE hr_mart_read TO ROLE hr_mart_write;

GRANT ROLE sales_mart_read TO ROLE sales_analyst;   GRANT USAGE ON WAREHOUSE sales_bi_wh TO ROLE sales_analyst;
GRANT ROLE sales_analyst TO ROLE sales_engineer;
GRANT ROLE sales_mart_write TO ROLE sales_engineer; GRANT ROLE sales_raw_read TO ROLE sales_engineer;
GRANT USAGE ON WAREHOUSE sales_etl_wh TO ROLE sales_engineer;
GRANT ROLE hr_mart_read TO ROLE hr_analyst;         GRANT USAGE ON WAREHOUSE hr_bi_wh TO ROLE hr_analyst;
GRANT ROLE hr_analyst TO ROLE hr_engineer;          GRANT ROLE hr_mart_write TO ROLE hr_engineer;
GRANT ROLE sales_mart_read TO ROLE support_agent;   GRANT ROLE pii_reader TO ROLE support_agent;
GRANT USAGE ON WAREHOUSE support_wh TO ROLE support_agent;

-- 3. hierarchy: functional roles under SYSADMIN, EXCEPT roles that can see raw PII: those sit under pii_steward,
--    which is deliberately outside the admin hierarchy so administrators don't inherit PII access (lesson 04)
GRANT ROLE sales_engineer TO ROLE SYSADMIN;  GRANT ROLE hr_engineer TO ROLE SYSADMIN;
GRANT ROLE support_agent TO ROLE pii_steward;  GRANT ROLE salary_reader TO ROLE pii_steward;
GRANT ROLE auditor TO ROLE SECURITYADMIN;

-- 4. people
GRANT ROLE sales_analyst TO USER ana;   GRANT ROLE sales_engineer TO USER ben;
GRANT ROLE hr_analyst TO USER hana;     GRANT ROLE salary_reader TO USER hana;   -- used as a secondary role
GRANT ROLE support_agent TO USER sam;   GRANT ROLE auditor TO USER otto;
GRANT ROLE pii_steward TO USER priya;   -- a named data-protection owner, not an administrator
```

## Step 2 — The access matrix as tests

The requirements become a table of `(user, privilege, table, allowed?)` cases plus three structural rules. `test_access.py`:

```python
"""The access matrix the platform promises, checked against grants.sql with the RBAC model.
Run with: pytest -q test_access.py"""
import pytest
from rbac import Account

acct = Account(); acct.run(open("grants.sql").read())
SALES_MART, SALES_RAW, HR_MART = "SALES.MART.ORDERS", "SALES.RAW.ORDERS", "HR.MART.SALARIES"

EXPECTED = [  # (user, privilege, table, allowed?)
    ("ANA",  "SELECT", SALES_MART, True),  ("ANA",  "SELECT", SALES_RAW, False), ("ANA",  "SELECT", HR_MART, False),
    ("ANA",  "UPDATE", SALES_MART, False),
    ("BEN",  "SELECT", SALES_MART, True),  ("BEN",  "SELECT", SALES_RAW, True),  ("BEN",  "UPDATE", SALES_MART, True),
    ("BEN",  "SELECT", HR_MART, False),
    ("HANA", "SELECT", HR_MART, True),     ("HANA", "SELECT", SALES_MART, False),("HANA", "UPDATE", HR_MART, False),
    ("SAM",  "SELECT", SALES_MART, True),  ("SAM",  "UPDATE", SALES_MART, False),("SAM",  "SELECT", HR_MART, False),
    ("OTTO", "SELECT", SALES_MART, False), ("OTTO", "SELECT", HR_MART, False),
]

@pytest.mark.parametrize("user,priv,table,allowed", EXPECTED)
def test_access_matrix(user, priv, table, allowed):
    assert (user in acct.who_can(priv, "TABLE", table)) == allowed

def test_only_the_pii_branch_is_outside_the_admin_hierarchy():
    assert acct.orphans() == ["PII_READER", "PII_STEWARD", "SALARY_READER", "SUPPORT_AGENT"]

def test_only_support_inherits_pii_reader():
    holders = sorted(r for r in acct.roles if "PII_READER" in acct.inherited(r) and r != "PII_READER")
    assert holders == ["PII_STEWARD", "SUPPORT_AGENT"]

def test_admins_do_not_inherit_pii_reader():
    for admin in ("SYSADMIN", "ACCOUNTADMIN", "SECURITYADMIN"):
        assert "PII_READER" not in acct.inherited(admin)

def test_admins_do_not_inherit_salary_access():
    # the salary masking policy unmasks for SALARY_READER, so no administrator may inherit it
    assert "SALARY_READER" in acct.roles
    for admin in ("SYSADMIN", "ACCOUNTADMIN", "SECURITYADMIN"):
        assert "SALARY_READER" not in acct.inherited(admin)
```

### What the tests caught

The first version of the grant script granted `support_agent` and `pii_steward` to SYSADMIN like every other functional
role, and `hr_mart_write` was created but never granted to anyone. The first run:

```text
FAILED test_access.py::test_no_orphan_roles_except_pii_reader - AssertionErro...
FAILED test_access.py::test_only_support_inherits_pii_reader - AssertionError...
FAILED test_access.py::test_admins_do_not_inherit_pii_reader - AssertionError...
3 failed, 16 passed in 0.02s
```

(The orphan test was renamed after the fix to say what it now checks.) Two design faults, both easy to miss by reading the script: **every administrator inherited raw PII** (through
`SYSADMIN → SUPPORT_AGENT → PII_READER`), and an access role was orphaned, so HR engineers couldn't write their own mart.
Moving the PII branch under `PII_STEWARD` and granting `hr_mart_write` to `hr_engineer` fixed both.

The third fault appeared while writing the salary masking policy. The obvious rule, "unmask for `HR_ANALYST`", fails
requirement 3, because `HR_ANALYST` sits under `HR_ENGINEER`, which sits under SYSADMIN. Asking the model confirmed it
(`'HR_ANALYST' in acct.inherited('SYSADMIN')` returned `True`). So the test came first, and failed:

```text
test_access.py:36: AssertionError
=========================== short test summary info ============================
FAILED test_access.py::test_admins_do_not_inherit_salary_access - AssertionEr...
1 failed, 19 passed in 0.02s
```

The fix: a dedicated `SALARY_READER` role under `PII_STEWARD`, granted to hana directly as a user (she activates it as a
**secondary role**), and a masking policy keyed on that role. The final run:

```text
test_access.py::test_access_matrix[ANA-SELECT-SALES.MART.ORDERS-True] PASSED [  5%]
test_access.py::test_access_matrix[ANA-SELECT-SALES.RAW.ORDERS-False] PASSED [ 10%]
test_access.py::test_access_matrix[ANA-SELECT-HR.MART.SALARIES-False] PASSED [ 15%]
test_access.py::test_access_matrix[ANA-UPDATE-SALES.MART.ORDERS-False] PASSED [ 20%]
test_access.py::test_access_matrix[BEN-SELECT-SALES.MART.ORDERS-True] PASSED [ 25%]
test_access.py::test_access_matrix[BEN-SELECT-SALES.RAW.ORDERS-True] PASSED [ 30%]
test_access.py::test_access_matrix[BEN-UPDATE-SALES.MART.ORDERS-True] PASSED [ 35%]
test_access.py::test_access_matrix[BEN-SELECT-HR.MART.SALARIES-False] PASSED [ 40%]
test_access.py::test_access_matrix[HANA-SELECT-HR.MART.SALARIES-True] PASSED [ 45%]
test_access.py::test_access_matrix[HANA-SELECT-SALES.MART.ORDERS-False] PASSED [ 50%]
test_access.py::test_access_matrix[HANA-UPDATE-HR.MART.SALARIES-False] PASSED [ 55%]
test_access.py::test_access_matrix[SAM-SELECT-SALES.MART.ORDERS-True] PASSED [ 60%]
test_access.py::test_access_matrix[SAM-UPDATE-SALES.MART.ORDERS-False] PASSED [ 65%]
test_access.py::test_access_matrix[SAM-SELECT-HR.MART.SALARIES-False] PASSED [ 70%]
test_access.py::test_access_matrix[OTTO-SELECT-SALES.MART.ORDERS-False] PASSED [ 75%]
test_access.py::test_access_matrix[OTTO-SELECT-HR.MART.SALARIES-False] PASSED [ 80%]
test_access.py::test_only_the_pii_branch_is_outside_the_admin_hierarchy PASSED [ 85%]
test_access.py::test_only_support_inherits_pii_reader PASSED             [ 90%]
test_access.py::test_admins_do_not_inherit_pii_reader PASSED             [ 95%]
test_access.py::test_admins_do_not_inherit_salary_access PASSED          [100%]
============================== 20 passed in 0.03s ==============================
```

The lesson generalises: **any policy that checks `IS_ROLE_IN_SESSION('X')` grants its exception to every role above `X`.**
Check the hierarchy above every role you name in a policy.

## Step 3 — Policies, network and audit

`policies.sql`. The policies use the patterns from lessons 03–05 and 09: one tag with two masking policies attached (one per
data type), a mapping-table row access policy, network rules, the SNOWFLAKE database roles for the auditor, and a daily alert
that enforces the same holder rule as the tests, but against the live account:

```sql
-- Governance objects for the platform. Run as SECURITYADMIN (policies, tags, network) and ACCOUNTADMIN where noted.
USE ROLE SECURITYADMIN;
CREATE DATABASE IF NOT EXISTS governance;
CREATE SCHEMA IF NOT EXISTS governance.policies WITH MANAGED ACCESS;

-- 1. classification vocabulary
CREATE TAG IF NOT EXISTS governance.policies.pii
  ALLOWED_VALUES 'email', 'phone', 'name', 'salary', 'national_id';

-- 2. tag-based masking: one policy per data type
CREATE OR REPLACE MASKING POLICY governance.policies.pii_string AS (val VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('PII_READER') THEN val
    WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('governance.policies.pii') = 'email' THEN '***@' || SPLIT_PART(val, '@', 2)
    ELSE '*** masked ***'
  END;
CREATE OR REPLACE MASKING POLICY governance.policies.pii_number AS (val NUMBER) RETURNS NUMBER ->
  CASE WHEN IS_ROLE_IN_SESSION('PII_READER') OR IS_ROLE_IN_SESSION('SALARY_READER') THEN val ELSE NULL END;
ALTER TAG governance.policies.pii SET MASKING POLICY governance.policies.pii_string,
                                     MASKING POLICY governance.policies.pii_number;

ALTER TABLE sales.mart.customers MODIFY COLUMN email SET TAG governance.policies.pii = 'email';
ALTER TABLE sales.mart.customers MODIFY COLUMN phone SET TAG governance.policies.pii = 'phone';
ALTER TABLE hr.mart.salaries MODIFY COLUMN salary SET TAG governance.policies.pii = 'salary';

-- 3. row access by region, driven by a mapping table
CREATE TABLE IF NOT EXISTS governance.policies.region_access (role_name VARCHAR, region VARCHAR);
INSERT INTO governance.policies.region_access VALUES
  ('SALES_ANALYST', 'EU'), ('SALES_ENGINEER', 'EU'), ('SALES_ENGINEER', 'US'), ('SUPPORT_AGENT', 'EU');
CREATE OR REPLACE ROW ACCESS POLICY governance.policies.region_rows AS (row_region VARCHAR) RETURNS BOOLEAN ->
  EXISTS (SELECT 1 FROM governance.policies.region_access m
          WHERE IS_ROLE_IN_SESSION(m.role_name) AND m.region = row_region);
ALTER TABLE sales.mart.orders ADD ROW ACCESS POLICY governance.policies.region_rows ON (region);

-- 4. network: everyone from the VPN; the ETL service only from its NAT gateway
CREATE NETWORK RULE governance.policies.corp_vpn TYPE = IPV4 MODE = INGRESS VALUE_LIST = ('203.0.113.0/24');
CREATE NETWORK RULE governance.policies.etl_nat  TYPE = IPV4 MODE = INGRESS VALUE_LIST = ('192.0.2.10/32');
CREATE NETWORK POLICY corp_only ALLOWED_NETWORK_RULE_LIST = ('governance.policies.corp_vpn');
CREATE NETWORK POLICY etl_only  ALLOWED_NETWORK_RULE_LIST = ('governance.policies.etl_nat');
ALTER USER etl_service SET NETWORK_POLICY = etl_only;
-- apply to the account only after testing on a user (lesson 03):
-- ALTER ACCOUNT SET NETWORK_POLICY = corp_only;

-- 5. auditing (ACCOUNTADMIN)
USE ROLE ACCOUNTADMIN;
GRANT DATABASE ROLE SNOWFLAKE.GOVERNANCE_VIEWER TO ROLE auditor;
GRANT DATABASE ROLE SNOWFLAKE.SECURITY_VIEWER TO ROLE auditor;

-- 6. daily alert: the PII and salary roles must stay with their approved holders (the same rule test_access.py checks)
USE ROLE SECURITYADMIN;
CREATE OR REPLACE ALERT governance.policies.pii_holders_check
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 7 * * * UTC'
  IF (EXISTS (
    SELECT name, grantee_name FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
    WHERE deleted_on IS NULL AND granted_on = 'ROLE'
      AND name IN ('PII_READER', 'SALARY_READER', 'SUPPORT_AGENT')
      AND grantee_name NOT IN ('SUPPORT_AGENT', 'PII_STEWARD')
    UNION ALL
    SELECT role, grantee_name FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
    WHERE deleted_on IS NULL
      AND role IN ('PII_READER', 'SALARY_READER', 'SUPPORT_AGENT', 'PII_STEWARD')
      AND grantee_name NOT IN ('SAM', 'HANA', 'PRIYA')
  ))
  THEN
    CALL SYSTEM$SEND_EMAIL('security_email_int', 'security-team@example.com',
      'PII role granted outside the approved set',
      'A PII or salary role has a holder outside the approved list. Check GRANTS_TO_ROLES and GRANTS_TO_USERS.');
ALTER ALERT governance.policies.pii_holders_check RESUME;
```

Points worth noticing:

- **Tag-based masking.** Classifying a new column (`SET TAG … = 'email'`) protects it immediately, with no policy change.
  A tag can carry one masking policy per data type, which is why there are two. Inside the string policy,
  `SYSTEM$GET_TAG_ON_CURRENT_COLUMN` lets one policy mask emails and other strings differently.
- **The managed-access schema** keeps policies and the mapping table under the schema owner's control. Object owners
  can't grant on them.
- **The row access policy reads a mapping table.** Onboarding a role to a region is an `INSERT`, not a policy change. Protect
  that table as carefully as the policy itself.
- **The network policy is applied to the service user first.** The account-level policy stays commented out until it has been
  tested, because a wrong account-level policy locks everyone out (lesson 03).
- **The alert checks state, not events.** ACCOUNT_USAGE views lag by up to a few hours. A rule like "nobody outside this list
  holds the role" works whatever the lag. "Alert on grants in the last hour" would miss some.

## Step 4 — Verify the policies

Snowflake can **simulate** a query as another role with `EXECUTE USING POLICY_CONTEXT`. Per the documentation, the role
running it must **own** the tables and hold the **APPLY** privilege for the policies, and the result reflects the role
hierarchy and grants. `policy_tests.sql` (expected result in each comment):

```sql
-- Expected results per role, run after policies.sql. POLICY_CONTEXT must be run by a role that OWNS the tables
-- and has APPLY on the policies (in this design, the schema owner with the APPLY privileges granted).
-- 1. analyst: emails masked, EU rows only
EXECUTE USING POLICY_CONTEXT(CURRENT_ROLE => 'SALES_ANALYST')
  AS SELECT COUNT_IF(email LIKE '***@%') = COUNT(*) AS all_masked FROM sales.mart.customers;            -- TRUE
EXECUTE USING POLICY_CONTEXT(CURRENT_ROLE => 'SALES_ANALYST')
  AS SELECT ARRAY_AGG(DISTINCT region) AS regions FROM sales.mart.orders;                                -- ["EU"]
-- 2. support agent: raw emails (inherits PII_READER), EU rows only
EXECUTE USING POLICY_CONTEXT(CURRENT_ROLE => 'SUPPORT_AGENT')
  AS SELECT COUNT_IF(email LIKE '***@%') AS masked_rows FROM sales.mart.customers;                      -- 0
-- 3. sales engineer: masked emails, EU and US rows
EXECUTE USING POLICY_CONTEXT(CURRENT_ROLE => 'SALES_ENGINEER')
  AS SELECT ARRAY_AGG(DISTINCT region) WITHIN GROUP (ORDER BY region) AS regions FROM sales.mart.orders;  -- ["EU","US"]
-- 4. SYSADMIN: masked emails and no salaries (PII_READER and SALARY_READER are outside its hierarchy)
EXECUTE USING POLICY_CONTEXT(CURRENT_ROLE => 'SYSADMIN')
  AS SELECT COUNT_IF(email LIKE '***@%') = COUNT(*) AS all_masked FROM sales.mart.customers;            -- TRUE
EXECUTE USING POLICY_CONTEXT(CURRENT_ROLE => 'SYSADMIN')
  AS SELECT COUNT(salary) AS visible_salaries FROM hr.mart.salaries;                                     -- 0
-- 5. signed in as hana, with the secondary role active: every salary visible
USE ROLE hr_analyst;
USE SECONDARY ROLES ALL;
SELECT COUNT(salary) = COUNT(*) AS all_visible FROM hr.mart.salaries;                                    -- TRUE
```

The last check signs in as hana, because it depends on a secondary role that only her user holds.

## How It Actually Works

When a query runs, Snowflake first checks **privileges**: the session's primary and secondary roles, expanded through the
role hierarchy, must hold `SELECT` on the table and `USAGE` on its schema and database. That's what the RBAC model computes,
and it's what the 16-row access matrix tests. Then, during compilation, the services layer rewrites the query: the **row
access policy** becomes a filter on the table, and each tagged column is wrapped in its **masking policy** expression. Both
are evaluated against the session's context (`IS_ROLE_IN_SESSION`, `CURRENT_USER`), so they inherit exactly the same role
hierarchy as privileges do. That's why the structural tests matter as much as the matrix. A masking rule is only as narrow
as the set of roles that inherit the role it names. Network policies sit in front of all of this, at sign-in. The alert is
just a scheduled query on the audit views, so it closes the loop: the tests check the design before deployment, and the alert
checks that the deployed account still matches it.

## Common mistakes

- **Granting every functional role to SYSADMIN, including the PII ones.** Administrators then inherit raw PII. That's the
  bug this project's first test run found.
- **Masking policies that name a role low in the hierarchy.** Every role above it is also unmasked.
- **Orphaned access roles.** A role with grants but no holders is dead weight, or a sign that someone can't do their job.
- **Testing access by hand once.** Grant scripts change. Keep the matrix as code and run it on every change.
- **An account-level network policy applied before testing on one user.**
- **Alerts on recent events from ACCOUNT_USAGE.** Their latency makes "last hour" checks unreliable. Check state.

## Exercise

1. Run `grants.sql`, `policies.sql` and `policy_tests.sql` in a trial account (create the `sales`, `hr`, warehouses and users
   first, replacing the example IP ranges and email address). Does every query in `policy_tests.sql` return the expected
   value? Note any differences and explain them.
2. Add a finance team with its own raw and mart schemas. Extend the expected matrix first, watch the new tests fail, then
   change the grants until they pass.
3. Add a `region` restriction for support agents in the US. Which file changes: the policy, the mapping table or the grants?
4. Grant `pii_reader` to `sales_analyst` in a test copy of `grants.sql`. Which tests fail? Then do the same in the trial
   account and confirm that the alert would fire on its next run (run its condition query by hand).
5. Write a sixth structural test: no user holds both `SALES_ENGINEER` and `HR_ENGINEER`, directly or through a role.
