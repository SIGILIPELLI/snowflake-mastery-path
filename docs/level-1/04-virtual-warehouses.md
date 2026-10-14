---
description: "Snowflake warehouses: sizes and credit rates, per-second billing with a 60-second minimum, auto-suspend modelled, resizing, timeouts and workload split."
---

# 04 · Virtual Warehouses: Sizing, Suspend & Credits

A virtual warehouse is where your money goes. Storage is cheap and cloud services are usually free (lesson 01), but
every second a warehouse runs costs credits, whether it's working or idle. Getting warehouses right is mostly about
two settings, **size** and **auto-suspend**, and one habit: **a warehouse per workload**.

## Creating a warehouse

```sql
CREATE WAREHOUSE IF NOT EXISTS learn_wh
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE
  STATEMENT_TIMEOUT_IN_SECONDS = 900
  COMMENT = 'Labs for the Snowflake Mastery Path';
```

The defaults, from Snowflake's `CREATE WAREHOUSE` and parameter reference, show why it's worth setting these
explicitly:

| Property | Default | Why you might change it |
|---|---|---|
| `WAREHOUSE_SIZE` | `XSMALL` | start small, then size from evidence |
| `AUTO_SUSPEND` | `600` seconds | 10 idle minutes are billed after every burst of work |
| `AUTO_RESUME` | `TRUE` | keep it on, so queries start the warehouse |
| `INITIALLY_SUSPENDED` | `FALSE` | otherwise creating the warehouse starts billing immediately |
| `STATEMENT_TIMEOUT_IN_SECONDS` | `172800` (2 days) | a runaway query can run for two days |
| `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` | `0` (no timeout) | queued queries wait forever |
| `MAX_CONCURRENCY_LEVEL` | `8` | how many queries run at once before others queue |

A two-day statement timeout on an X-Large warehouse is 768 credits for one forgotten cross join. Set a timeout that
matches the workload on every warehouse, or at least on the account and the warehouses users can reach.

## Sizes and credits

For standard (Gen1) warehouses, from Snowflake's documentation:

| Size | Credits/hour | | Size | Credits/hour |
|---|---|---|---|---|
| X-Small | 1 | | 2X-Large | 32 |
| Small | 2 | | 3X-Large | 64 |
| Medium | 4 | | 4X-Large | 128 |
| Large | 8 | | 5X-Large | 256 |
| X-Large | 16 | | 6X-Large | 512 |

Each size has twice the compute of the one below and costs twice as much per hour. Billing is **per second**, with a
**60-second minimum each time the warehouse starts** (resumes).

**Gen2** standard warehouses run on newer hardware, consume more credits per hour than Gen1 (third-party guides cite
about 1.35× on AWS and 1.25× on Azure; the authoritative rates are in Snowflake's Service Consumption Table), and aim to
finish work faster. The Gen1/Gen2 situation is changing. In October 2026 Snowflake's warehouse overview page said Gen2
"isn't the default for new standard warehouses", while the `CREATE WAREHOUSE` reference lists a `GENERATION` property
defaulting to `'2'`, the Gen2 page says new standard warehouses default to Gen2 unless it isn't available in your region,
and the default `RESOURCE_CONSTRAINT` depends on your region and when your organization was created. After creating a warehouse, check what you actually got:

```sql
SHOW WAREHOUSES LIKE 'LEARN_WH';
```

The output includes the size, state, auto-suspend, and generation or resource constraint columns. Level 4 · 06 compares
the generations and the newer warehouse types.

## Why doubling the size often costs nothing extra

For a query that parallelizes well (a big scan and aggregation), a warehouse twice the size finishes in roughly half
the time, at twice the rate, so it costs about the same and you get the answer sooner. That's the most counter-intuitive
and useful fact about Snowflake sizing. It breaks down when:

- the query is small. An X-Small already finishes in a second or two, and a bigger warehouse just costs more for the same
  60-second minimum;
- the work doesn't parallelize (one huge sort, a skewed join key, a single-threaded UDF);
- the query is waiting on something else, such as a lock, queueing, or a slow external function.

So size **per workload**, by testing. Run the real job at two sizes and compare elapsed time and credits in the query
history (Level 4 · 05). If doubling the size halves the time, keep the bigger one. If it doesn't, go back down.

## Auto-suspend: the setting that matters most

How long a warehouse keeps running after its last query is pure cost. Here is a model of the billing rules documented
above, applied to two workloads (Python, run locally; a model, not measurements from a Snowflake account):

```python
CREDITS_PER_HOUR = {"XSMALL": 1, "SMALL": 2, "MEDIUM": 4, "LARGE": 8, "XLARGE": 16}

def billed_seconds(queries, auto_suspend):
    """queries: list of (start_second, duration_seconds), sorted. Returns (billed_seconds, resumes)."""
    billed, resumes = 0, 0
    run_start = run_end = None          # current running interval of the warehouse
    for start, dur in queries:
        if run_end is not None and start <= run_end + auto_suspend:
            run_end = max(run_end, start + dur)         # still running (or within idle window)
        else:
            if run_start is not None:
                billed += max(60, (run_end + auto_suspend) - run_start)
            run_start, run_end = start, start + dur; resumes += 1
    billed += max(60, (run_end + auto_suspend) - run_start)
    return billed, resumes

dashboard = [(i * 90, 4) for i in range(40)]      # 40 queries of 4 s, one every 90 s
batch = [(i * 300, 300) for i in range(6)]        # 6 queries of 5 min, back to back

for name, wl in (("dashboard: 40 x 4 s queries, one every 90 s", dashboard),
                 ("batch: 6 x 5 min queries back to back", batch)):
    busy = sum(d for _, d in wl)
    print(name, f"(busy time {busy} s)")
    for susp in (60, 300, 600):
        secs, resumes = billed_seconds(wl, susp)
        print(f"   AUTO_SUSPEND={susp:>3}s  billed {secs:>5} s  resumes {resumes:>2}  "
              f"XSMALL {secs/3600*1:.3f} cr   MEDIUM {secs/3600*4:.3f} cr")
```

```text
dashboard: 40 x 4 s queries, one every 90 s (busy time 160 s)
   AUTO_SUSPEND= 60s  billed  2560 s  resumes 40  XSMALL 0.711 cr   MEDIUM 2.844 cr
   AUTO_SUSPEND=300s  billed  3814 s  resumes  1  XSMALL 1.059 cr   MEDIUM 4.238 cr
   AUTO_SUSPEND=600s  billed  4114 s  resumes  1  XSMALL 1.143 cr   MEDIUM 4.571 cr
batch: 6 x 5 min queries back to back (busy time 1800 s)
   AUTO_SUSPEND= 60s  billed  1860 s  resumes  1  XSMALL 0.517 cr   MEDIUM 2.067 cr
   AUTO_SUSPEND=300s  billed  2100 s  resumes  1  XSMALL 0.583 cr   MEDIUM 2.333 cr
   AUTO_SUSPEND=600s  billed  2400 s  resumes  1  XSMALL 0.667 cr   MEDIUM 2.667 cr
```

The dashboard did **160 seconds of work** and was billed for **2,560 to 4,114 seconds**: 16 to 26 times its busy time,
depending only on auto-suspend. With the default 600 seconds, it never suspended during the hour. With 60 seconds it
suspended after every query and paid the 60-second minimum 40 times, which was still the cheapest option here.

The trade-off the model doesn't show is the **cache**. A suspended warehouse loses its local SSD cache of table data, so
the first queries after a resume read everything from object storage again (Level 2 · 04). For dashboards hitting the
same tables, a few minutes of auto-suspend can pay for itself in speed. For ad-hoc and batch work, 60 seconds is a good
default. Note too that Snowflake checks for idleness periodically, so actual suspension can happen somewhat later than
the exact setting.

## Resizing, suspending, monitoring

```sql
ALTER WAREHOUSE learn_wh SET WAREHOUSE_SIZE = 'MEDIUM';   -- running queries finish on the old size
ALTER WAREHOUSE learn_wh SUSPEND;
ALTER WAREHOUSE learn_wh RESUME IF SUSPENDED;
ALTER WAREHOUSE learn_wh SET AUTO_SUSPEND = 60 STATEMENT_TIMEOUT_IN_SECONDS = 900;

SHOW WAREHOUSES;
SELECT * FROM TABLE(INFORMATION_SCHEMA.WAREHOUSE_METERING_HISTORY(DATEADD('day', -7, CURRENT_DATE())));
```

Resizing takes effect for **new** queries. Queries already running finish on the existing compute, and queued queries
pick up the new size. A larger size is a good tool for one heavy job: resize, run it, resize back.

## One warehouse per workload

```sql
CREATE WAREHOUSE IF NOT EXISTS etl_wh WAREHOUSE_SIZE = 'LARGE'  AUTO_SUSPEND = 60  INITIALLY_SUSPENDED = TRUE;
CREATE WAREHOUSE IF NOT EXISTS bi_wh  WAREHOUSE_SIZE = 'SMALL'  AUTO_SUSPEND = 300 INITIALLY_SUSPENDED = TRUE;
CREATE WAREHOUSE IF NOT EXISTS ds_wh  WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60  INITIALLY_SUSPENDED = TRUE;
```

Separate warehouses give each workload its own performance (the ETL load can't slow the dashboards), its own cost line
in the bill, its own timeout and auto-suspend, and its own access control (Level 3 · 01: grant `USAGE` on `bi_wh` only to
the BI role).

## How It Actually Works

A warehouse is a set of compute nodes that Snowflake provisions from the cloud provider when the warehouse starts or
resumes, drawn from pools Snowflake keeps warm, which is why resuming usually takes seconds rather than minutes. Each
size step doubles the node resources. A query's plan is split into parallel work over the micro-partitions it must
scan, so more nodes means more partitions processed at once, which is the source of "twice the size, half the time"
for scan-heavy queries.

Each node has local SSD used as a cache of recently read micro-partition data (the "warehouse cache"). It survives
between queries while the warehouse runs and is lost when it suspends. Up to `MAX_CONCURRENCY_LEVEL` queries share the
nodes at once. When the warehouse is saturated, new queries **queue**, and how long they wait is reported separately in
the query history, which is the signal for multi-cluster scaling (Level 4 · 06). Billing meters the time nodes are
allocated to your warehouse, per second after the first minute, regardless of how busy they are.

## Common mistakes

- **Leaving `AUTO_SUSPEND` at the default 600 seconds** for bursty workloads. The idle tail can cost more than the work.
- **No statement timeout.** The default lets a query run for two days.
- **Sizing up "to be safe".** Measure instead. Small queries gain nothing from a bigger warehouse.
- **Never sizing up for heavy jobs**, out of fear of the rate, when twice the size can cost the same and finish twice as
  fast.
- **One shared warehouse for ETL, BI and ad-hoc work.** That means contention and no cost attribution.
- **Creating warehouses without `INITIALLY_SUSPENDED = TRUE`**, which starts billing at creation.

## Exercise

1. Extend the billing model with a "resume cost" assumption: the first query after each resume takes 3 seconds longer
   (cold cache). Does `AUTO_SUSPEND = 60` still win for the dashboard workload? At what gap between queries does a longer
   auto-suspend become cheaper?
2. In a trial account, run the same heavy aggregation over `SNOWFLAKE_SAMPLE_DATA.TPCH_SF100.LINEITEM` on X-Small, Small
   and Medium (disable the result cache with `ALTER SESSION SET USE_CACHED_RESULT = FALSE`). Record elapsed time and
   compute credits for each from the query history. Does doubling the size halve the time?
3. Create `etl_wh`, `bi_wh` and `ds_wh` as above with appropriate timeouts, and write the `SHOW WAREHOUSES` query (with
   `RESULT_SCAN`) that lists each warehouse's size, auto-suspend and state.
4. Check your trial account's warehouses for the generation they were created with, and find the current Gen2 credit
   multiplier for your cloud in Snowflake's Service Consumption Table.
