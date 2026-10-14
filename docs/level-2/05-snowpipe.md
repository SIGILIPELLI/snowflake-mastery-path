---
description: "Continuous loading with Snowpipe: auto-ingest pipes, per-GB billing, the 14-day metadata trap for re-staged files, monitoring, and Snowpipe Streaming."
---

# 05 · Continuous Loading with Snowpipe

Lesson 05 of Level 1 loaded files with `COPY INTO` on a warehouse you control. That's ideal for big, scheduled batches.
For data that trickles in all day (files every few minutes from an application, a CDC tool or a partner's export), a
warehouse that resumes every five minutes to load a few megabytes wastes most of what it bills. **Snowpipe** runs the
same `COPY` logic on Snowflake-managed serverless compute whenever new files appear, and bills by the gigabyte.

## A pipe is a saved COPY

```sql
CREATE PIPE IF NOT EXISTS learn_db.raw.orders_pipe
  AUTO_INGEST = TRUE
  COMMENT = 'Loads order files as they land in s3://my-company-landing/orders/'
AS
COPY INTO learn_db.raw.orders (order_id, customer_id, ordered_at, channel, sku, quantity, unit_price, status, file_name)
FROM (SELECT $1, $2, $3, $4, $5, $6, $7, $8, METADATA$FILENAME FROM @learn_db.raw.s3_landing/orders/)
FILE_FORMAT = (FORMAT_NAME = 'learn_db.raw.csv_gz');
```

The pipe holds a `COPY INTO` statement, and Snowpipe runs it for each new file. There are two ways for it to learn about
new files:

- **`AUTO_INGEST = TRUE`**: the cloud provider sends an **event notification** when a file lands, for example an S3 event
  notification to the SQS queue Snowflake creates for the pipe. For S3, `SHOW PIPES` shows that queue's ARN in the
  `notification_channel` column, and you configure the bucket to send `ObjectCreated` events to it. On GCS and Azure, you
  create a notification integration instead.
- **REST API**: your application calls Snowpipe's `insertFiles` endpoint with the file names after uploading them, which
  is useful for internal stages or when you control the writer.

To load files already in the stage when the pipe was created (Snowpipe only reacts to new events), run once:

```sql
ALTER PIPE learn_db.raw.orders_pipe REFRESH;     -- queues files staged in the last 7 days
```

## How Snowpipe differs from bulk COPY

From Snowflake's documentation:

| | Bulk `COPY INTO` | Snowpipe |
|---|---|---|
| Trigger | you run it (or a task does) | file events or REST calls |
| Compute | your warehouse | serverless |
| Typical latency | as often as you run it | typically within a minute of Snowpipe learning about a file |
| Billing | warehouse time | **0.0037 credits per GB** (all editions since 8 December 2025) |
| Default `ON_ERROR` | `ABORT_STATEMENT` | **`SKIP_FILE`** |
| Transaction | one per `COPY` statement | per file (files may be combined or split across transactions) |
| Load metadata | per table, 64 days | **per pipe, 14 days** |

Billing is on the file's **uncompressed** size for text formats (CSV, JSON, XML) and the stored size for binary formats
(Parquet, Avro, ORC). A 1 GB gzip CSV that's 5 GB uncompressed is billed as 5 GB.

## The cost difference for trickle data

A model of the documented rates (Python, locally). It **assumes** each small scheduled `COPY` takes 10 seconds on an
X-Small with a 60-second auto-suspend, so each run bills 70 seconds:

```python
SNOWPIPE_PER_GB = 0.0037
def snowpipe(gb_per_day): return gb_per_day * SNOWPIPE_PER_GB
def scheduled_copy(runs_per_day, seconds_per_run=10, auto_suspend=60, credits_per_hour=1):
    return runs_per_day * max(60, seconds_per_run + auto_suspend) / 3600 * credits_per_hour
```

```text
 uncompressed GB/day  Snowpipe cr/day |  COPY every 5 min  COPY every hour
                   1           0.0037 |             5.600            0.467
                   3           0.0111 |             5.600            0.467
                  10           0.0370 |             5.600            0.467
                  30           0.1110 |             5.600            0.467
```

For trickle volumes, running a warehouse every five minutes costs **hundreds of times** more than Snowpipe: 5.6 credits a
day of mostly idle minimums against cents. Even hourly loads cost more until volumes reach well over 100 GB/day. For
large daily batches, compare against measured warehouse load time for your own files. Load speed depends heavily on file
format, size and transformation, so it isn't modelled here.

## The 14-day trap: re-staging a corrected file

Snowflake's documentation is explicit:

> Each pipe keeps load metadata, which is a record of the files that it processed in the last 14 days … Snowpipe ignores
> any staged file whose path and name match a file in this record, even if the file was modified … Because Snowpipe also
> records files that failed to load, it ignores a corrected file that you stage again with …

…the same name. Picture the realistic incident: a file fails to load (`SKIP_FILE`, because of one bad row), someone
fixes the row and uploads the file again **under the same name**, and Snowpipe silently ignores it for 14 days. The fix
is a new file name (add a version or timestamp suffix), or a one-off manual `COPY INTO` of that file on a warehouse.
`ALTER PIPE … REFRESH` won't help, because the name is already recorded.

## Monitoring

```sql
SELECT SYSTEM$PIPE_STATUS('learn_db.raw.orders_pipe');      -- execution state, pending file count, last errors

SELECT file_name, status, row_count, row_parsed, first_error_message, last_load_time
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
       TABLE_NAME => 'learn_db.raw.orders',
       START_TIME => DATEADD('hour', -24, CURRENT_TIMESTAMP())))
ORDER BY last_load_time DESC;

SELECT start_time, end_time, credits_used, bytes_billed, bytes_inserted, files_inserted
FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE pipe_name = 'ORDERS_PIPE'
ORDER BY start_time DESC;
```

Alert on files with a `Load failed` status. Under `SKIP_FILE`, a bad file is skipped without stopping anything else, so
nobody notices unless something is watching. Pausing for maintenance:

```sql
ALTER PIPE learn_db.raw.orders_pipe SET PIPE_EXECUTION_PAUSED = TRUE;
ALTER PIPE learn_db.raw.orders_pipe SET PIPE_EXECUTION_PAUSED = FALSE;
```

## Snowpipe Streaming

When data arrives as **rows**, not files (Kafka topics, application events, CDC streams), writing small files just to
have Snowpipe read them adds latency and cost. **Snowpipe Streaming** lets a client (the Snowflake Kafka connector, or
your own code using Snowflake's ingest SDK) write rows directly into tables. Snowflake's documentation describes its
high-performance architecture as delivering up to 20 GB/s per table and ingest-to-queryable latency as low as 5 seconds,
with ordered, exactly-once ingestion through named channels and offset tokens. The per-GB rate above covers it too.
Streaming wasn't exercised for this course, so check the current SDK and connector documentation before designing
around it.

## How It Actually Works

With auto-ingest, the cloud provider publishes an event per new object to a queue owned by Snowflake. Snowpipe consumes
those events, filters them against the pipe's stage path and its 14-day load metadata, and puts the remaining files in
a per-pipe ingestion queue. Snowflake-managed compute picks up queued files, runs the pipe's `COPY` logic on them
(possibly combining small files or splitting large ones), and commits the rows. Because it's not one statement over a
known file list, there's no single transaction: files appear in the table as they're processed, in no guaranteed order.
That's why downstream processing usually uses streams (next lesson) rather than assuming a load "finished".

Billing per GB means file count and file size no longer change what you pay directly, but they still affect latency and
efficiency. Snowflake's general advice of roughly 100–250 MB compressed files, or new files about once a minute for
continuous sources, still applies.

## Common mistakes

- **A warehouse waking every few minutes for small loads.** That's Snowpipe's job.
- **Re-staging a fixed file under the same name** and waiting for it to load. It won't, for 14 days.
- **Not monitoring `SKIP_FILE` failures.** They're silent by design.
- **Expecting ordering** across files. Process downstream by content (timestamps, sequence numbers), not arrival.
- **Forgetting `REFRESH`** for files that were staged before the pipe existed.
- **Credentials in stages** instead of storage integrations.

## Exercise

1. In a trial account with an S3 bucket (or Azure/GCS equivalent), create the stage, integration and auto-ingest pipe.
   Upload the four order files one at a time and watch them appear with `COPY_HISTORY`.
2. Upload a version of `orders_part2.csv.gz` with a broken row. Confirm it's skipped, fix it, upload it again with the same
   name, and observe that nothing happens. Then load it properly.
3. Using your actual daily volume and the model above, at what volume would an hourly scheduled `COPY` on an X-Small become
   cheaper than Snowpipe? Measure the real `COPY` time for one of your files rather than using the 10-second assumption.
4. Write a monitoring query that returns every file loaded with errors, or skipped, in the last 24 hours across all pipes in
   a database.
