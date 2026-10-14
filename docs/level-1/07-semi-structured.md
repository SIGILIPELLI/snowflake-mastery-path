---
description: "JSON in Snowflake: VARIANT columns, path and cast syntax, LATERAL FLATTEN and the OUTER flag, SQL NULL vs JSON null, and how VARIANT is stored."
---

# 07 · Semi-Structured Data: VARIANT & FLATTEN

Snowflake stores JSON (and Avro, Parquet, ORC and XML) in a column of type **`VARIANT`** and lets you query it with
path expressions, without defining a schema first. That makes it easy to land raw API responses and event payloads and
model them later. It also hides a few traps. An `INNER`-style `FLATTEN` silently drops rows with empty arrays, JSON
`null` isn't SQL `NULL`, and a field with nulls in it is stored less efficiently. Each of these is shown here on the
`customers.json` file from lesson 05.

## Loading JSON into a VARIANT

```sql
CREATE TABLE IF NOT EXISTS learn_db.raw.customers_json (
    v          VARIANT,
    file_name  VARCHAR,
    loaded_at  TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()::TIMESTAMP_NTZ
);

COPY INTO learn_db.raw.customers_json (v, file_name)
FROM (SELECT $1, METADATA$FILENAME FROM @learn_db.raw.landing/customers/)
FILE_FORMAT = (FORMAT_NAME = 'learn_db.raw.json_lines');
```

Each line of the file becomes one row with the whole object in `v`. `METADATA$FILENAME` records which file each row came
from, which is invaluable when you trace a bad record later. A record looks like this:

```json
{"customer_id": 1, "name": "Customer 1", "country": "ES", "signup": "2024-11-23",
 "contact": {"email": "c1@example.com", "phone": "+1-555-0001"}, "tags": []}
```

## Paths and casts

```sql
SELECT v:customer_id::NUMBER          AS customer_id,
       v:name::VARCHAR                AS name,
       v:contact.email::VARCHAR       AS email,
       v:contact.phone::VARCHAR       AS phone,
       v['country']::VARCHAR          AS country,
       v:tags[0]::VARCHAR             AS first_tag,
       TO_DATE(v:signup::VARCHAR)     AS signup_date
FROM learn_db.raw.customers_json
LIMIT 5;
```

- `:` steps into the top level, and `.` steps further (`v:contact.email`). Bracket notation (`v['country']`)
  handles keys with spaces or special characters.
- **Field names are case-sensitive**: `v:Name` and `v:name` are different paths.
- Without `::type`, a path returns a `VARIANT`, displayed with JSON quotes (`"Customer 1"`). Always cast to the type you
  mean, both for display and for correct comparisons and joins.
- A path that doesn't exist returns `NULL`, not an error. That's convenient, and it means typos produce silent `NULL`s.

## SQL NULL vs JSON null

The file has two kinds of "no phone number": a key whose value is `null` (every 7th customer, 714 of them) and, in other
datasets, a key that's missing entirely. Snowflake's documentation distinguishes:

- **SQL `NULL`**: the value is missing (also what a non-existent path returns).
- **JSON null** ("VARIANT NULL"): a JSON `null` value, stored as a VARIANT holding `null`.

```sql
SELECT COUNT(*)                                    AS customers,
       COUNT_IF(v:contact.phone IS NULL)           AS sql_null_phone,      -- JSON nulls are NOT SQL NULL
       COUNT_IF(IS_NULL_VALUE(v:contact.phone))    AS json_null_phone,
       COUNT_IF(v:contact.phone::VARCHAR IS NULL)  AS null_after_cast
FROM learn_db.raw.customers_json;
```

For this file you'd expect `sql_null_phone` = 0 (every record *has* the key), `json_null_phone` = 714, and
`null_after_cast` = 714, because casting a JSON null to a string produces SQL `NULL`. Those expected values come from the
file contents (714 records with `"phone": null`), not from a Snowflake run. If JSON nulls mean nothing special in your
data, the file format option `STRIP_NULL_VALUES = TRUE` removes them at load time, and that also helps storage, as
explained below.

## FLATTEN: arrays into rows

`tags` is an array. To count customers per tag, turn each array element into a row with `LATERAL FLATTEN`:

```sql
SELECT t.value::VARCHAR AS tag, COUNT(*) AS customers
FROM learn_db.raw.customers_json c,
     LATERAL FLATTEN(input => c.v:tags) t
GROUP BY tag
ORDER BY customers DESC;
```

`FLATTEN` returns the columns `SEQ`, `KEY`, `PATH`, `INDEX`, `VALUE` and `THIS`. `VALUE` is the element, `INDEX` its
position. Run on the same file in DuckDB (with `UNNEST`, DuckDB's equivalent; not Snowflake):

```text
tag        | customers
vip        |      1524
returning  |      1523
b2b        |      1500
newsletter |      1494
mobile     |      1464
```

Now count customers after flattening:

```text
rows_after_inner_unnest | customers_kept
                   7505 |           3737
```

There are 5,000 customers, but only **3,737** survive the flatten: the **1,263 customers with an empty `tags` array**
vanished, because an element-less array produces no rows, exactly like an inner join. Snowflake's `FLATTEN` behaves the
same way by default. To keep them, use `OUTER => TRUE`, which emits one row with `NULL`s for empty arrays:

```sql
SELECT c.v:customer_id::NUMBER AS customer_id, t.value::VARCHAR AS tag
FROM learn_db.raw.customers_json c,
     LATERAL FLATTEN(input => c.v:tags, OUTER => TRUE) t;
```

This is the most common silent-data-loss bug in semi-structured pipelines. A report built on the inner flatten
undercounts customers by 25% and nothing errors.

`FLATTEN` also walks objects (`KEY` holds the field name) and, with `RECURSIVE => TRUE`, entire nested structures. That
helps you discover what fields a messy payload actually contains:

```sql
SELECT DISTINCT f.path, TYPEOF(f.value) AS type
FROM learn_db.raw.customers_json c,
     LATERAL FLATTEN(input => c.v, RECURSIVE => TRUE) f
ORDER BY f.path;
```

## Building JSON

```sql
SELECT OBJECT_CONSTRUCT('id', o.customer_id, 'country', c.v:country::VARCHAR) AS obj,
       ARRAY_AGG(o.order_id) WITHIN GROUP (ORDER BY o.order_id)        AS order_ids
FROM learn_db.raw.orders o
JOIN learn_db.raw.customers_json c ON c.v:customer_id::NUMBER = o.customer_id
GROUP BY o.customer_id, c.v:country::VARCHAR;
```

`OBJECT_CONSTRUCT` drops keys whose values are SQL `NULL`. `OBJECT_CONSTRUCT_KEEP_NULL` keeps them as JSON nulls.
`PARSE_JSON` turns a string into a VARIANT, and `TO_JSON` does the reverse.

## From raw to modelled

The usual pattern is to land raw JSON in a VARIANT column, then expose a typed view (or table) for everyone else:

```sql
CREATE OR REPLACE VIEW learn_db.analytics.customers AS
SELECT v:customer_id::NUMBER       AS customer_id,
       v:name::VARCHAR             AS name,
       v:country::VARCHAR          AS country,
       TO_DATE(v:signup::VARCHAR)  AS signup_date,
       v:contact.email::VARCHAR    AS email,
       v:contact.phone::VARCHAR    AS phone,
       v:tags                      AS tags,
       file_name, loaded_at
FROM learn_db.raw.customers_json;
```

New fields in the payload don't break the load, and the view decides what's exposed and how it's typed.

## How It Actually Works

A `VARIANT` isn't stored as a text blob. When you load semi-structured data, Snowflake inspects the elements and
**extracts frequently occurring, consistently typed fields into their own hidden columns** inside each micro-partition,
stored and compressed like ordinary columns, with min/max metadata for pruning. The rest is kept in a single parsed
binary structure. Querying `v:country` on a well-behaved payload therefore reads only the extracted `country` subcolumn,
nearly as cheaply as a real column.

Snowflake's documentation lists the conditions that **prevent** extraction. One is that an element containing **even a
single JSON `null`** isn't extracted. So `contact.phone` in this dataset, with 714 nulls, would stay inside the generic
structure, and a filter on it can't use subcolumn statistics. Elements with mixed types (sometimes a string, sometimes a
number) are also not extracted, and by default extraction is limited to 200 elements per partition per table. That's
the technical reason behind the advice to strip nulls and keep types consistent, and to promote fields you filter on
often into real columns.

`FLATTEN` is a **table function** applied once per input row (that's what `LATERAL` expresses): it returns zero or more
rows per input, and the outer query joins each input row to its output rows. Zero output rows for an empty array means
the input row disappears, unless `OUTER => TRUE` asks for a placeholder row.

## Common mistakes

- **Forgetting `OUTER => TRUE`**, which dropped 1,263 of 5,000 customers here.
- **Comparing uncast VARIANTs** (`v:country = 'PT'` works by coercion, but joins and `GROUP BY` on uncast values behave
  unexpectedly). Cast with `::`.
- **Treating JSON null as SQL NULL.** `IS NULL` is false for it. Use `IS_NULL_VALUE`, cast first, or strip nulls at load
  time.
- **Case mistakes in paths.** `v:Country` silently returns `NULL`.
- **Leaving hot filter fields inside VARIANT** with nulls or mixed types, which blocks subcolumn extraction and pruning.
- **Exposing raw VARIANT to analysts.** Publish a typed view.

## Exercise

1. Load `customers.json` and run the null-counting query. Do your three counts match the expected 0 / 714 / 714? Reload
   with `STRIP_NULL_VALUES = TRUE` and run it again.
2. Run the tag count with and without `OUTER => TRUE`, then write a query that counts customers per tag *including* a
   `'(none)'` bucket for customers without tags.
3. Use the recursive `FLATTEN` query to list every path in the payload and its types. Add a record where `customer_id`
   is a string (`"42"`) and run it again.
4. Create the `analytics.customers` view and join it to `raw.orders` to find paid revenue per country.
