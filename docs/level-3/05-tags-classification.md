---
description: "Snowflake object tags: allowed values, inheritance and propagation, tag-based masking for every tagged column, and sensitive data classification."
---

# 05 · Object Tagging & Data Classification

Attaching a masking policy to each column (lesson 04) works for ten columns. It doesn't work for ten thousand across
hundreds of tables that teams create every week. **Tags** let you describe data once (`pii = 'email'`, `cost_center =
'marketing'`) and drive governance from those descriptions: masking that follows the tag, cost reports by tag, audits of
where sensitive data lives. **Sensitive data classification** goes a step further and proposes the tags for you. This lesson
covers both, including why proposed tags still need a human.

## Creating and setting tags

```sql
USE ROLE SECURITYADMIN;
CREATE SCHEMA IF NOT EXISTS security_db.tags;

CREATE TAG IF NOT EXISTS security_db.tags.pii
  ALLOWED_VALUES 'email', 'phone', 'name', 'address', 'national_id'
  COMMENT = 'Kind of personal data in a column';

CREATE TAG IF NOT EXISTS security_db.tags.cost_center
  COMMENT = 'Who pays for this object';

ALTER TABLE sales.mart.customers MODIFY COLUMN email SET TAG security_db.tags.pii = 'email';
ALTER TABLE sales.mart.customers MODIFY COLUMN phone SET TAG security_db.tags.pii = 'phone';
ALTER WAREHOUSE bi_wh SET TAG security_db.tags.cost_center = 'analytics';
ALTER DATABASE sales SET TAG security_db.tags.cost_center = 'sales';
```

From Snowflake's documentation:

- **`ALLOWED_VALUES`** restricts a tag to a fixed vocabulary, so nobody can set `pii = 'emial'`.
- Tags are **inherited** down the object hierarchy. A tag set on a database applies to its schemas, tables and columns,
  unless a lower level sets its own value. `cost_center = 'sales'` on the database tags every table in it.
- Tags can be configured to **propagate** automatically from source objects to targets (for example, from a table to tables
  created from it).
- Quotas: at most **50 tags per object**, and at most 50 different tags across the columns of one table.

Finding tags:

```sql
SELECT * FROM TABLE(sales.INFORMATION_SCHEMA.TAG_REFERENCES('sales.mart.customers', 'TABLE'));
SELECT * FROM TABLE(sales.INFORMATION_SCHEMA.TAG_REFERENCES_ALL_COLUMNS('sales.mart.customers', 'TABLE'));

SELECT object_database, object_schema, object_name, column_name, tag_value
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE tag_name = 'PII'
ORDER BY 1, 2, 3, 4;
```

The `apply_method` column in these results shows whether a tag was set manually, inherited or propagated.

## Tag-based masking: protect every tagged column

Set masking policies **on the tag**, and every column carrying that tag is masked by the policy matching the column's data
type, with no per-column `ALTER`:

```sql
CREATE OR REPLACE MASKING POLICY security_db.policies.pii_string
  AS (val VARCHAR) RETURNS VARCHAR ->
  CASE
    WHEN IS_ROLE_IN_SESSION('PII_READER') THEN val
    WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('security_db.tags.pii') = 'email' THEN '***@' || SPLIT_PART(val, '@', 2)
    ELSE '*** masked ***'
  END;

CREATE OR REPLACE MASKING POLICY security_db.policies.pii_number
  AS (val NUMBER) RETURNS NUMBER ->
  CASE WHEN IS_ROLE_IN_SESSION('PII_READER') THEN val ELSE NULL END;

ALTER TAG security_db.tags.pii SET MASKING POLICY security_db.policies.pii_string,
                                   MASKING POLICY security_db.policies.pii_number;
```

- A tag can carry **one masking policy per data type**. A `VARCHAR` column tagged `pii` uses `pii_string`, and a `NUMBER`
  column uses `pii_number`.
- `SYSTEM$GET_TAG_ON_CURRENT_COLUMN` lets one policy behave differently per tag value: partial masking for emails, full
  masking for everything else.
- A masking policy assigned **directly** to a column takes precedence over a tag-based one, which is how you make exceptions.
- Because tags can be set at the table, schema or database level, tagging a whole schema `pii` masks every string column in
  it. That's powerful, and too broad for most schemas. Tag columns.

The effect is a governance workflow where new tables are protected as soon as their columns are tagged correctly, which
moves the problem to *getting the tags right*.

## Sensitive data classification

**Sensitive data classification** (Enterprise Edition) samples column values, assigns each sensitive column a **semantic
category** (such as `EMAIL`, `NAME`, `PHONE_NUMBER`) and a **privacy category** (identifier, quasi-identifier, sensitive),
and records them as system tags (`SNOWFLAKE.CORE.SEMANTIC_CATEGORY`, `SNOWFLAKE.CORE.PRIVACY_CATEGORY`). A **classification
profile** associated with a database classifies its tables and views automatically, and can map those system tags onto your
own tags, so a column classified as `EMAIL` gets `security_db.tags.pii = 'email'` and therefore the masking policy:

```sql
CREATE OR REPLACE SNOWFLAKE.DATA_PRIVACY.CLASSIFICATION_PROFILE security_db.tags.default_profile({
    'minimum_object_age_for_classification_days': 0,
    'auto_tag': true
  });

ALTER DATABASE sales SET CLASSIFICATION_PROFILE = 'security_db.tags.default_profile';
```

The current documentation also describes custom categories (your own detectors) and an optional AI mode. Check it for the
profile options available in your account. This feature wasn't run for this course.

### Why classification still needs review

Classification is **sample-based and probabilistic**. To see why that matters, here is a toy classifier (written for this
lesson, **not** Snowflake's) that samples 1,000 customers from the course data and tests each column against simple
detectors:

```python
"""A toy column classifier (NOT Snowflake's): sample values, test each column against simple detectors,
and report the share of non-null sampled values that match. Illustrates why classification is
sample-based and probabilistic."""
import json, random, re
random.seed(1)
rows = [json.loads(l) for l in open("customers.json")]
flat = [{"customer_id": r["customer_id"], "name": r["name"], "country": r["country"], "signup": r["signup"],
         "email": r["contact"]["email"], "phone": r["contact"]["phone"]} for r in rows]
DETECTORS = {
    "EMAIL": re.compile(r"^[^@\s]+@[^@\s]+\.[a-z]{2,}$"),
    "PHONE_NUMBER": re.compile(r"^\+?[0-9][0-9\- ]{6,}$"),
    "COUNTRY": re.compile(r"^(PT|ES|FR|DE|IN|US|BR|GB)$"),
    "DATE": re.compile(r"^\d{4}-\d{2}-\d{2}$"),
}
sample = random.sample(flat, 1000)
for col in flat[0]:
    values = [str(r[col]) for r in sample if r[col] is not None]
    best = max(DETECTORS, key=lambda d: sum(bool(DETECTORS[d].match(v)) for v in values))
    share = sum(bool(DETECTORS[best].match(v)) for v in values) / len(values)
    nulls = sum(r[col] is None for r in sample)
    label = best if share >= 0.8 else "-"
    print(f"{col:12} nulls {nulls:>3}/1000  best match {best:13} {share:6.1%}  -> {label}")
```

```text
customer_id  nulls   0/1000  best match EMAIL           0.0%  -> -
name         nulls   0/1000  best match EMAIL           0.0%  -> -
country      nulls   0/1000  best match COUNTRY       100.0%  -> COUNTRY
signup       nulls   0/1000  best match PHONE_NUMBER  100.0%  -> PHONE_NUMBER
email        nulls   0/1000  best match EMAIL         100.0%  -> EMAIL
phone        nulls 120/1000  best match PHONE_NUMBER  100.0%  -> PHONE_NUMBER
```

It found the email and country columns correctly, and found phone numbers despite 120 null values in the sample. It also
labelled **`signup` dates as phone numbers**: a value like `2024-11-23` matches a loose phone pattern just as well as a date
pattern, and the tie went to the detector listed first. Snowflake's classifier is far more sophisticated than three regular
expressions, but the general lesson carries over. Automated classification produces **proposals with false positives and
false negatives** (a free-text "notes" column full of phone numbers that only sometimes matches, an ID that looks like a
national identifier). Review the proposed categories, especially before they drive masking automatically, and audit the
results periodically.

## Tags for cost allocation

Tags aren't only for security. Tagging warehouses, databases and users with `cost_center` lets you attribute credits:

```sql
SELECT t.tag_value AS cost_center, SUM(m.credits_used) AS credits_30d
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY m
JOIN SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES t
  ON t.domain = 'WAREHOUSE' AND t.object_name = m.warehouse_name AND t.tag_name = 'COST_CENTER'
WHERE m.start_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY 2 DESC;
```

Level 4 · 05 builds on this for chargeback.

## How It Actually Works

A tag is a schema-level object, and a tag assignment is metadata linking `(tag, value)` to an object or column. When a query
touches a column, the compiler resolves its **effective tags**: the column's own assignments, plus those inherited from its
table, schema and database, with the most specific level winning. If any effective tag has a masking policy for the
column's data type (and no masking policy is set directly on the column), the compiler applies that policy exactly as in
lesson 04, with `SYSTEM$GET_TAG_ON_CURRENT_COLUMN` returning the resolved tag value. Classification runs as a background
service that reads samples of column values, evaluates them with its detectors, and writes system tag assignments, which
from then on behave like any other tags.

## Common mistakes

- **Free-form tag values.** Use `ALLOWED_VALUES`.
- **Tagging at the schema level for masking.** Every column of that type gets masked. Tag the columns.
- **One masking policy per tag "for strings"** and nothing for numbers or dates, leaving those columns unprotected.
- **Trusting automatic classification without review.** It proposes, and someone accountable approves.
- **Tags that nobody queries.** Build the audit and cost reports that make them worth maintaining.
- **Forgetting direct policies override tag policies.** An old per-column policy silently wins.

## Exercise

1. Create the `pii` tag with allowed values and tag-based masking policies for `VARCHAR` and `NUMBER`, then tag the course's
   customer columns and verify masking with `POLICY_CONTEXT` for a role without `PII_READER`.
2. Tag the `sales` database with `cost_center = 'sales'` and one table with `cost_center = 'marketing'`. Query
   `TAG_REFERENCES` and confirm which value each table resolves to.
3. Fix the toy classifier so dates aren't classified as phone numbers (a stricter phone pattern? a priority order?). Then
   construct an input that fools your fixed version.
4. In a trial account, associate a classification profile with a database containing the course's customers, and compare
   Snowflake's semantic categories with the toy's.
