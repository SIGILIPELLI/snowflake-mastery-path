---
description: "Snowflake access control: system roles, the hierarchy, access vs functional roles, future grants, managed access, database roles and a grant analyser."
---

# 01 · Access Control: Roles, Privileges & Hierarchies

In Snowflake, **privileges are granted to roles, and roles are granted to users** (and to other roles). Nothing is
accessible unless a grant allows it. That model is simple, and in a real account it grows into hundreds of roles and
thousands of grants that nobody can hold in their head. This lesson covers the built-in roles, the design pattern that
keeps custom roles manageable, the features that keep grants from rotting, and a small offline tool for answering "who
can read this table?" from a grant script.

## The system roles

From Snowflake's documentation:

| Role | Purpose |
|---|---|
| `ACCOUNTADMIN` | Top-level role. Encapsulates `SYSADMIN` and `SECURITYADMIN`; account parameters, billing. Grant to very few users. |
| `SECURITYADMIN` | Manages any grant globally (`MANAGE GRANTS`); creates and manages users and roles. Inherits `USERADMIN`. |
| `USERADMIN` | Creates users and roles (`CREATE USER`, `CREATE ROLE`). |
| `SYSADMIN` | Creates warehouses, databases and the objects in them. Custom roles should roll up to it. |
| `PUBLIC` | Granted automatically to every user and role. Anything it owns is available to everyone. |
| `ORGADMIN` / `GLOBALORGADMIN` | Organization-level tasks (accounts, org-wide usage). `GLOBALORGADMIN` exists only in the organization account. |

Snowflake's own recommendations for `ACCOUNTADMIN`: require MFA for its users, assign it to **at least two** users
(resetting a lone administrator's lost credentials goes through support procedures that can take up to two business
days), and don't use it to create objects. Day-to-day work happens in custom roles under `SYSADMIN`.

## Designing roles: access roles and functional roles

The pattern that scales is two layers:

- **Access roles** hold privileges on objects, one per (container, level): `sales_mart_read`, `sales_mart_write`,
  `sales_raw_read`. They are never granted to users directly.
- **Functional roles** describe what people do (`analyst`, `data_engineer`, `finance_viewer`) and are composed from
  access roles. Users get functional roles.

```sql
USE ROLE USERADMIN;
CREATE ROLE IF NOT EXISTS sales_mart_read;
CREATE ROLE IF NOT EXISTS sales_mart_write;
CREATE ROLE IF NOT EXISTS sales_raw_read;
CREATE ROLE IF NOT EXISTS analyst;
CREATE ROLE IF NOT EXISTS data_engineer;

USE ROLE SECURITYADMIN;
GRANT USAGE ON DATABASE sales TO ROLE sales_mart_read;
GRANT USAGE ON SCHEMA sales.mart TO ROLE sales_mart_read;
GRANT SELECT ON ALL TABLES IN SCHEMA sales.mart TO ROLE sales_mart_read;
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales.mart TO ROLE sales_mart_read;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA sales.mart TO ROLE sales_mart_write;
GRANT INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA sales.mart TO ROLE sales_mart_write;
GRANT ROLE sales_mart_read TO ROLE sales_mart_write;
GRANT USAGE ON DATABASE sales TO ROLE sales_raw_read;
GRANT USAGE ON SCHEMA sales.raw TO ROLE sales_raw_read;
GRANT SELECT ON ALL TABLES IN SCHEMA sales.raw TO ROLE sales_raw_read;
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales.raw TO ROLE sales_raw_read;

GRANT ROLE sales_mart_read TO ROLE analyst;
GRANT USAGE ON WAREHOUSE bi_wh TO ROLE analyst;
GRANT ROLE analyst TO ROLE data_engineer;
GRANT ROLE sales_mart_write TO ROLE data_engineer;
GRANT ROLE sales_raw_read TO ROLE data_engineer;
GRANT USAGE ON WAREHOUSE etl_wh TO ROLE data_engineer;

-- every functional role rolls up to SYSADMIN, so administrators can manage what these roles own
GRANT ROLE analyst TO ROLE SYSADMIN;
GRANT ROLE data_engineer TO ROLE SYSADMIN;

GRANT ROLE analyst TO USER ana;
GRANT ROLE data_engineer TO USER ben;
```

Three details make the difference between grants that work and grants that rot:

- **Reading a table needs `USAGE` on its database and schema** as well as `SELECT` on the table. Missing `USAGE` is the most
  common "it says the table doesn't exist" problem.
- **`ON FUTURE TABLES`** grants privileges on tables created later. `ON ALL TABLES` covers only those that exist now.
  You almost always want both.
- **A role inherits everything granted to roles granted to it.** `data_engineer` gets everything `analyst` can do.

## Inspecting grants

```sql
SHOW GRANTS TO ROLE analyst;              -- what this role has been granted
SHOW GRANTS OF ROLE analyst;              -- who (users, roles) has this role
SHOW GRANTS ON TABLE sales.mart.daily_revenue;
SHOW FUTURE GRANTS IN SCHEMA sales.mart;
SHOW GRANTS TO USER ana;
```

These show **direct** grants. Effective access through the hierarchy has to be computed, and that's the job of the next
section's tool (or of the `GRANTS_TO_ROLES` and `GRANTS_TO_USERS` views in `ACCOUNT_USAGE`, Level 3 · 09).

## Who can read this table? An offline analyser

Grant scripts belong in version control, and reviewing them is much easier with a tool that computes effective access.
Here is a deliberately small model of Snowflake's RBAC in Python. It ignores ownership, secondary roles and database
roles, and covers the grant forms above:

```python
"""A small model of Snowflake role-based access control, for reasoning about grant scripts offline.
Understands:  GRANT ROLE r TO ROLE p | GRANT ROLE r TO USER u |
              GRANT <privs> ON <TYPE> <name> TO ROLE r   (privs comma-separated; TYPE = WAREHOUSE, DATABASE, SCHEMA, TABLE, VIEW)
              GRANT <privs> ON ALL|FUTURE TABLES IN SCHEMA <schema> TO ROLE r
It computes effective privileges through the role hierarchy (a role has the privileges of every role granted to it).
It is a teaching model: it ignores ownership, secondary roles, database roles and managed-access rules."""
import re
from collections import defaultdict

class Account:
    def __init__(self):
        self.parents = defaultdict(set)      # role -> roles it is granted to (its "parents")
        self.children = defaultdict(set)     # role -> roles granted to it (it inherits these)
        self.user_roles = defaultdict(set)
        self.privs = defaultdict(set)        # role -> {(PRIV, TYPE, NAME)}
        self.roles = {"ACCOUNTADMIN", "SECURITYADMIN", "USERADMIN", "SYSADMIN", "PUBLIC"}
        for child, parent in [("SYSADMIN", "ACCOUNTADMIN"), ("SECURITYADMIN", "ACCOUNTADMIN"), ("USERADMIN", "SECURITYADMIN")]:
            self._role_to_role(child, parent)

    def _role_to_role(self, child, parent):
        self.children[parent].add(child); self.parents[child].add(parent); self.roles |= {child, parent}

    def run(self, script):
        script = re.sub(r"--[^\n]*", "", script)            # drop comments first, then split statements
        for stmt in [s.strip() for s in script.split(";") if s.strip()]:
            s = re.sub(r"\s+", " ", stmt).upper()
            if s.startswith("USE ROLE "):
                continue                                  # session context only
            if m := re.fullmatch(r"CREATE ROLE (IF NOT EXISTS )?(\S+)", s):
                self.roles.add(m[2])
            elif m := re.fullmatch(r"GRANT ROLE (\S+) TO ROLE (\S+)", s):
                self._role_to_role(m[1], m[2])
            elif m := re.fullmatch(r"GRANT ROLE (\S+) TO USER (\S+)", s):
                self.user_roles[m[2]].add(m[1])
            elif m := re.fullmatch(r"GRANT (.+?) ON (ALL|FUTURE) TABLES IN SCHEMA (\S+) TO ROLE (\S+)", s):
                for p in m[1].split(","):
                    self.privs[m[4]].add((p.strip(), "TABLE", m[3] + ".*"))
            elif m := re.fullmatch(r"GRANT (.+?) ON (WAREHOUSE|DATABASE|SCHEMA|TABLE|VIEW) (\S+) TO ROLE (\S+)", s):
                for p in m[1].split(","):
                    self.privs[m[4]].add((p.strip(), m[2], m[3]))
            else:
                raise ValueError(f"not understood: {stmt}")

    def inherited(self, role):
        seen, stack = set(), [role]
        while stack:
            r = stack.pop()
            if r not in seen:
                seen.add(r); stack.extend(self.children[r])
        return seen

    def effective(self, role):
        return {p for r in self.inherited(role) for p in self.privs[r]}

    def can(self, role, priv, typ, name):
        for p, t, n in self.effective(role):
            if p in (priv, "ALL") and t == typ and (n == name or (n.endswith(".*") and name.startswith(n[:-1]))):
                return True
        return False

    def who_can(self, priv, typ, name):
        return sorted(u for u, rs in self.user_roles.items() if any(self.can(r, priv, typ, name) for r in rs))

    def orphans(self):
        """custom roles that SYSADMIN does not inherit: objects they own escape SYSADMIN's oversight"""
        return sorted(r for r in self.roles if r not in self.inherited("SYSADMIN") and r not in
                      {"ACCOUNTADMIN", "SECURITYADMIN", "USERADMIN", "PUBLIC"} and r not in self.inherited("SECURITYADMIN"))
```

Run against the grant script above, plus an `intern` role that someone granted directly on a PII table and forgot to
attach to the hierarchy:

```python
acct = Account()
acct.run(open("grants.sql").read())          # the script above, plus the intern's three statements
for q in [("SELECT", "TABLE", "SALES.MART.DAILY_REVENUE"), ("SELECT", "TABLE", "SALES.RAW.CUSTOMERS_PII"),
          ("UPDATE", "TABLE", "SALES.MART.DAILY_REVENUE")]:
    print(f"{q[0]:6} on {q[2]:26} -> users: {acct.who_can(*q)}")
print("data_engineer inherits:", sorted(acct.inherited("DATA_ENGINEER")))
print("roles SYSADMIN cannot see into:", acct.orphans())
```

```text
SELECT on SALES.MART.DAILY_REVENUE   -> users: ['ANA', 'BEN']
SELECT on SALES.RAW.CUSTOMERS_PII    -> users: ['BEN', 'CY']
UPDATE on SALES.MART.DAILY_REVENUE   -> users: ['BEN']
data_engineer inherits: ['ANALYST', 'DATA_ENGINEER', 'SALES_MART_READ', 'SALES_MART_WRITE', 'SALES_RAW_READ']
roles SYSADMIN cannot see into: ['INTERN']
```

The report surfaces two things a reviewer wants to know: the intern **can read the PII table**, through a direct grant
nobody would find by looking at the access roles, and `INTERN` is an **orphan** outside the `SYSADMIN` hierarchy, so
objects it creates would be invisible to the administrators who are supposed to manage them.

The first version of this tool had its own bug, and it's instructive. It dropped any statement that *started* with a
comment line, so the `-- users` heading silently swallowed `GRANT ROLE analyst TO USER ana`, and the report said Ana
couldn't read the mart. A security tool that silently misreads its input is worse than none. Test it on cases where you
know the answer.

## Keeping grants manageable

**Managed access schemas.** In a normal schema, each object's owner can grant access to it. In a schema created
`WITH MANAGED ACCESS`, only the schema owner (or a role with `MANAGE GRANTS`) can grant privileges on objects inside,
which centralizes control:

```sql
CREATE SCHEMA sales.mart WITH MANAGED ACCESS;
ALTER SCHEMA sales.raw ENABLE MANAGED ACCESS;
```

**Database roles** are roles that live **inside a database**. They're the unit for granting access to shared databases
(Level 3 · 07), and they travel with a database when it's cloned or replicated:

```sql
CREATE DATABASE ROLE sales.reader;
GRANT USAGE ON SCHEMA sales.mart TO DATABASE ROLE sales.reader;
GRANT SELECT ON ALL TABLES IN SCHEMA sales.mart TO DATABASE ROLE sales.reader;
GRANT DATABASE ROLE sales.reader TO ROLE analyst;
```

**Secondary roles.** A session has one primary role, which determines what owns newly created objects, and can also
activate secondary roles whose privileges are combined for reads:

```sql
USE SECONDARY ROLES ALL;      -- every role granted to the user contributes privileges
USE SECONDARY ROLES NONE;     -- only the primary role
```

Per Snowflake's docs, privileges granted **directly to users** (rather than to roles) are only considered when secondary
roles are set to `ALL`. Prefer roles anyway.

## How It Actually Works

Every securable object has exactly one **owner role**, which has full control, including granting privileges on it (unless
the object is in a managed access schema). When a statement runs, Snowflake checks the privileges of the session's
**primary role and its inherited roles** (and secondary roles, if active) for each object involved. The role hierarchy is
a directed graph. Granting role A to role B makes B a parent of A, and B inherits A's privileges, transitively. The
system roles are already wired into it (`USERADMIN` → `SECURITYADMIN` → `ACCOUNTADMIN`, `SYSADMIN` → `ACCOUNTADMIN`),
which is why custom roles should be granted to `SYSADMIN`: otherwise objects they own fall outside the hierarchy that
administrators control.

Future grants are stored on the container (schema or database) and applied automatically when a matching object is created
there. Schema-level future grants take precedence over database-level ones for the same object type.

## Common mistakes

- **Granting privileges to users directly**, or functional roles that hold object privileges directly. Use access roles.
- **Forgetting `USAGE`** on the database and schema.
- **Only `ON ALL TABLES`**, so tomorrow's table is inaccessible. Add `ON FUTURE TABLES`.
- **Custom roles not rolled up to `SYSADMIN`.** Orphaned ownership.
- **Using `ACCOUNTADMIN` for daily work**, or having only one `ACCOUNTADMIN` user.
- **Objects owned by `PUBLIC`.** Everyone can access them.
- **Reviewing grant scripts by eye.** Compute effective access, and test the tool that computes it.

## Exercise

1. Build the access-role and functional-role structure for two domains (`sales`, `hr`) with read and write levels, in a
   trial account. Then answer with `SHOW GRANTS` and your own reasoning: can an `analyst` read `hr.mart.salaries`?
2. Extend `rbac.py` to understand `GRANT DATABASE ROLE … TO ROLE …` and `GRANT OWNERSHIP ON TABLE …`, and add a report of
   every user who can **write** to any table in `sales.mart`.
3. Create a managed access schema and show that a table owner who isn't the schema owner can't grant `SELECT` on their own
   table.
4. Find every custom role in your account that isn't granted (directly or indirectly) to `SYSADMIN`, using
   `SHOW ROLES` and `SHOW GRANTS OF ROLE` with `RESULT_SCAN`.
