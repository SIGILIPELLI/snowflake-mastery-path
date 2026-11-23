---
title: "Snowflake Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Snowflake cheat sheet: 22 quick-reference tables from every level of the Snowflake Mastery Path, each linked to its full lesson."
---

# Snowflake Cheat Sheet

Quick-reference tables from **22 lessons** of the free Snowflake Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Snowflake Foundations

### [01 · How Snowflake Is Built: Storage, Compute &amp; Services](level-1/01-architecture.md)

*Why separation matters, concretely*

| Situation | Traditional shared-nothing warehouse | Snowflake |
|---|---|---|
| ETL load runs while analysts query | Both compete for the same nodes | Separate warehouses, no CPU contention |
| Data grows 10× | Add nodes and rebalance data | Storage grows on its own, and compute is unchanged |
| A big query once a month | Size the cluster for the peak | Resize a warehouse for that query, then shrink it |
| Nobody is querying at night | Cluster still runs (and bills) | Warehouses suspend, so compute costs nothing |
| A new team needs the same data | Copy data to their cluster | Point their warehouse at the same tables |

### [03 · Databases, Schemas, Tables &amp; Data Types](level-1/03-objects-and-types.md)

*Three kinds of table*

| | Permanent | Transient | Temporary |
|---|---|---|---|
| Lives until | dropped | dropped | the session ends |
| Visible to | anyone with privileges | anyone with privileges | only the creating session |
| Time Travel | up to 90 days (Enterprise+), 1 on Standard | 0 or 1 day | 0 or 1 day |
| Fail-safe (7 days) | yes | **no** | **no** |

### [04 · Virtual Warehouses: Sizing, Suspend &amp; Credits](level-1/04-virtual-warehouses.md)

*Creating a warehouse*

| Property | Default | Why you might change it |
|---|---|---|
| `WAREHOUSE_SIZE` | `XSMALL` | start small, then size from evidence |
| `AUTO_SUSPEND` | `600` seconds | 10 idle minutes are billed after every burst of work |
| `AUTO_RESUME` | `TRUE` | keep it on, so queries start the warehouse |
| `INITIALLY_SUSPENDED` | `FALSE` | otherwise creating the warehouse starts billing immediately |
| `STATEMENT_TIMEOUT_IN_SECONDS` | `172800` (2 days) | a runaway query can run for two days |
| `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` | `0` (no timeout) | queued queries wait forever |
| `MAX_CONCURRENCY_LEVEL` | `8` | how many queries run at once before others queue |

### [05 · Loading Data: Stages, File Formats &amp; COPY INTO](level-1/05-loading-copy-into.md)

*Stages*

| Stage | Reference | Use |
|---|---|---|
| User stage | `@~` | your own scratch files |
| Table stage | `@%orders` | files for one table only |
| Named internal stage | `@learn_db.raw.landing` | files uploaded from clients, managed by Snowflake |
| Named external stage | `@learn_db.raw.s3_landing` | files in your own S3/GCS/Azure bucket |

### [08 · Time Travel, UNDROP &amp; Fail-safe](level-1/08-time-travel-failsafe.md)

*Retention*

| | Standard Edition | Enterprise and higher |
|---|---|---|
| Default retention | 1 day | 1 day |
| Maximum, permanent objects | 1 day | **90 days** |
| Transient and temporary objects | 0 or 1 day | 0 or 1 day |
| Fail-safe after retention | 7 days (permanent only) | 7 days (permanent only) |

### [09 · Zero-Copy Cloning &amp; Environments](level-1/09-zero-copy-cloning.md)

*What comes along, and what doesn't*

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

## Level 2 · Performance & Data Engineering

### [05 · Continuous Loading with Snowpipe](level-2/05-snowpipe.md)

*How Snowpipe differs from bulk COPY*

| | Bulk `COPY INTO` | Snowpipe |
|---|---|---|
| Trigger | you run it (or a task does) | file events or REST calls |
| Compute | your warehouse | serverless |
| Typical latency | as often as you run it | typically within a minute of Snowpipe learning about a file |
| Billing | warehouse time | **0.0037 credits per GB** (all editions since 8 December 2025) |
| Default `ON_ERROR` | `ABORT_STATEMENT` | **`SKIP_FILE`** |
| Transaction | one per `COPY` statement | per file (files may be combined or split across transactions) |
| Load metadata | per table, 64 days | **per pipe, 14 days** |

### [06 · Streams: Change Data Capture](level-2/06-streams-cdc.md)

*Net changes, not a log*

| Changes since the offset | Stream contents |
|---|---|
| `INSERT` row 1 | one `INSERT` row, `ISUPDATE = FALSE` |
| `UPDATE` existing row 2 | `DELETE` (old values) + `INSERT` (new values), both `ISUPDATE = TRUE` |
| `INSERT` row 3, then `UPDATE` it | **one** `INSERT` with the final values, `ISUPDATE = FALSE` (it's a new row as far as the delta is concerned) |
| `INSERT` row 4, then `DELETE` it | **nothing** |
| `UPDATE` row 5 three times | one `DELETE`/`INSERT` pair: original values → latest values |
| `DELETE` existing row 6 | one `DELETE` row |

### [07 · Tasks &amp; Task Graphs](level-2/07-tasks.md)

*Where the compute comes from*

| | User-managed warehouse | Serverless |
|---|---|---|
| Declared by | `WAREHOUSE = etl_wh` | omitting `WAREHOUSE` (optionally `USER_TASK_MANAGED_INITIAL_WAREHOUSE_SIZE`) |
| Sizing | you choose | Snowflake predicts from recent runs, up to the equivalent of XXLARGE |
| Billing | warehouse time, shared with whatever else runs there | per-second serverless compute at the serverless-task rate |
| Privilege needed | `USAGE` on the warehouse | the account-level `EXECUTE MANAGED TASK` |

### [08 · Dynamic Tables](level-2/08-dynamic-tables.md)

*Dynamic tables, streams and tasks, or a view?*

| Need | Use |
|---|---|
| A transformation result that should stay within N minutes of its sources | **dynamic table** |
| Complex procedural logic, side effects, calls to external systems, notifications | **streams + tasks** (or procedures) |
| Exact control over when and how each step runs | **streams + tasks** |
| Always-current results, cheap to compute at query time | a **view** |
| An expensive aggregate on one table, transparently used by queries (Enterprise) | a **materialized view** |

### [09 · Unloading, External Tables &amp; Iceberg](level-2/09-unloading-iceberg.md)

*Apache Iceberg tables*

| | Snowflake-managed (Snowflake is the catalog) | Externally managed (another catalog) |
|---|---|---|
| Writes from Snowflake | yes, full DML | limited / read-focused |
| Table maintenance (e.g. compaction) | Snowflake does it | the external catalog/engine |
| Data location | your bucket (via an **external volume**) or Snowflake storage | your bucket |
| Other engines read it via | the Snowflake Horizon Catalog's Iceberg REST API, or Open Catalog sync | their own catalog |
| Needs | external volume (or `SNOWFLAKE_MANAGED`) | external volume + **catalog integration** |

## Level 3 · Security, Governance & Sharing

### [01 · Access Control: Roles, Privileges &amp; Hierarchies](level-3/01-rbac.md)

*The system roles*

| Role | Purpose |
|---|---|
| `ACCOUNTADMIN` | Top-level role. Encapsulates `SYSADMIN` and `SECURITYADMIN`; account parameters, billing. Grant to very few users. |
| `SECURITYADMIN` | Manages any grant globally (`MANAGE GRANTS`); creates and manages users and roles. Inherits `USERADMIN`. |
| `USERADMIN` | Creates users and roles (`CREATE USER`, `CREATE ROLE`). |
| `SYSADMIN` | Creates warehouses, databases and the objects in them. Custom roles should roll up to it. |
| `PUBLIC` | Granted automatically to every user and role. Anything it owns is available to everyone. |
| `ORGADMIN` / `GLOBALORGADMIN` | Organization-level tasks (accounts, org-wide usage). `GLOBALORGADMIN` exists only in the organization account. |

### [02 · Authentication: MFA, Key Pairs, OAuth &amp; SSO](level-3/02-authentication.md)

*The end of single-factor passwords*

| Estimated date | Affected users | Phase |
|---|---|---|
| Sep 2025 – Jan 2026 | human users | mandatory MFA for all Snowsight users |
| May 2026 – Jul 2026 | human users, legacy service users | strong authentication for **new** users |
| Aug 2026 – Oct 2026 | human users, legacy service users | strong authentication for **all** users |

### [08 · Encryption, Key Management &amp; Compliance](level-3/08-encryption-compliance.md)

*Rotation vs rekeying*

| | Key rotation | Periodic rekeying |
|---|---|---|
| What happens | keys older than **30 days** are retired, and new keys are created for new data | data protected by a retired key older than **one year** is **re-encrypted** with a new key |
| Old key | kept, used only to **decrypt** existing data, destroyed when no longer needed | destroyed once its data has been re-encrypted |
| Edition | all | **Enterprise and higher**, enabled by `ACCOUNTADMIN` |
| Cost to you | none | Snowflake documents a possible additional storage charge during the transition |

### [09 · Auditing with ACCOUNT_USAGE &amp; Access History](level-3/09-auditing-account-usage.md)

*ACCOUNT_USAGE vs INFORMATION_SCHEMA*

| | `SNOWFLAKE.ACCOUNT_USAGE` | each database's `INFORMATION_SCHEMA` |
|---|---|---|
| Scope | the whole account | one database (plus account-level table functions) |
| Includes dropped objects | yes (with a `DELETED` timestamp) | no |
| Latency | **45 minutes to 3 hours**, varies by view | none |
| Retention of history | **1 year** | 7 days to 6 months, varies |

## Level 4 · Building, Scaling & Operating

### [01 · UDFs &amp; Stored Procedures](level-4/01-udfs-procedures.md)

*Function or procedure?*

| | UDF | Stored procedure |
|---|---|---|
| Called | inside a statement: `SELECT net_amount(amount, 10) …` | on its own: `CALL archive_old_orders(90)` |
| Returns | one value per row (scalar) or a set of rows (table function) | one value (or a table), once |
| May run DML/DDL | no | yes |
| Typical use | reusable calculations, parsing, business rules | admin jobs, multi-step loads, anything with control flow |

### [04 · Cortex AI Functions](level-4/04-cortex-ai.md)

*The functions*

| Function | Does | Returns |
|---|---|---|
| `AI_CLASSIFY` | assigns text (or an image or document) to one or more of your categories | object with a `labels` array |
| `AI_FILTER` | evaluates a natural-language condition | `BOOLEAN`, usable in `WHERE` and `JOIN … ON` |
| `AI_SENTIMENT` | overall sentiment, optionally per aspect (up to 10) | object with `categories`: `positive`, `negative`, `neutral`, `mixed` or `unknown` |
| `AI_EXTRACT` | pulls named fields out of text or documents | object |
| `AI_SUMMARIZE_AGG`, `AI_AGG` | summarise or answer a prompt **across many rows** | string |
| `AI_TRANSLATE`, `AI_REDACT` | translate, remove PII | string |
| `AI_EMBED`, `AI_SIMILARITY` | vectors and similarity for search and clustering | vector, number |
| `AI_COMPLETE` | general prompt to a chosen model, optionally with a response schema | string or object |

### [05 · Cost Management: Monitors, Budgets &amp; Attribution](level-4/05-cost-management.md)

*Budgets: forecasts and alerts for everything*

| | Resource monitor | Budget |
|---|---|---|
| Covers | warehouses only | warehouses, serverless features, AI services, … |
| Period | daily, weekly, monthly, yearly or never | calendar month |
| Acts | notifies, **suspends** warehouses | notifies (email, cloud queue, webhook); can call your stored procedures |
| Timing | thresholds on actual usage | projections, refreshed up to every 6.5 hours by default |

### [06 · Warehouse Scaling: Up, Out &amp; Adaptive](level-4/06-warehouse-scaling.md)

*Two symptoms, two levers*

| Symptom | Evidence | Lever |
|---|---|---|
| **Each query is slow**, even alone | long execution time, spilling to local or remote storage | **scale up** (a bigger size) |
| **Queries wait** for their turn | high `queued_overload_time`, fine execution time | **scale out** (more clusters) |

### [07 · Replication, Failover &amp; Business Continuity](level-4/07-replication-bcdr.md)

*What can be replicated, on which edition*

| Capability | Edition, per the documentation |
|---|---|
| Database and share replication | **all editions** |
| Replication groups with other account objects (users, roles, warehouses, integrations, policies, …) | **Business Critical** or higher |
| **Failover groups** (promote a replica to primary) | **Business Critical** or higher |
| **Client Redirect** (connection URLs that follow the primary) | **Business Critical** or higher |

### [08 · dbt &amp; CI/CD for Snowflake](level-4/08-dbt-cicd.md)

*A bug that passed every test*

| problem          | order_date | region | orders | revenue |
| ---------------- | ---------- | ------ | ------ | ------- |
| missing or wrong | 2026-10-13 | EU     |      2 |     150 |
| unexpected       | 2026-10-13 | EU     |      3 |     200 |

### [09 · Migrating to Snowflake](level-4/09-migration.md)

*Translation runs, but does it mean the same?*

| Query | What changed silently |
|---|---|
| T-SQL `total_items / order_count` | In SQL Server, integer ÷ integer is integer division (7 / 2 = 3). In Snowflake, division of integers returns a decimal (3.5). Averages change. |
| T-SQL `first_name + ' ' + last_name` | `+` concatenates strings in SQL Server. In Snowflake, `+` is arithmetic, so this fails at run time. Use `\|\|` or `CONCAT`. |
| T-SQL `TOP 10 … ORDER BY amount DESC` | Handled correctly: the translator added `NULLS LAST`, because the two systems order NULLs differently. Good translators know these rules. Your hand-written code may not. |
| BigQuery `SAFE_CAST(value AS INT64)` | Became `CAST`, which **errors** on bad values instead of returning NULL. The Snowflake equivalent is `TRY_CAST`. |
| BigQuery `` `proj.web.events` `` | Became `"proj"."web"."events"`: quoted, therefore **case-sensitive lowercase**. Snowflake stores unquoted names in upper case, so this won't find a table created as `events`. |
| BigQuery `EXTRACT(DAYOFWEEK …)` | BigQuery numbers days 1 (Sunday) to 7. Snowflake's `DAYOFWEEK` with the default `WEEK_START = 0` returns 0 (Sunday) to 6. Every value is off by one. |
| BigQuery `ARRAY_AGG(… LIMIT 5)` | Snowflake's `ARRAY_AGG` syntax has no `LIMIT`. This fails to compile in Snowflake, so at least it fails loudly. |
| Redshift `NVL`, `GETDATE()`, `LISTAGG` | Translated correctly. |

---

Go deeper with the full [Snowflake Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
