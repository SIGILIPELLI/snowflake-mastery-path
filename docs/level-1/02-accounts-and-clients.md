---
description: "Snowflake account identifiers, Snowsight, Snowflake CLI connections with key-pair auth, and connecting from Python with the right session context."
---

# 02 · Accounts, Snowsight, Snowflake CLI & Python

Before writing any SQL you need three things: an **account** to connect to, a **client** to connect with, and an
understanding of the **session context** (role, warehouse, database, schema) that every statement runs in. Most
"permission denied" and "no active warehouse" errors new users hit come from the last one.

## Accounts and identifiers

A Snowflake account lives in one cloud region and belongs to an **organization**. The preferred way to name an account
is the organization-based identifier:

```text
<orgname>-<account_name>              e.g.  myorg-analytics
https://<orgname>-<account_name>.snowflakecomputing.com
```

Older material uses the **account locator** form (`xy12345.eu-west-1`), which still works but ties the identifier to a
region. Use the `orgname-accountname` form in new configuration. Once connected you can check where you are:

```sql
SELECT CURRENT_ORGANIZATION_NAME(), CURRENT_ACCOUNT_NAME(), CURRENT_REGION(), CURRENT_ACCOUNT();
```

## Snowsight

**Snowsight** is the web interface: SQL worksheets, Python notebooks, dashboards, data browsing, query history with the
Query Profile (Level 2 · 03), cost views and administration. A worksheet has its own role, warehouse, database and schema
selectors at the top, which set the session context for the statements in it. When a query fails in a worksheet, check
those four selectors first.

## Session context

Every statement runs with a current **role**, **warehouse**, **database** and **schema**:

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE learn_wh;
USE DATABASE learn_db;
USE SCHEMA public;

SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA();
```

- **Role** decides what you're allowed to do (Level 3 · 01). Your user has a default role, and you can switch to any
  role granted to you. The same user can see different objects under different roles.
- **Warehouse** is the compute that runs queries. Without one, anything that scans data fails with an error saying no
  active warehouse is selected. Metadata operations and result-cache hits don't need one.
- **Database and schema** resolve unqualified names. `orders` means `learn_db.public.orders` here. Fully qualified
  names (`database.schema.object`) work regardless of context, and scripts that run unattended should use them.

## Snowflake CLI

Snowflake's own documentation now calls **SnowSQL** "a legacy command-line client". It's still supported (version 1.5.x
until 16 April 2028), but new features go only into **Snowflake CLI**, the `snow` command. Install it with pip (or
pipx, or Homebrew):

```bash
pip install snowflake-cli
snow --version
```

```text
Snowflake CLI version: 3.29.0
```

The top-level commands show how wide its scope is: running SQL, managing stages and objects, Snowpark functions,
Streamlit apps, notebooks, dbt projects, Git repositories and more (excerpt of real `snow --help` output):

```text
│ connection     Manages connections to Snowflake.                             │
│ cortex         Provides access to Snowflake Cortex.                          │
│ dbt            Manages dbt on Snowflake projects.                            │
│ git            Manages git repositories in Snowflake.                        │
│ object         Manages Snowflake objects like warehouses and stages          │
│ snowpark       Manages procedures and functions.                             │
│ sql            Executes SQL statements against Snowflake interactively or    │
│                from a query, file, or standard input.                        │
│ stage          Manages stages.                                               │
│ streamlit      Manages a Streamlit app in Snowflake.                         │
```

### A connection with key-pair authentication

Snowflake is phasing out single-factor password sign-ins: humans need multi-factor authentication, and scripts and
service users should use key pairs, OAuth, programmatic access tokens or workload identity federation (Level 3 · 02
has the details and dates). Key-pair authentication is the simplest to set up from a laptop. Generate an RSA key pair:

```bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```

(`-nocrypt` leaves the private key unencrypted, which is fine for a lab. In production, encrypt it, or better, keep it
in a secret manager.) Register the **public** key on your user, pasting the base64 body without the
`-----BEGIN/END-----` lines:

```sql
ALTER USER ana SET RSA_PUBLIC_KEY = 'MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC...';
DESC USER ana;   -- the RSA_PUBLIC_KEY_FP property shows the key's SHA-256 fingerprint
```

You can compute that fingerprint locally and compare it with `DESC USER`. The two should match:

```bash
openssl rsa -pubin -in rsa_key.pub -outform DER | openssl dgst -sha256 -binary | openssl enc -base64
```

```text
SHA256:xWy4qmg9a9Y2zY8a0g/qRNp2QGRbDv7emETjmB0BSSU=
```

(That's the fingerprint of the throwaway key generated for this lesson.) Then add a named connection. With
`--no-interactive` nothing is asked, and nothing is sent to Snowflake yet:

```bash
snow connection add --connection-name trial --account myorg-myaccount --user ANA \
  --authenticator SNOWFLAKE_JWT --private-key-file ~/keys/rsa_key.p8 \
  --warehouse LEARN_WH --database LEARN_DB --schema PUBLIC --role SYSADMIN --no-interactive
```

```text
Wrote new connection trial to …/config.toml
```

The configuration it wrote:

```toml
[connections.trial]
account = "myorg-myaccount"
user = "ANA"
database = "LEARN_DB"
schema = "PUBLIC"
warehouse = "LEARN_WH"
role = "SYSADMIN"
authenticator = "SNOWFLAKE_JWT"
private_key_file = "~/keys/rsa_key.p8"
```

From here, `snow connection test -c trial` checks the connection, and `snow sql -c trial -q "SELECT CURRENT_USER()"`
runs a query. (Those two contact Snowflake, so they weren't run for this course.) The same config file is read by the
Python connector and Snowpark, so one place defines how all your tools connect. You can see the connector reading it
without any network call by asking for a connection name that doesn't exist:

```python
import snowflake.connector
try:
    snowflake.connector.connect(connection_name="prod")
except Exception as e:
    print(type(e).__name__, str(e))
```

```text
Error Invalid connection_name 'prod', known ones are ['trial']
```

The connector looks in the directory given by the `SNOWFLAKE_HOME` environment variable (here pointed at the lab
directory) or in its default per-user config directory (`~/.snowflake` on many setups). With the file readable by
other users (`chmod 644`), the same call still worked but printed a warning, `Bad owner or permissions on …`.
`chmod 600` the file, and keep it out of version control.

## Connecting from Python

The **Snowflake Connector for Python** follows Python's DB-API. Installed versions used for this lesson:
`snowflake-connector-python` 4.7.5 (pulled in alongside Snowflake CLI 3.29.0).

```python
import snowflake.connector

conn = snowflake.connector.connect(
    connection_name="trial",          # reads [connections.trial] from the config file
)
try:
    cur = conn.cursor()
    cur.execute("SELECT CURRENT_USER(), CURRENT_ROLE(), CURRENT_WAREHOUSE()")
    print(cur.fetchone())

    # bind parameters: never build SQL with f-strings from user input
    cur.execute("SELECT COUNT(*) FROM orders WHERE region = %(region)s", {"region": "EU"})
    print(cur.fetchone()[0])
    print("query id:", cur.sfqid)     # look it up in Snowsight's query history
finally:
    conn.close()
```

A few facts confirmed by inspecting the installed package:

```text
snowflake-connector-python 4.7.5
write_pandas exists: True
DictCursor exists: True
paramstyle default: pyformat
```

- The default parameter style is **`pyformat`** (`%(name)s` and `%s`), with values escaped on the client side.
  Server-side binding with `?` or `:1` is available by setting `snowflake.connector.paramstyle = "qmark"` (or
  `"numeric"`) before connecting.
- `snowflake.connector.pandas_tools.write_pandas(conn, df, "TABLE_NAME")` bulk-loads a DataFrame by staging it as
  Parquet and running `COPY INTO`, which is far faster than row-by-row `INSERT`s.
- `cur.sfqid` is the **query ID**. Log it with your application's errors so you can find the query in Snowflake's
  history.

## How It Actually Works

A client connects over HTTPS to `<account>.snowflakecomputing.com`, which is the **cloud services** layer. With key-pair
authentication, the client signs a short-lived JSON Web Token with your private key. Snowflake verifies the signature
against the public key registered on your user, so the private key never leaves your machine. On success the client
receives a **session token** and attaches it to every request. The session holds your current role, warehouse, database,
schema and parameters (time zone, statement timeout and so on), which is why `USE` statements affect later statements
in the same session but not other sessions.

Queries are submitted as requests. Small results come back inline. Large results are written by Snowflake to internal
storage as compressed chunks, and the client downloads them in parallel directly from object storage, which is why
fetching millions of rows is much faster than the request round trip alone would suggest. Every statement gets a
**query ID** that ties together the client call, the query history, the Query Profile and the access history (Level 3 · 09).

## Common mistakes

- **No warehouse in the session**, often because the user's default warehouse isn't set or the role can't use it.
- **Relying on context in scripts.** Fully qualify names in jobs, so a changed default doesn't send writes to the wrong
  schema.
- **Passwords in code or config committed to Git.** Use key pairs or tokens, and keep `config.toml` private.
- **Building SQL with f-strings.** Use bind parameters.
- **Starting new work on SnowSQL.** It's legacy. Use Snowflake CLI.
- **Using the account locator** in new config. The `orgname-accountname` identifier is region-independent.

## Exercise

1. In a trial account, create a user for scripts, register a key pair, and verify that the fingerprint from `DESC USER`
   matches the one you computed with `openssl`.
2. Configure two named connections (one with `SYSADMIN`, one with a read-only role you'll create in Level 3) and run the
   same `SELECT CURRENT_ROLE()` through each with `snow sql -c`.
3. Write a Python function that runs a parameterized query and returns rows as dicts (`DictCursor`), and logs the query
   ID on error. Find a failing query in Snowsight's history using that ID.
4. Run `SHOW PARAMETERS IN SESSION;` and find three parameters that would change query results or behaviour (for
   example, the time zone).
