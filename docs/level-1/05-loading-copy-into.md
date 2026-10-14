---
description: "Bulk loading into Snowflake: stages, file formats, PUT, COPY INTO with ON_ERROR and VALIDATION_MODE, load metadata, file sizing and planted bad rows."
---

# 05 · Loading Data: Stages, File Formats & COPY INTO

Almost all data enters Snowflake the same way: files land in a **stage** (a location in cloud storage), and
**`COPY INTO`** reads them into a table in parallel. Row-by-row `INSERT`s from an application are the slow, expensive
exception. This lesson loads a synthetic sales dataset with three planted errors, and shows how to find them before they
reach your table.

## The dataset

The course uses one generated dataset throughout Level 1. Run the generator yourself and you'll get the same files,
because it's seeded:

```python
"""Generate the course's synthetic sales dataset: orders (CSV, gzip, 4 files) and customers (JSON lines)."""
import csv, gzip, json, random, datetime as dt
random.seed(2026)
COUNTRIES = ["PT", "ES", "FR", "DE", "IN", "US", "BR", "GB"]
customers = []
for cid in range(1, 5001):
    customers.append({"customer_id": cid, "name": f"Customer {cid}", "country": random.choice(COUNTRIES),
                      "signup": (dt.date(2024, 1, 1) + dt.timedelta(days=random.randrange(700))).isoformat(),
                      "contact": {"email": f"c{cid}@example.com", "phone": None if cid % 7 == 0 else f"+1-555-{cid:04d}"},
                      "tags": random.sample(["vip", "newsletter", "b2b", "mobile", "returning"], k=random.randint(0, 3))})
with open("customers.json", "w") as f:
    for c in customers: f.write(json.dumps(c) + "\n")
start = dt.datetime(2026, 1, 1)
rows = []
for oid in range(1, 200_001):
    ts = start + dt.timedelta(seconds=random.randrange(270 * 86400))
    qty = random.choice([1, 1, 1, 2, 2, 3, 5])
    price = round(random.uniform(3, 250), 2)
    rows.append([oid, random.randint(1, 5000), ts.strftime("%Y-%m-%d %H:%M:%S"), random.choice(["web", "store", "app"]),
                 f"SKU-{random.randint(1, 800):04d}", qty, f"{price:.2f}", random.choice(["paid", "paid", "paid", "refunded", "cancelled"])])
rows.sort(key=lambda r: r[2])
# three deliberately broken rows
rows[1000][5] = "two"                         # non-numeric quantity
rows[50000][2] = "2026-02-30 10:00:00"        # impossible date
rows[150000] = rows[150000][:6]               # missing columns
header = ["order_id", "customer_id", "ordered_at", "channel", "sku", "quantity", "unit_price", "status"]
for part in range(4):
    chunk = rows[part * 50_000:(part + 1) * 50_000]
    with gzip.open(f"orders_part{part}.csv.gz", "wt", newline="") as f:
        w = csv.writer(f); w.writerow(header); w.writerows(chunk)
print("written")
```

```text
866022 orders_part0.csv.gz
866020 orders_part1.csv.gz
865836 orders_part2.csv.gz
866416 orders_part3.csv.gz
894302 customers.json
```

```text
order_id,customer_id,ordered_at,channel,sku,quantity,unit_price,status
107572,573,2026-01-01 00:02:42,app,SKU-0492,1,167.53,paid
42731,4234,2026-01-01 00:05:22,store,SKU-0746,1,170.83,cancelled
```

Four gzip files of about 0.87 MB each (3.06 MB uncompressed) hold 200,000 orders. Three rows are broken on purpose: a
quantity of `two`, the date `2026-02-30`, and a row missing its last two columns.

## Stages

A stage is where files wait to be loaded:

| Stage | Reference | Use |
|---|---|---|
| User stage | `@~` | your own scratch files |
| Table stage | `@%orders` | files for one table only |
| Named internal stage | `@learn_db.raw.landing` | files uploaded from clients, managed by Snowflake |
| Named external stage | `@learn_db.raw.s3_landing` | files in your own S3/GCS/Azure bucket |

```sql
CREATE STAGE IF NOT EXISTS learn_db.raw.landing
  DIRECTORY = (ENABLE = TRUE)
  COMMENT = 'Files uploaded with PUT for the Level 1 labs';
```

For external stages, use a **storage integration** (an account-level object that holds a cloud IAM role Snowflake
assumes), never credentials pasted into the stage:

```sql
CREATE STORAGE INTEGRATION s3_landing_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-landing'
  STORAGE_ALLOWED_LOCATIONS = ('s3://my-company-landing/orders/');

CREATE STAGE learn_db.raw.s3_landing
  URL = 's3://my-company-landing/orders/'
  STORAGE_INTEGRATION = s3_landing_int;
```

(Creating the integration then requires adding Snowflake's IAM user to the role's trust policy on the AWS side. `DESC
INTEGRATION s3_landing_int` shows the values to use.)

## File formats

A file format describes how to parse files, and defining it once as an object keeps every load consistent:

```sql
CREATE FILE FORMAT IF NOT EXISTS learn_db.raw.csv_gz
  TYPE = CSV
  COMPRESSION = GZIP
  SKIP_HEADER = 1
  FIELD_DELIMITER = ','
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF = ('', 'NULL')
  EMPTY_FIELD_AS_NULL = TRUE
  ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE;

CREATE FILE FORMAT IF NOT EXISTS learn_db.raw.json_lines
  TYPE = JSON
  STRIP_OUTER_ARRAY = FALSE;
```

## Uploading with PUT

`PUT` uploads local files to an internal stage. It runs from a client (Snowflake CLI, a driver), not from a Snowsight
worksheet:

```bash
snow stage copy "./orders_part*.csv.gz" @learn_db.raw.landing/orders/ -c trial
snow sql -c trial -q "LIST @learn_db.raw.landing/orders/"
```

`snow stage copy --help` notes that a local glob pattern "has to be enclosed in quotes", hence the quotes. The
equivalent SQL from a driver is `PUT file://./orders_part*.csv.gz @learn_db.raw.landing/orders/ AUTO_COMPRESS = FALSE`.
The files are already gzipped, and `AUTO_COMPRESS = FALSE` stops the client from compressing them again.

## Validate before you load

```sql
CREATE TABLE IF NOT EXISTS learn_db.raw.orders (
    order_id     NUMBER(38,0),
    customer_id  NUMBER(38,0),
    ordered_at   TIMESTAMP_NTZ,
    channel      VARCHAR,
    sku          VARCHAR,
    quantity     NUMBER(10,0),
    unit_price   NUMBER(10,2),
    status       VARCHAR
);

COPY INTO learn_db.raw.orders
  FROM @learn_db.raw.landing/orders/
  FILE_FORMAT = (FORMAT_NAME = 'learn_db.raw.csv_gz')
  VALIDATION_MODE = RETURN_ALL_ERRORS;
```

`VALIDATION_MODE` parses the files and **returns the errors without loading anything**. Each error row names the file,
line, column and the problem. For this dataset, you should see exactly the three planted problems: a numeric conversion
failure for `two`, a timestamp conversion failure for `2026-02-30`, and a column-count mismatch. (`RETURN_ERRORS`
returns errors only from the first file that has any. `RETURN_ALL_ERRORS` also includes files partially loaded by an
earlier `ON_ERROR = CONTINUE` run.)

You can check what to expect without a Snowflake account. Here's DuckDB 1.5.6 locally (not Snowflake), with its
reject table, typing the same columns:

```sql
-- DuckDB 1.5.6 (local), not Snowflake
CREATE TABLE orders AS SELECT * FROM read_csv('orders_part*.csv.gz', header=true, store_rejects=true,
  columns={'order_id':'BIGINT','customer_id':'BIGINT','ordered_at':'TIMESTAMP','channel':'VARCHAR','sku':'VARCHAR',
           'quantity':'INTEGER','unit_price':'DECIMAL(10,2)','status':'VARCHAR'});
SELECT COUNT(*) AS rows_loaded FROM orders;
SELECT DISTINCT line, column_name, error_type, left(csv_line, 58) AS csv_line FROM reject_errors ORDER BY csv_line;
```

```text
rows_loaded: 199997

│   1002 │ quantity    │ CAST            │ 36809,2098,2026-01-02 10:09:16,app,SKU-0500,two,64.20,pai │
│      2 │ status      │ MISSING COLUMNS │ 119204,411,2026-07-21 23:35:33,web,SKU-0362,3             │
│      2 │ unit_price  │ MISSING COLUMNS │ 119204,411,2026-07-21 23:35:33,web,SKU-0362,3             │
│      2 │ ordered_at  │ CAST            │ 58910,3983,2026-02-30 10:00:00,store,SKU-0256,1,76.57,canc│
```

(Trimmed from DuckDB's box output.) Three bad rows, 199,997 good ones. One DuckDB detail is worth knowing for testing
this way: an earlier attempt that only ran `SELECT COUNT(*)` over the files reported 199,999 rows. The engine skipped
converting columns it didn't need, so the two conversion errors were never hit. Validate by loading the typed columns.

## Load with an error policy

```sql
COPY INTO learn_db.raw.orders
  FROM @learn_db.raw.landing/orders/
  FILE_FORMAT = (FORMAT_NAME = 'learn_db.raw.csv_gz')
  ON_ERROR = 'CONTINUE';
```

`ON_ERROR` decides what bad rows do:

| Value | Effect |
|---|---|
| `ABORT_STATEMENT` | **default for `COPY`**: the first error stops the load and nothing is loaded |
| `CONTINUE` | load good rows, skip bad ones (reports one error per file in the result) |
| `SKIP_FILE` | skip any file containing an error (**default for Snowpipe**) |
| `SKIP_FILE_10`, `'SKIP_FILE_10%'` | skip a file once its errors reach a count or exceed a percentage |

With the default, this dataset loads **nothing**: the first planted error aborts the statement. With `CONTINUE` you'd
expect 199,997 rows loaded, with `ROWS_PARSED` exceeding `ROWS_LOADED` by the bad rows in the affected files. That
expectation comes from the data, as validated above. It's not output from a Snowflake run. `COPY` returns one result row
per file (status, rows parsed, rows loaded, first error), and the `VALIDATE` table function retrieves all errors from a
previous `COPY` by its query ID:

```sql
SELECT * FROM TABLE(VALIDATE(learn_db.raw.orders, JOB_ID => '_last'));
```

`SKIP_FILE` is the safer choice when a file is a unit of meaning (one day's export): either all of it loads or none. The
docs warn that `SKIP_FILE` buffers each whole file and is slower than `CONTINUE` or `ABORT_STATEMENT`.

## Running the same COPY twice

```sql
COPY INTO learn_db.raw.orders FROM @learn_db.raw.landing/orders/
  FILE_FORMAT = (FORMAT_NAME = 'learn_db.raw.csv_gz') ON_ERROR = 'CONTINUE';
```

The second run loads nothing, because every file is skipped as already loaded. `COPY` keeps **load metadata** per table,
recording which files (by name and checksum) it has loaded, and skips them. That metadata **expires after 64 days**. After
that, a file whose load status is unknown is skipped by default, unless you set `LOAD_UNCERTAIN_FILES = TRUE`.
`FORCE = TRUE` reloads everything and duplicates rows. Use it only after truncating.

## Transforming during load, and loading by column name

`COPY` can select, reorder and cast columns from the staged files:

```sql
COPY INTO learn_db.raw.orders (order_id, customer_id, ordered_at, channel, sku, quantity, unit_price, status)
FROM (
    SELECT $1::NUMBER, $2::NUMBER, TRY_TO_TIMESTAMP_NTZ($3), LOWER($4), $5,
           TRY_TO_NUMBER($6), $7::NUMBER(10,2), $8
    FROM @learn_db.raw.landing/orders/
)
FILE_FORMAT = (FORMAT_NAME = 'learn_db.raw.csv_gz');
```

`TRY_TO_*` functions return `NULL` instead of failing, so `two` and `2026-02-30` become `NULL`s you can find later rather
than errors. That's useful, and dangerous if nobody looks. For Parquet, Avro and ORC, `MATCH_BY_COLUMN_NAME =
CASE_INSENSITIVE` maps file columns to table columns by name, and `INFER_SCHEMA` can generate the table definition:

```sql
CREATE TABLE learn_db.raw.events
  USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
    FROM TABLE(INFER_SCHEMA(LOCATION => '@learn_db.raw.landing/events/', FILE_FORMAT => 'learn_db.raw.parquet_fmt'))
  );
```

## Sizing files

Snowflake recommends files of roughly **100–250 MB compressed** (or larger) for bulk loading. Files are the unit of
parallelism: one file is processed by one thread, so one huge file can't use the whole warehouse, and millions of tiny
files spend more time on per-file overhead than on data. The 0.87 MB files here are far below that, which is fine for a
lab but would waste credits at scale. Batch small outputs before loading.

## How It Actually Works

`COPY INTO` runs on your warehouse. Snowflake lists the files in the stage path, filters out those recorded in the
table's load metadata, and distributes the remaining files across the warehouse's threads. Each thread decompresses
and parses its file, applies the file format and any transformation, and writes rows into new micro-partitions. The
whole `COPY` is one transaction. Under `ABORT_STATEMENT` an error means no micro-partitions are committed. Under
`CONTINUE`, the good rows from all files are committed together. Afterwards, the load metadata records each file's name,
size, checksum and status, which is how a repeated `COPY` knows to skip them, and why that protection only lasts as long
as the metadata (64 days).

`PUT` never touches a warehouse. The client encrypts (and optionally compresses) the files and uploads them straight to
the stage's cloud storage. An external stage with a storage integration means Snowflake reads your bucket by assuming
the IAM role you granted it, so no keys are stored in Snowflake.

## Common mistakes

- **Relying on the default `ABORT_STATEMENT`** in a pipeline and discovering a single bad row has blocked a whole day's
  load. Choose `ON_ERROR` deliberately and monitor errors.
- **`ON_ERROR = CONTINUE` without monitoring** the difference between rows parsed and loaded.
- **`TRY_TO_*` conversions that silently turn bad data into `NULL`s.** Count them after loading.
- **`FORCE = TRUE`** to "fix" a load, which duplicates every previously loaded row.
- **Credentials in stage definitions.** Use storage integrations.
- **Thousands of tiny files**, or one 20 GB file. Aim for 100–250 MB compressed.
- **Testing a load with `COUNT(*)` only.** As DuckDB showed, unused columns may never be converted. Validate typed loads.

## Exercise

1. Generate the dataset, upload it to an internal stage, and run `COPY … VALIDATION_MODE = RETURN_ALL_ERRORS`. Compare the
   three errors Snowflake reports with the DuckDB result above.
2. Load with `ON_ERROR = 'SKIP_FILE'` and record which file(s) were skipped and how many rows that cost. Then reload the
   skipped files with `ON_ERROR = 'CONTINUE'`, using the `FILES = (...)` option to target them.
3. Load the same files with the `TRY_TO_*` transformation, then write a query that finds every row where a conversion
   produced `NULL`.
4. Load `customers.json` into a table with a single `VARIANT` column (lesson 07 queries it).
