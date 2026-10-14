---
description: "Snowflake replication and business continuity: replication and failover groups, refresh modes, data loss modelled, Client Redirect, and a failover runbook."
---

# 07 · Replication, Failover & Business Continuity

Snowflake is built to survive the loss of a data centre within a region. Losing a whole **region** (or a cloud provider's
region-wide outage) is a different event, and surviving it is your design decision: replicate to an account in another region,
decide how much data you can afford to lose, and practise switching over. This lesson covers the objects that do the
replication, the refresh options, what a schedule actually means for data loss, and redirecting clients.

!!! note "What was run"
    Replication needs at least two accounts (and Business Critical Edition for failover), so nothing here was run. The data-loss
    figures come from a small model with stated assumptions, and the SQL was checked against Snowflake's October 2026
    documentation, including features that are new in 2026 (Optimized Refresh and RPO Assurance).

## What can be replicated, on which edition

| Capability | Edition, per the documentation |
|---|---|
| Database and share replication | **all editions** |
| Replication groups with other account objects (users, roles, warehouses, integrations, policies, …) | **Business Critical** or higher |
| **Failover groups** (promote a replica to primary) | **Business Critical** or higher |
| **Client Redirect** (connection URLs that follow the primary) | **Business Critical** or higher |

A **replication group** is a set of objects replicated together to one or more target accounts, read-only there. A **failover
group** is a replication group that can also be **promoted**: the secondary becomes the read-write primary. Both give
**point-in-time consistency** across everything in the group, so a role and the tables it's granted on arrive together.

Not everything is replicated. Per the documentation, **temporary tables**, **external tables** and **hybrid tables** are skipped
(and won't exist after a failover). Dynamic tables and clustering are replicated. Snowflake-managed Iceberg tables need their
external volume replicated too. **Warehouses** arrive suspended. Read the "replication considerations" page for each object type
you depend on. A failover that brings back tables but not the stage integration your pipelines need is only half a recovery.

## Setting it up

On the **source** account:

```sql
USE ROLE ACCOUNTADMIN;
CREATE FAILOVER GROUP prod_fg
  OBJECT_TYPES = USERS, ROLES, WAREHOUSES, RESOURCE MONITORS, DATABASES, INTEGRATIONS, NETWORK POLICIES, ACCOUNT PARAMETERS
  ALLOWED_DATABASES = sales, hr, governance
  ALLOWED_INTEGRATION_TYPES = SECURITY INTEGRATIONS, STORAGE INTEGRATIONS
  ALLOWED_ACCOUNTS = myorg.prod_dr
  REPLICATION_SCHEDULE = '10 MINUTE'
  OPTIMIZED_REFRESH = TRUE;
```

On the **target** account (in another region):

```sql
USE ROLE ACCOUNTADMIN;
CREATE FAILOVER GROUP prod_fg AS REPLICA OF myorg.prod.prod_fg;
ALTER FAILOVER GROUP prod_fg REFRESH;                -- the first, full refresh; afterwards the schedule takes over
SHOW FAILOVER GROUPS;
```

## Three refresh modes

The documentation now describes three modes for failover groups:

| Mode | How refreshes happen | Billing (besides data transfer) |
|---|---|---|
| **Replication Classic** (the default for existing groups) | on `REPLICATION_SCHEDULE` or manual `REFRESH` | Snowflake-provided compute, service type `REPLICATION` |
| **Optimized Refresh** (`OPTIMIZED_REFRESH = TRUE`) | on a schedule of **at most 6 hours** (10 minutes or less recommended); processes only **metadata changes** since the last refresh, so duration follows your rate of change rather than your object count | based on replicated data volume and object changes |
| **RPO Assurance** (`RPO_ASSURANCE = TRUE`, no schedule) | **continuous**, managed by Snowflake, with an RPO target that is backed by an SLA for covered account pairs | as Optimized Refresh |

Table **data** is replicated incrementally in every mode. Optimized Refresh makes the **metadata** pass incremental too, which matters
most for accounts with many databases, tables, roles and grants. RPO Assurance currently needs a request to your account team. In all
modes, **data transfer** between regions is billed on the target account, at the source region's rate, plus the target's storage
for secondary databases. Per the documentation, charges apply even when a refresh fails.

## What a schedule means for data loss

The **recovery point objective (RPO)** is how much committed data you may lose. With a schedule, it isn't the schedule
interval. A refresh copies the primary as it was when the refresh started, and the copy is usable only when the refresh finishes:

```python
# Model (not Snowflake): how much committed data a failover can lose with a scheduled refresh.
# Assumptions: a refresh replicates the primary as of the moment it starts and becomes visible on the secondary
# when it finishes; refreshes start every SCHEDULE minutes, or as soon as the previous one finishes if it overran.
# Failure moment is uniform over a day. Data loss = failure time - snapshot time of the last completed refresh.
def rpo(schedule, duration, horizon=24 * 60, step=0.1):
    refreshes, t = [], 0.0                                   # (snapshot_time, visible_time)
    while t < horizon + 1000:
        refreshes.append((t, t + duration))
        t = max(t + schedule, t + duration)
    losses = []
    f = 120.0                                                # start measuring after a warm-up
    while f < horizon:
        snapshot = max(s for s, v in refreshes if v <= f)
        losses.append(f - snapshot)
        f += step
    return sum(losses) / len(losses), max(losses)

print(f"{'schedule':>9} {'refresh takes':>14} {'avg loss':>9} {'worst loss':>11}")
for schedule, duration in [(60, 5), (60, 30), (10, 2), (10, 8), (10, 25)]:
    avg, worst = rpo(schedule, duration)
    print(f"{schedule:>6} min {duration:>10} min {avg:>6.1f} min {worst:>8.1f} min")
```

```text
 schedule  refresh takes  avg loss  worst loss
    60 min          5 min   35.0 min     65.0 min
    60 min         30 min   60.0 min     90.0 min
    10 min          2 min    7.0 min     12.0 min
    10 min          8 min   13.0 min     18.0 min
    10 min         25 min   37.5 min     50.0 min
```

The worst case is the interval **plus** the refresh duration (65 minutes on an hourly schedule with 5-minute refreshes, not 60).
And when refreshes take longer than the interval (the last row), refreshes run back to back and the worst case becomes twice the
refresh time. That's the case Optimized Refresh addresses by making refresh time depend on change rather than size, and the one
RPO Assurance takes off your hands. Measure your real refresh durations:

```sql
SELECT phase_name, start_time, end_time, DATEDIFF(second, start_time, end_time) AS seconds
FROM TABLE(INFORMATION_SCHEMA.REPLICATION_GROUP_REFRESH_PROGRESS('prod_fg'));
```

## Client Redirect

After a failover, every tool's connection string would still point at the old account. A **connection** object gives clients a
stable URL (`<organization>-<connection>.snowflakecomputing.com`) that Snowflake resolves to whichever account is primary:

```sql
-- source account
CREATE CONNECTION prod_conn;
ALTER CONNECTION prod_conn ENABLE FAILOVER TO ACCOUNTS myorg.prod_dr;

-- target account
CREATE CONNECTION prod_conn AS REPLICA OF myorg.prod.prod_conn;
```

Point BI tools, drivers and pipelines at the connection URL from the start. Changing every client during an outage is exactly the
work you don't want to do then.

## The failover runbook

Promotion happens in the **target** account, which is reachable while the source region is down:

```sql
-- in the target account
USE ROLE ACCOUNTADMIN;
ALTER FAILOVER GROUP prod_fg SUSPEND IMMEDIATE;      -- only if a scheduled refresh is in progress and blocks promotion
ALTER FAILOVER GROUP prod_fg PRIMARY;                -- the replicas become read-write
ALTER CONNECTION prod_conn PRIMARY;                  -- clients now resolve to this account

-- resume what doesn't move by itself
ALTER WAREHOUSE etl_wh RESUME;
ALTER TASK sales.mart.merge_orders RESUME;
```

Per the documentation, promotion fails while a refresh is in progress. `SUSPEND` waits for it, and `SUSPEND IMMEDIATE` cancels a
scheduled one, with a warning that cancelling during the data-download phases can leave an inconsistent state, so know which phase
you're in before choosing. A runbook also needs the parts SQL doesn't cover: who decides to fail over, how pipelines that write to
the old primary are stopped, how you'll **fail back** (the old primary becomes a secondary and must be refreshed first), and a
scheduled **drill** that proves all of it works.

## How It Actually Works

A refresh starts by taking a consistent snapshot of the primary group: the metadata for every object in it, and the list of
micro-partitions each table version references. Because micro-partitions are immutable (Level 1 · 01), the target only needs files
it doesn't already have, so table data is naturally incremental. Snowflake copies those files across regions, applies the metadata,
and switches the secondary to the new snapshot atomically, which is what gives point-in-time consistency. Promotion flips the
group's role. The secondary's objects become writable, and the old primary, when it's back, becomes a secondary that can only
receive refreshes. Client Redirect is a DNS-level indirection: the connection URL resolves to whichever account currently holds
the primary connection.

## Common mistakes

- **Replicating databases but not roles, users and integrations.** The data arrives, and nobody can use it.
- **Equating the schedule with the RPO.** Add the refresh duration, and watch for refreshes that overrun.
- **Relying on temporary, external or hybrid tables** without a plan to recreate them.
- **Hard-coded account URLs** in clients. Use a connection URL.
- **Never testing failover.** A drill is the only proof that the runbook, the grants and the pipelines all work.
- **Forgetting that warehouses and tasks need resuming** on the new primary.

## Exercise

1. List every object your most important pipeline needs (databases, roles, integrations, stages, network policies, tasks), and
   mark which a failover group would bring across and which it wouldn't.
2. With two accounts, replicate one database with a replication group, refresh it twice, and query the refresh history. How long
   did each refresh take, and how much did it transfer?
3. Rerun `rpo_model.py` with your measured durations and a 5-minute schedule. What's your worst-case data loss?
4. Write your failover runbook as a checklist with owners, including fail-back, and schedule a drill.
