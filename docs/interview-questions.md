---
title: "Snowflake Interview Questions and Answers (180 Questions)"
description: "180 Snowflake interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Snowflake Mastery Path."
---

# Snowflake Interview Questions and Answers

**180 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Snowflake Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Snowflake Foundations

### [01 · How Snowflake Is Built: Storage, Compute &amp; Services](level-1/01-architecture.md)

**Q1.** What are Snowflake's three architecture layers?

**Answer:** Storage, compute (virtual warehouses) and cloud services

Storage and compute scale independently.

**Q2.** What is the practical benefit of separating storage from compute?

**Answer:** Several warehouses can query the same data at once without competing, and each can be sized and suspended independently

Workload isolation.

**Q3.** What does the cloud services layer handle?

**Answer:** Authentication, metadata, query parsing and optimisation, and access control

It coordinates the platform.

**Q4.** What kind of storage format does Snowflake use internally?

**Answer:** Compressed, columnar micro-partitions managed by Snowflake

You do not manage files.

### [02 · Accounts, Snowsight, Snowflake CLI &amp; Python](level-1/02-accounts-and-clients.md)

**Q5.** What is a Snowflake account identifier made of?

**Answer:** An organisation and account name (or locator plus region/cloud) that forms the account URL

Used by drivers and the web UI.

**Q6.** Which client is the browser interface?

**Answer:** Snowsight

SnowSQL is the older command-line client; the Snowflake CLI is newer.

**Q7.** Why prefer key-pair or SSO authentication for services?

**Answer:** No shared passwords, and credentials can be rotated or revoked centrally

Service users should not use passwords.

**Q8.** What does a driver connection need at minimum?

**Answer:** Account identifier, user, authenticator or credentials, and a role/warehouse for queries

Set default role and warehouse.

### [03 · Databases, Schemas, Tables &amp; Data Types](level-1/03-objects-and-types.md)

**Q9.** What is the object hierarchy for tables?

**Answer:** Account &gt; database &gt; schema &gt; table

Fully qualified names are db.schema.object.

**Q10.** Which type stores JSON documents?

**Answer:** VARIANT

Also OBJECT and ARRAY.

**Q11.** What is a transient table?

**Answer:** A table with no Fail-safe period and at most 1 day of Time Travel, cheaper for staging data

Persists until dropped, unlike a temporary table.

**Q12.** What does a temporary table do?

**Answer:** Exists only for the session and is dropped when it ends

Good for intermediate results.

### [04 · Virtual Warehouses: Sizing, Suspend &amp; Credits](level-1/04-virtual-warehouses.md)

**Q13.** A Medium warehouse (4 credits/hour) runs for 90 minutes. How many credits are used?

```python
print(4*1.5)
```

**Answer:** 6.0

Credits = rate per hour x hours (per-second billing, 60-second minimum per resume). (Any standard SQL here was run on PostgreSQL 18; Snowflake itself was not used.)

**Q14.** Warehouse sizes double their credit rate each step: XS=1, S=2, M=4, L=8. How many credits per hour for an XL?

```python
print(1*2**4)
```

**Answer:** 16

XL is four doublings from XS. (Any standard SQL here was run on PostgreSQL 18; Snowflake itself was not used.)

**Q15.** What happens to billing when a warehouse is suspended?

**Answer:** Compute credits stop accruing (storage is billed separately)

Use auto-suspend.

**Q16.** When is a larger warehouse most likely to help?

**Answer:** For a single large, complex query that benefits from more compute, not for many small concurrent queries

Concurrency is addressed with multi-cluster warehouses.

### [05 · Loading Data: Stages, File Formats &amp; COPY INTO](level-1/05-loading-copy-into.md)

**Q17.** What does COPY INTO &lt;table&gt; FROM @stage do?

**Answer:** Bulk-loads staged files into a table using a file format

Tracks loaded files to avoid reloading.

**Q18.** What is a stage?

**Answer:** A named location (internal or external) where files are kept for loading and unloading

@~, @%table and named stages.

**Q19.** What does ON_ERROR = CONTINUE do in COPY INTO?

**Answer:** Loads valid rows and skips rows with errors instead of aborting

Check the load results.

**Q20.** How does Snowflake avoid loading the same file twice?

**Answer:** Load metadata tracks file names and ETags for about 64 days

FORCE = TRUE overrides it.

### [06 · Snowflake SQL: Windows, QUALIFY &amp; Extensions](level-1/06-snowflake-sql.md)

**Q21.** What does this standard SQL, which also runs on Snowflake, return?

```sql
create table o(id int, cust text, amt numeric); insert into o values (1,'a',10),(2,'b',20),(3,'a',30),(4,'b',5);

select cust, sum(amt) as total, row_number() over (order by sum(amt) desc) rn from o group by cust order by rn;
```

**Answer:** a,40,1; b,25,2

Aggregates inside window ordering work the same in both systems. The standard-SQL part was run on PostgreSQL 18 and its output checked; Snowflake itself was not used, so any Snowflake-only syntax is untested here.

**Q22.** What does QUALIFY do in Snowflake SQL?

**Answer:** Filters the results of window functions, like HAVING does for aggregates

Other engines need a subquery.

**Q23.** Which function parses a string into a date in Snowflake?

**Answer:** TO_DATE (or TRY_TO_DATE to return NULL on failure)

TRY_ functions avoid errors on bad data.

**Q24.** What does this return (the same logic as Snowflake's IFF/NVL)?

```sql
create table t(x int); insert into t values (1),(null),(3);

select sum(coalesce(x,0)), count(x), count(*) from t;
```

**Answer:** 4,2,3

count(x) ignores NULL; count(*) does not. The standard-SQL part was run on PostgreSQL 18 and its output checked; Snowflake itself was not used, so any Snowflake-only syntax is untested here.

### [07 · Semi-Structured Data: VARIANT &amp; FLATTEN](level-1/07-semi-structured.md)

**Q25.** Which syntax reads a JSON field in Snowflake?

**Answer:** col:path.field::type (for example v:user.name::string)

Colon traverses, double-colon casts.

**Q26.** What does FLATTEN do?

**Answer:** Expands an array or object into rows (used with LATERAL)

Use for nested arrays.

**Q27.** Why cast values from VARIANT?

**Answer:** Values are typed as VARIANT and compare/aggregate better after ::string or ::number

Avoid implicit surprises.

**Q28.** What does this JSON-style extraction return in standard PostgreSQL (the idea Snowflake does with :)?

```sql
create table e(v jsonb); insert into e values ('{"user":{"name":"Ana"},"items":[1,2,3]}');

select v->'user'->>'name', jsonb_array_length(v->'items') from e;
```

**Answer:** Ana,3

Path access plus array length; Snowflake uses v:user.name::string and ARRAY_SIZE(v:items). The standard-SQL part was run on PostgreSQL 18 and its output checked; Snowflake itself was not used, so any Snowflake-only syntax is untested here.

### [08 · Time Travel, UNDROP &amp; Fail-safe](level-1/08-time-travel-failsafe.md)

**Q29.** What is Time Travel?

**Answer:** Querying or restoring data as it was within a retention period (for example AT or BEFORE, UNDROP)

Standard edition default is 1 day; Enterprise can extend up to 90.

**Q30.** What is Fail-safe?

**Answer:** A non-configurable 7-day recovery period after Time Travel ends, accessible only through Snowflake support

Not for routine recovery.

**Q31.** How do you undo an accidental DROP TABLE within retention?

**Answer:** UNDROP TABLE name

Also UNDROP SCHEMA and DATABASE.

**Q32.** A table has 1 day Time Travel plus 7 days Fail-safe. How many days after deletion can Snowflake still potentially recover the data?

```python
print(1+7)
```

**Answer:** 8

Only the Time Travel part is self-service. (Any standard SQL here was run on PostgreSQL 18; Snowflake itself was not used.)

### [09 · Zero-Copy Cloning &amp; Environments](level-1/09-zero-copy-cloning.md)

**Q33.** What does CREATE TABLE ... CLONE copy?

**Answer:** Metadata only at first, sharing the underlying micro-partitions (zero-copy)

Storage grows only as the clone diverges.

**Q34.** What are common uses for clones?

**Answer:** Dev/test copies, safe experiments and point-in-time snapshots

Cheap and fast.

**Q35.** What happens to a source table when you change the clone?

**Answer:** The source is unchanged; modified data is stored separately

Copy-on-write.

**Q36.** Can you clone using Time Travel?

**Answer:** Yes, with AT/BEFORE to clone a past state

Within retention.

### [10 · Project — Load &amp; Analyse a Sales Dataset](level-1/10-project-sales-analytics.md)

**Q37.** What does the sales analytics project practise?

**Answer:** Loading, modelling and querying data end to end with warehouses, stages and SQL analytics

Level 1 capstone.

**Q38.** Which query finds each customer's latest order (a typical analytics pattern)?

```sql
create table o(id int, cust text, ts date); insert into o values (1,'a','2026-01-01'),(2,'a','2026-02-01'),(3,'b','2026-01-15');

select cust, id from (select cust, id, row_number() over (partition by cust order by ts desc) rn from o) x where rn = 1 order by cust;
```

**Answer:** a,2; b,3

In Snowflake you could write this with QUALIFY row_number() = 1. The standard-SQL part was run on PostgreSQL 18 and its output checked; Snowflake itself was not used, so any Snowflake-only syntax is untested here.

**Q39.** Why define a clustering or loading convention early?

**Answer:** Consistent structures keep loads and queries predictable and cheap

Plan before data grows.

**Q40.** What should you check after loading?

**Answer:** Row counts, rejected rows and sample values

Reconcile with the source.

## Level 2 · Performance & Data Engineering

### [01 · Micro-Partitions &amp; Pruning](level-2/01-micro-partitions-pruning.md)

**Q41.** What are micro-partitions?

**Answer:** Contiguous units of 50-500 MB (uncompressed) of columnar data, immutable and with metadata on value ranges

Snowflake maintains them automatically.

**Q42.** What is partition pruning?

**Answer:** Skipping micro-partitions whose min/max metadata shows they cannot match the filter

Reduces scanned data.

**Q43.** A table has 1,000 micro-partitions and a filter matches data in 25 of them. What fraction is scanned?

```python
print(25/1000)
```

**Answer:** 0.025

Good pruning scans a small fraction. (Any standard SQL here was run on PostgreSQL 18; Snowflake itself was not used.)

**Q44.** Why does data order affect pruning?

**Answer:** Well-clustered ranges let metadata exclude many partitions; random order spreads values everywhere

Clustering keys help.

### [02 · Clustering Keys &amp; Automatic Clustering](level-2/02-clustering.md)

**Q45.** When is a clustering key worth considering?

**Answer:** On large tables with queries that filter or join on a few columns and where pruning is poor

It costs credits to maintain.

**Q46.** What does SYSTEM$CLUSTERING_INFORMATION tell you?

**Answer:** How well the table is clustered on the given columns (depth and overlap)

Lower average depth is better.

**Q47.** What is the cost of automatic clustering?

**Answer:** Serverless compute credits for reclustering, plus extra storage

Measure the benefit.

**Q48.** Which columns suit clustering keys?

**Answer:** Columns frequently used in filters with moderate cardinality (for example a date)

Very high or very low cardinality is poor.

### [03 · Query Profile &amp; Performance Tuning](level-2/03-query-profile-tuning.md)

**Q49.** Where do you look for why a query is slow?

**Answer:** The Query Profile (operators, data scanned, spilling, pruning)

Find the dominant operator.

**Q50.** What does 'spilling to remote storage' indicate?

**Answer:** The warehouse ran out of memory for an operation; a larger warehouse or query rewrite may help

Local then remote spilling.

**Q51.** What does a poor partitions scanned / total ratio mean?

**Answer:** The filter is not pruning well

Consider clustering or better predicates.

**Q52.** What is a common cause of exploding joins?

**Answer:** A join on non-unique keys multiplying rows

Check join cardinality.

### [04 · The Three Caches](level-2/04-caching-layers.md)

**Q53.** What does the result cache do?

**Answer:** Returns an identical, still-valid query result without using a warehouse, for up to 24 hours

Invalidated by underlying data changes.

**Q54.** What is the warehouse (local disk) cache?

**Answer:** Cached micro-partition data on the warehouse's storage that speeds repeat scans while it runs

Lost when the warehouse suspends.

**Q55.** What does the metadata cache answer?

**Answer:** Queries such as COUNT(*) or MIN/MAX from metadata without scanning

Cloud services layer.

**Q56.** Why can aggressive auto-suspend hurt performance?

**Answer:** The local cache is lost on suspend, so the next query re-reads data

Balance suspend time and cost.

### [05 · Continuous Loading with Snowpipe](level-2/05-snowpipe.md)

**Q57.** What does Snowpipe do?

**Answer:** Loads files continuously shortly after they arrive in a stage, using serverless compute

Event driven (auto-ingest) or REST.

**Q58.** How is Snowpipe billed?

**Answer:** Per-second serverless compute plus a small per-file overhead

Avoid many tiny files.

**Q59.** What is the recommended file size for ingestion?

**Answer:** Roughly 100-250 MB compressed, avoiding very many tiny files

Balance latency and efficiency.

**Q60.** How can you tell if a pipe loaded a file?

**Answer:** Check COPY_HISTORY or the pipe status functions

Monitor failures.

### [06 · Streams: Change Data Capture](level-2/06-streams-cdc.md)

**Q61.** What is a Snowflake stream?

**Answer:** An object that records changes (inserts, updates, deletes) to a table since it was last consumed

Used for change data capture.

**Q62.** Which metadata columns does a stream expose?

**Answer:** METADATA$ACTION, METADATA$ISUPDATE and METADATA$ROW_ID

An update appears as a delete plus insert with ISUPDATE true.

**Q63.** When does a stream's offset advance?

**Answer:** When it is consumed inside a DML transaction (for example INSERT ... SELECT)

Plain SELECT does not advance it.

**Q64.** What is a risk of leaving a stream unconsumed?

**Answer:** It can go stale if the source's retention window passes

Consume regularly.

### [07 · Tasks &amp; Task Graphs](level-2/07-tasks.md)

**Q65.** What is a task?

**Answer:** A scheduled (or triggered) SQL statement or procedure call run by Snowflake

Schedule via minutes or CRON.

**Q66.** How do you build a dependency chain of tasks?

**Answer:** Use AFTER &lt;predecessor&gt; to form a DAG under a root task

Resume child tasks before the root.

**Q67.** What does WHEN SYSTEM$STREAM_HAS_DATA('s') do in a task?

**Answer:** Runs the task only if the stream has changes

Saves credits.

**Q68.** Why must tasks be explicitly resumed?

**Answer:** New tasks are created suspended

ALTER TASK ... RESUME.

### [08 · Dynamic Tables](level-2/08-dynamic-tables.md)

**Q69.** What does a dynamic table do?

**Answer:** Materialises a query's result and keeps it refreshed within a TARGET_LAG

Declarative pipelines.

**Q70.** What does TARGET_LAG = '5 minutes' mean?

**Answer:** Snowflake tries to keep the data no more than about 5 minutes behind its sources

A freshness goal.

**Q71.** What is an advantage over streams and tasks?

**Answer:** You declare the result and Snowflake manages refresh, dependencies and incremental logic

Less plumbing.

**Q72.** When might streams and tasks still be preferable?

**Answer:** When you need fine-grained custom logic or control over processing

Choose the tool to fit.

### [09 · Unloading, External Tables &amp; Iceberg](level-2/09-unloading-iceberg.md)

**Q73.** What does COPY INTO &lt;location&gt; do?

**Answer:** Unloads query results to files in a stage or cloud storage

Formats include CSV, JSON and Parquet.

**Q74.** Why unload as Parquet?

**Answer:** A compact, columnar, widely supported format for lakes and engines

Good for sharing with other tools.

**Q75.** What is an Iceberg table in Snowflake?

**Answer:** A table in the open Apache Iceberg format that other engines can also read

Interoperability.

**Q76.** What is a benefit of open table formats?

**Answer:** Avoid lock-in and let several engines share the same data

Governance still matters.

### [10 · Project — An Incremental ELT Pipeline](level-2/10-project-incremental-pipeline.md)

**Q77.** What does the incremental pipeline project combine?

**Answer:** Stage and Snowpipe, streams, tasks (or dynamic tables) and merges

An end-to-end flow.

**Q78.** Which MERGE-style upsert logic is shown here (standard SQL on PostgreSQL)?

```sql
create table t(id int primary key, v text); insert into t values (1,'old'),(2,'keep'); create table s(id int, v text); insert into s values (1,'new'),(3,'add');

insert into t select id, v from s on conflict (id) do update set v = excluded.v; select * from t order by id;
```

**Answer:** 1,new; 2,keep; 3,add

Snowflake uses MERGE INTO ... WHEN MATCHED ... WHEN NOT MATCHED; PostgreSQL's ON CONFLICT achieves the same upsert. The standard-SQL part was run on PostgreSQL 18 and its output checked; Snowflake itself was not used, so any Snowflake-only syntax is untested here.

**Q79.** Why make the load idempotent?

**Answer:** Re-running after a failure must not duplicate rows

Use MERGE with keys.

**Q80.** How do you monitor the pipeline?

**Answer:** Task history, pipe status, stream staleness and row-count checks

Alert on failures.

## Level 3 · Security, Governance & Sharing

### [01 · Access Control: Roles, Privileges &amp; Hierarchies](level-3/01-rbac.md)

**Q81.** What is RBAC in Snowflake?

**Answer:** Privileges are granted to roles and roles are granted to users

Roles can inherit from roles.

**Q82.** Which system role manages users and roles?

**Answer:** SECURITYADMIN (USERADMIN creates users and roles)

SYSADMIN owns objects.

**Q83.** Why avoid using ACCOUNTADMIN for daily work?

**Answer:** It is too powerful; reserve it for account-level tasks

Least privilege.

**Q84.** What are future grants?

**Answer:** Grants that apply automatically to objects created later in a schema or database

Reduce manual work.

### [02 · Authentication: MFA, Key Pairs, OAuth &amp; SSO](level-3/02-authentication.md)

**Q85.** What is the recommended authentication for human users?

**Answer:** SSO with MFA

Add network policies.

**Q86.** What is key-pair authentication for?

**Answer:** Service accounts and drivers without passwords

Rotate keys.

**Q87.** What do you do for a service account that must not log in interactively?

**Answer:** Use TYPE = SERVICE (or restrict to key-pair and network policy)

Limit exposure.

**Q88.** Why enforce MFA for privileged roles?

**Answer:** A stolen password alone cannot grant access

Phishing resistance.

### [03 · Network Policies &amp; Private Connectivity](level-3/03-network-security.md)

**Q89.** What does a network policy do?

**Answer:** Allows or blocks access by IP address ranges

Applies at account or user level.

**Q90.** What is private connectivity (such as AWS PrivateLink) for?

**Answer:** Reaching Snowflake without traversing the public internet

A higher-security option.

**Q91.** What is a risk of a too-narrow IP allow list?

**Answer:** Locking out legitimate users or services

Test and keep a break-glass path.

**Q92.** Where else should access controls be applied?

**Answer:** Roles, masking and row policies, not only the network

Defence in depth.

### [04 · Masking &amp; Row Access Policies](level-3/04-masking-row-access.md)

**Q93.** What does a masking policy do?

**Answer:** Returns masked or full column values depending on the querying role

Applied to columns.

**Q94.** What does a row access policy do?

**Answer:** Filters which rows a role can see

Based on role or mapping table.

**Q95.** The same idea as a role-based mask, on PostgreSQL: what does the analyst see?

```sql
create table c(name text, email text); insert into c values ('ana','ana@x.com'); create view c_masked as select name, case when current_user = 'postgres' then email else '***' end as email from c;

select * from c_masked;
```

**Answer:** ana,ana@x.com

Snowflake masking policies evaluate CURRENT_ROLE() similarly; here the current user is the admin so the real value shows. The standard-SQL part was run on PostgreSQL 18 and its output checked; Snowflake itself was not used, so any Snowflake-only syntax is untested here.

**Q96.** Why prefer policies over copies of masked tables?

**Answer:** One governed source avoids drift and duplicate data

Central policy.

### [05 · Object Tagging &amp; Data Classification](level-3/05-tags-classification.md)

**Q97.** What are object tags?

**Answer:** Key-value metadata attached to objects for governance and cost allocation

Tag-based masking possible.

**Q98.** What does data classification do?

**Answer:** Detects and tags likely sensitive columns such as emails or national IDs

Review the results.

**Q99.** How can tags help cost control?

**Answer:** Tag warehouses by team to attribute credits

Use account usage views.

**Q100.** Why combine tags with masking?

**Answer:** Apply protection automatically to every column carrying a sensitive tag

Scales governance.

### [06 · Secure Views &amp; Secure Functions](level-3/06-secure-views-udfs.md)

**Q101.** What is a secure view?

**Answer:** A view whose definition and optimisation internals are hidden, preventing data leakage through the optimiser

Required for sharing.

**Q102.** Why are secure views required for shares?

**Answer:** To avoid exposing underlying data through query optimisation side channels

Consumers see only the view.

**Q103.** What is a SQL UDF?

**Answer:** A user-defined function written in SQL that you call in queries

Scalar or table functions.

**Q104.** What do you give consumers of a secure view?

**Answer:** SELECT on the view only

Least privilege.

### [07 · Secure Data Sharing &amp; the Marketplace](level-3/07-data-sharing.md)

**Q105.** What is Snowflake data sharing?

**Answer:** Granting another account read access to live data without copying it

Providers and consumers.

**Q106.** Does a consumer pay for the provider's storage?

**Answer:** No; the consumer pays for their own compute to query it

The data is not copied.

**Q107.** What are reader accounts for?

**Answer:** Providing access to consumers who do not have a Snowflake account

Provider pays compute.

**Q108.** What is the Marketplace?

**Answer:** A catalogue of shared datasets and apps

Listings.

### [08 · Encryption, Key Management &amp; Compliance](level-3/08-encryption-compliance.md)

**Q109.** What is encryption at rest in Snowflake?

**Answer:** All data is encrypted by default with a hierarchical key model

Tri-Secret Secure adds customer-managed keys.

**Q110.** Why rotate keys?

**Answer:** Limit exposure if a key is compromised

Automatic rotation exists.

**Q111.** Which compliance frameworks does Snowflake support (examples)?

**Answer:** SOC 2, ISO 27001 and others depending on edition and region

Check the current compliance documentation.

**Q112.** What is your responsibility under shared responsibility?

**Answer:** Configuring access, policies and data handling correctly

Misconfiguration is the main risk.

### [09 · Auditing with ACCOUNT_USAGE &amp; Access History](level-3/09-auditing-account-usage.md)

**Q113.** Where do you audit who queried what?

**Answer:** ACCOUNT_USAGE views such as QUERY_HISTORY and ACCESS_HISTORY

They have some latency.

**Q114.** What is a limitation of ACCOUNT_USAGE views?

**Answer:** Data can lag by up to a few hours and is retained for about a year

Use INFORMATION_SCHEMA for near real time.

**Q115.** What does ACCESS_HISTORY show?

**Answer:** Which objects and columns were accessed by queries

Useful for governance.

**Q116.** What should you alert on?

**Answer:** Failed logins, privilege grants and unusual data exports

Practical detection.

### [10 · Project — A Governed Multi-Team Platform](level-3/10-project-governed-platform.md)

**Q117.** What does the governed platform project assemble?

**Answer:** RBAC, masking, row policies, tags, secure sharing and auditing

Governance at scale.

**Q118.** Which comes first when designing access?

**Answer:** Define roles by job function and grant least privilege

Role hierarchy.

**Q119.** How do you prove controls work?

**Answer:** Test with different roles and review audit logs

Evidence.

**Q120.** Why document the role model?

**Answer:** So reviewers and new admins understand and maintain it

Operational clarity.

## Level 4 · Building, Scaling & Operating

### [01 · UDFs &amp; Stored Procedures](level-4/01-udfs-procedures.md)

**Q121.** What is a stored procedure for?

**Answer:** Procedural logic with control flow, executed with the caller's or owner's rights

Languages include SQL scripting, JavaScript and Python.

**Q122.** What do UDFs return compared with procedures?

**Answer:** UDFs return values used inside queries; procedures perform actions and can return a result

Different roles.

**Q123.** What does EXECUTE AS OWNER mean?

**Answer:** The procedure runs with its owner's privileges

Careful with privilege escalation.

**Q124.** Why write set-based SQL instead of loops in procedures?

**Answer:** Set-based statements use the engine's parallelism

Row-by-row is slow.

### [02 · Snowpark for Python](level-4/02-snowpark-python.md)

**Q125.** What is Snowpark?

**Answer:** A developer framework for running DataFrame-style code (Python, Java, Scala) inside Snowflake

Code is pushed to the data.

**Q126.** Where does Snowpark Python code execute?

**Answer:** On Snowflake's compute, translated to SQL or run in sandboxed Python

Avoid pulling data to the client.

**Q127.** What do lazy DataFrames mean?

**Answer:** Operations build a query that runs only when an action like collect or show is called

Similar to Spark.

**Q128.** Why avoid collect() on big tables?

**Answer:** It brings all rows to the client and defeats pushdown

Aggregate in Snowflake.

### [03 · Python UDFs, Vectorized UDFs &amp; UDTFs](level-4/03-python-udfs.md)

**Q129.** What can a Python UDF do?

**Answer:** Run Python logic per row inside a query, with approved packages

Packages come from the Anaconda channel.

**Q130.** What is a vectorised UDF?

**Answer:** A UDF that processes batches of rows (such as pandas) for better performance

Reduces per-row overhead.

**Q131.** Why prefer built-in SQL functions over Python UDFs when possible?

**Answer:** They are faster and optimised

Use UDFs where SQL cannot.

**Q132.** How are package dependencies handled?

**Answer:** Declared with PACKAGES or imported from a stage

Sandboxed environment.

### [04 · Cortex AI Functions](level-4/04-cortex-ai.md)

**Q133.** What is Snowflake Cortex?

**Answer:** AI functions and services (such as LLM functions and search) available in SQL

Use under governance.

**Q134.** What should you consider before sending data to an LLM function?

**Answer:** Data governance, cost and where processing occurs

Check policies.

**Q135.** Why validate LLM outputs in pipelines?

**Answer:** They can be wrong or inconsistent; add checks and sampling

Human review for critical use.

**Q136.** How can you control Cortex cost?

**Answer:** Limit rows and models, monitor usage and set budgets

Credits apply.

### [05 · Cost Management: Monitors, Budgets &amp; Attribution](level-4/05-cost-management.md)

**Q137.** Which items drive Snowflake cost?

**Answer:** Compute (credits), storage and data transfer/serverless features

Compute is usually the largest.

**Q138.** What is a resource monitor?

**Answer:** A guardrail that notifies or suspends warehouses when credit thresholds are reached

Set per warehouse or account.

**Q139.** What is the cheapest quick win for idle compute?

**Answer:** Set a short AUTO_SUSPEND and enable AUTO_RESUME

Avoid paying for idle.

**Q140.** A Small warehouse (2 credits/hour) left running 24/7 for 30 days uses how many credits?

```python
print(2*24*30)
```

**Answer:** 1440

That is why auto-suspend matters. (Any standard SQL here was run on PostgreSQL 18; Snowflake itself was not used.)

### [06 · Warehouse Scaling: Up, Out &amp; Adaptive](level-4/06-warehouse-scaling.md)

**Q141.** What is multi-cluster warehouse scaling for?

**Answer:** Handling concurrency by adding clusters when queries queue

Scale out versus scale up.

**Q142.** When do you scale up (bigger size)?

**Answer:** For slow individual queries limited by compute or memory

Test sizes.

**Q143.** What is the Standard vs Economy scaling policy?

**Answer:** Standard favours starting clusters to avoid queuing; Economy favours conserving credits

Choose by workload.

**Q144.** How do you right-size?

**Answer:** Measure query times and queuing at different sizes and choose the cheapest that meets the SLA

Larger can finish faster at similar cost.

### [07 · Replication, Failover &amp; Business Continuity](level-4/07-replication-bcdr.md)

**Q145.** What does database replication provide?

**Answer:** A read-only copy in another region or account for DR and sharing

Refresh on a schedule.

**Q146.** What does failover add?

**Answer:** Promoting the secondary to primary during an outage

Plan and test it.

**Q147.** What are RPO and RTO?

**Answer:** Acceptable data loss and acceptable recovery time

Drive the design.

**Q148.** Why test failover regularly?

**Answer:** An untested plan may fail when needed

Run drills.

### [08 · dbt &amp; CI/CD for Snowflake](level-4/08-dbt-cicd.md)

**Q149.** What does dbt do with Snowflake?

**Answer:** Transforms data with version-controlled SQL models, tests and documentation

Compiled to SQL.

**Q150.** What should CI do for a dbt project?

**Answer:** Build changed models in a clone or dev schema and run tests before merging

Zero-copy clones make this cheap.

**Q151.** How do you promote changes safely?

**Answer:** Pull request, review, CI checks, then deploy via a pipeline

Infrastructure as code.

**Q152.** Why use separate roles for CI and production?

**Answer:** Least privilege and auditability

Limit blast radius.

### [09 · Migrating to Snowflake](level-4/09-migration.md)

**Q153.** What is the first step of a migration to Snowflake?

**Answer:** Inventory sources, dependencies and requirements, then plan phases

Know what you have.

**Q154.** Why validate migrated data?

**Answer:** Row counts and checksums confirm nothing was lost or changed

Reconcile before cutover.

**Q155.** What is a common migration risk?

**Answer:** Dialect differences in SQL and procedural logic

Test code conversion.

**Q156.** How do you reduce cutover risk?

**Answer:** Run old and new in parallel and cut over by workload

Phased.

### [10 · Capstone — A Cost-Aware Data Platform](level-4/10-capstone-cost-aware-platform.md)

**Q157.** What is the capstone's goal?

**Answer:** A cost-aware, governed Snowflake platform design with pipelines, security and monitoring

Bring Level 4 together.

**Q158.** Which evidence shows cost awareness?

**Answer:** Resource monitors, tagged warehouses, auto-suspend and usage dashboards

Measure.

**Q159.** How do you handle residual risk?

**Answer:** Document it and decide to accept, mitigate or transfer

Risk register.

**Q160.** What belongs in the README?

**Answer:** Architecture, roles, pipelines, runbooks and costs

Handover.

## Mixed questions across all levels

**Q161.** What does this return?

```sql
create table t(id int, grp text, v int); insert into t values (1,'a',10),(2,'a',20),(3,'b',5);

select grp, sum(v), avg(v)::int, count(*) from t group by grp order by grp;
```

**Answer:** a,30,15,2; b,5,5,1

Grouped aggregates; the same in Snowflake. The standard-SQL part was run on PostgreSQL 18 and its output checked; Snowflake itself was not used, so any Snowflake-only syntax is untested here.

**Q162.** What does this window query return?

```sql
create table t(d int, v int); insert into t values (1,10),(2,20),(3,30);

select d, sum(v) over (order by d) running, lag(v) over (order by d) prev from t order by d;
```

**Answer:** 1,10,; 2,30,10; 3,60,20

A running sum and the previous row's value (NULL first). The standard-SQL part was run on PostgreSQL 18 and its output checked; Snowflake itself was not used, so any Snowflake-only syntax is untested here.

**Q163.** What does this NULL-handling query return?

```sql
create table t(x int); insert into t values (1),(null),(3);

select count(*), count(x), sum(x), coalesce(max(x),0), avg(x)::numeric(3,1) from t;
```

**Answer:** 3,2,4,3,2.0

NULLs are ignored by aggregates; count(*) counts rows. The standard-SQL part was run on PostgreSQL 18 and its output checked; Snowflake itself was not used, so any Snowflake-only syntax is untested here.

**Q164.** A table is 500 micro-partitions and your filter touches 20. What does pruning mean for cost?

**Answer:** Only the matching partitions are scanned, so the query reads far less data

Min/max metadata excludes the rest.

**Q165.** Why is auto-suspend a main cost control?

**Answer:** Compute credits accrue only while a warehouse runs, so idle warehouses should suspend

Per-second billing with a 60-second minimum on resume.

**Q166.** An XL warehouse (16 credits/hour) runs 15 minutes. How many credits (per-second billing)?

```python
print(16*0.25)
```

**Answer:** 4.0

Rate x hours. (Any standard SQL here was run on PostgreSQL 18; Snowflake itself was not used.)

**Q167.** Which cache can answer an identical repeated query without using a warehouse?

**Answer:** The result cache (valid for 24 hours if data is unchanged)

Cloud services return the stored result.

**Q168.** What is the difference between Time Travel and Fail-safe?

**Answer:** Time Travel is user-accessible recovery within the retention period; Fail-safe is a 7-day last-resort period handled by Snowflake support

Fail-safe is not configurable or for routine recovery.

**Q169.** Why is a zero-copy clone cheap?

**Answer:** It shares existing micro-partitions and stores only changes made afterwards

Ideal for dev/test.

**Q170.** A stream shows an updated row as which pair of records?

**Answer:** A DELETE and an INSERT with METADATA$ISUPDATE = TRUE

Consume streams in DML to advance the offset.

**Q171.** Which object keeps a result continuously refreshed within a freshness goal?

**Answer:** A dynamic table with TARGET_LAG

Declarative pipelines.

**Q172.** Why prefer role-based masking policies over duplicate masked tables?

**Answer:** One governed table avoids drift and extra storage while each role sees what it is allowed to

Policies attach to columns.

**Q173.** Why are secure views used for data sharing?

**Answer:** They hide definitions and prevent optimiser side-channel leakage of data consumers shouldn't see

A requirement for shares.

**Q174.** What should a service account use for authentication?

**Answer:** Key-pair (or OAuth) with a network policy, not a shared password

Rotate keys.

**Q175.** Where do you audit which columns were accessed by whom?

**Answer:** ACCOUNT_USAGE.ACCESS_HISTORY (with QUERY_HISTORY)

Views have some latency.

**Q176.** When should you scale a warehouse out (multi-cluster) instead of up?

**Answer:** When many concurrent queries queue, not when a single query is slow

Scale up for single heavy queries.

**Q177.** What does Snowpark pushdown mean for large DataFrames?

**Answer:** Operations are translated to SQL and run in Snowflake instead of pulling data to the client

Avoid collect() on big tables.

**Q178.** How do you make an incremental load safe to re-run?

**Answer:** Make it idempotent with MERGE on a key so duplicates and re-processing change nothing

Verify with a second run.

**Q179.** What is the right use of Cortex or LLM functions in a pipeline?

**Answer:** With governance, cost limits and validation of outputs, since results can be wrong

Sample and monitor.

**Q180.** What is the safest migration cutover approach?

**Answer:** Run old and new in parallel, reconcile results and cut over workload by workload

Validate with row counts and checksums.

## Practical coding questions

### Sales analytics with window functions

Write three analytics queries over a small sales table: monthly revenue with a running total and share of region, the best rep per region and the month-over-month change. These run on PostgreSQL here because Snowflake is not available to me; the SQL is standard, and where Snowflake offers QUALIFY the comments say so.

Starter code:

```
create table sales(id int, region text, rep text, sold_on date, amount numeric);
insert into sales values
 (1,'north','ana','2026-01-05',120),(2,'north','bo','2026-01-09',80),(3,'north','ana','2026-02-02',200),
 (4,'south','cy','2026-01-07',150),(5,'south','di','2026-01-20',90),(6,'south','cy','2026-02-11',60),
 (7,'north','bo','2026-02-15',140),(8,'south','di','2026-02-28',300);

-- TODO 1: monthly revenue per region (to_char(sold_on,'YYYY-MM')) with a running total per region and each month's % of the region total (1 decimal); order by region, month
-- TODO 2: the best rep per region by total amount (ties by rep name) - Snowflake would use QUALIFY; here use a subquery with row_number()
-- TODO 3: month-over-month change in revenue per region using lag(); the first month's change is NULL (printed as empty)
```

Expected output:

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

??? success "Show a solution"

    ```
    create table sales(id int, region text, rep text, sold_on date, amount numeric);
    insert into sales values
     (1,'north','ana','2026-01-05',120),(2,'north','bo','2026-01-09',80),(3,'north','ana','2026-02-02',200),
     (4,'south','cy','2026-01-07',150),(5,'south','di','2026-01-20',90),(6,'south','cy','2026-02-11',60),
     (7,'north','bo','2026-02-15',140),(8,'south','di','2026-02-28',300);

    -- 1. monthly revenue per region with running total and share of the region's total
    select region, to_char(sold_on,'YYYY-MM') as month, sum(amount) as revenue,
           sum(sum(amount)) over (partition by region order by to_char(sold_on,'YYYY-MM')) as running,
           round(100 * sum(amount) / sum(sum(amount)) over (partition by region), 1) as pct_of_region
    from sales group by region, to_char(sold_on,'YYYY-MM')
    order by region, month;

    -- 2. best rep per region (the Snowflake way would be QUALIFY row_number() = 1)
    select region, rep, total from (
      select region, rep, sum(amount) as total,
             row_number() over (partition by region order by sum(amount) desc, rep) as rn
      from sales group by region, rep
    ) x where rn = 1 order by region;

    -- 3. month-over-month change for each region
    select region, month, revenue, revenue - lag(revenue) over (partition by region order by month) as change from (
      select region, to_char(sold_on,'YYYY-MM') as month, sum(amount) as revenue from sales group by 1, 2
    ) m order by region, month;
    ```

    Window functions can wrap aggregates (sum(sum(amount)) over ...), and row_number() in a subquery is exactly what QUALIFY expresses in Snowflake. lag() gives the previous month, NULL for the first. Verified by running it on PostgreSQL 18; not run on Snowflake.

### An idempotent change-capture upsert

Simulate a stream plus MERGE: find the new and changed staging rows, apply them to the target only when values really differ and prove a second run finds nothing left to do. PostgreSQL's ON CONFLICT stands in for Snowflake's MERGE; Snowflake itself was not used.

Starter code:

```
create table target(id int primary key, name text, plan text, updated date);
insert into target values (1,'ana','free','2026-01-01'),(2,'bo','pro','2026-01-01'),(3,'cy','free','2026-01-01');
create table staging(id int, name text, plan text, updated date);
insert into staging values (2,'bo','free','2026-02-01'),(3,'cy','free','2026-02-01'),(4,'di','pro','2026-02-01');

-- TODO 1: list the changes a stream would report: staging rows that are new ("insert") or whose (name, plan) differ from target ("update"), ordered by id
-- TODO 2: apply them idempotently with INSERT ... ON CONFLICT (Snowflake: MERGE), changing a row only when (name, plan) really differ, then show target ordered by id
-- TODO 3: count staging rows that still differ from target (should be 0 on a re-run)
```

Expected output:

```text
2 | update
4 | insert
1 | ana | free | 2026-01-01
2 | bo | free | 2026-02-01
3 | cy | free | 2026-01-01
4 | di | pro | 2026-02-01
0
```

??? success "Show a solution"

    ```
    create table target(id int primary key, name text, plan text, updated date);
    insert into target values (1,'ana','free','2026-01-01'),(2,'bo','pro','2026-01-01'),(3,'cy','free','2026-01-01');
    create table staging(id int, name text, plan text, updated date);
    insert into staging values (2,'bo','free','2026-02-01'),(3,'cy','free','2026-02-01'),(4,'di','pro','2026-02-01');

    -- 1. the changes a stream would report: new rows and rows whose values differ
    select s.id, case when t.id is null then 'insert' else 'update' end as action
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
    ```

    IS DISTINCT FROM compares row values null-safely, so unchanged rows are not rewritten and the pending count drops to zero. In Snowflake you would write MERGE INTO ... WHEN MATCHED AND (...) THEN UPDATE WHEN NOT MATCHED THEN INSERT. Verified by running it on PostgreSQL 18; not run on Snowflake.

---

More free practice: the [Snowflake Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
