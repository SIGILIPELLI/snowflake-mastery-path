---
description: "Snowflake Python UDFs, vectorized UDFs and UDTFs: runtimes, Anaconda and PyPI packages, external access, and when batching helps (measured)."
---

# 03 · Python UDFs, Vectorized UDFs & UDTFs

Some logic is easier in Python than in SQL: parsing a messy string, scoring with a trained model, calling a library.
Snowflake runs Python **inside the warehouse**, next to the data, as user-defined functions. This lesson covers the
three shapes (scalar, vectorized and table functions), how packages get there, and a measurement that complicates the
usual advice that vectorized UDFs are always faster.

## A scalar Python UDF

```sql
CREATE OR REPLACE FUNCTION sales.mart.email_domain(email VARCHAR)
RETURNS VARCHAR
LANGUAGE PYTHON
RUNTIME_VERSION = '3.12'
HANDLER = 'domain'
AS
$$
def domain(email):
    if email is None or "@" not in email:
        return None
    return email.split("@", 1)[1].lower()
$$;

SELECT sales.mart.email_domain(email) AS domain, COUNT(*) FROM sales.mart.customers GROUP BY 1 ORDER BY 2 DESC;
```

`RUNTIME_VERSION` pins the Python version. Per the documentation (October 2026) the supported versions are 3.10 to 3.14.
`HANDLER` names the function to call. SQL `NULL` arrives as Python `None`, so handle it explicitly, as above.

## Packages

Third-party libraries come from three places:

- **The Snowflake Anaconda channel**, listed with `PACKAGES = ('pandas', 'numpy')`. Available versions are listed in
  `INFORMATION_SCHEMA.PACKAGES`.
- **PyPI**, through Snowflake's **Artifact Repository**: set `ARTIFACT_REPOSITORY = snowflake.snowpark.pypi_shared_repository`
  and list the packages. Per the documentation, the database role `SNOWFLAKE.PYPI_REPOSITORY_USER` is required and is
  granted to `PUBLIC` by default. A default repository can also be set for an account, database or schema with
  `DEFAULT_PYTHON_ARTIFACT_REPOSITORY`.
- **Your own code**, uploaded to a stage and listed with `IMPORTS = ('@stage/mylib.zip')`.

```sql
CREATE OR REPLACE FUNCTION ml.scoring.parse_phone(raw VARCHAR)
RETURNS VARCHAR
LANGUAGE PYTHON
RUNTIME_VERSION = '3.12'
ARTIFACT_REPOSITORY = snowflake.snowpark.pypi_shared_repository
PACKAGES = ('phonenumbers')
HANDLER = 'parse'
AS
$$
import phonenumbers

def parse(raw):
    if raw is None:
        return None
    try:
        return phonenumbers.format_number(phonenumbers.parse(raw, "GB"), phonenumbers.PhoneNumberFormat.E164)
    except phonenumbers.NumberParseException:
        return None
$$;
```

Handler code can't reach the internet by default. Calling an external API needs an **external access integration**:
network rules (`MODE = EGRESS`) for the allowed hosts, optional **secrets** for credentials, and
`EXTERNAL_ACCESS_INTEGRATIONS = (…)` on the function. Treat it as a security review item, like the network policies in
Level 3 · 03.

## Vectorized UDFs

A scalar handler is called **once per row**. A **vectorized** handler is called once per **batch** of rows, which arrive as a
pandas DataFrame with columns in argument order (`df[0]`, `df[1]`, …), and it returns a pandas Series or array:

```sql
CREATE OR REPLACE FUNCTION ml.scoring.churn_score(amount FLOAT, tenure_months FLOAT)
RETURNS FLOAT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.12'
PACKAGES = ('pandas', 'numpy')
HANDLER = 'score'
AS
$$
import numpy as np
import pandas
from _snowflake import vectorized

@vectorized(input=pandas.DataFrame)
def score(df):
    return 1 / (1 + np.exp(-(0.004 * df[0] - 0.03 * df[1] + 0.5)))
$$;
```

The usual advice is "vectorize for speed". Here's what that actually depends on. The same two handlers (email parsing and
a numeric score), each written both ways and timed on one million rows, locally:

```python
# Local timing of UDF HANDLERS only (plain Python 3.14 + pandas 2.3 on this machine, not Snowflake):
# the same logic called once per row (scalar UDF) vs once per batch of 8,192 rows (vectorized UDF).
import math, time, numpy as np, pandas as pd
rng = np.random.default_rng(7)
N, BATCH = 1_000_000, 8192
emails = pd.Series([f"user{i}@{d}" for i, d in enumerate(rng.choice(["example.com", "Mail.Example.org"], N))])
amount, tenure = pd.Series(rng.uniform(1, 500, N)), pd.Series(rng.integers(0, 120, N).astype(float))

def run(name, scalar, vectorized, cols):
    t0 = time.perf_counter(); a = [scalar(*row) for row in zip(*cols)]; t1 = time.perf_counter()
    b = pd.concat([vectorized(pd.DataFrame({i: c[s:s + BATCH] for i, c in enumerate(cols)}))
                   for s in range(0, N, BATCH)])
    t2 = time.perf_counter()
    same = np.allclose(a, b) if isinstance(a[0], float) else list(b) == a
    print(f"{name:<14} scalar {t1 - t0:5.2f} s   vectorized {t2 - t1:5.2f} s   identical results: {same}")

# 1. string parsing
run("email domain",
    lambda e: e.split("@", 1)[1].lower(),
    lambda df: df[0].str.split("@", n=1).str[1].str.lower(),
    [emails])
# 2. numeric scoring
run("churn score",
    lambda a, t: 1 / (1 + math.exp(-(0.004 * a - 0.03 * t + 0.5))),
    lambda df: 1 / (1 + np.exp(-(0.004 * df[0] - 0.03 * df[1] + 0.5))),
    [amount, tenure])
print(f"rows: {N:,}  batch: {BATCH:,}")
```

```text
email domain   scalar  0.18 s   vectorized  0.44 s   identical results: True
churn score    scalar  0.22 s   vectorized  0.04 s   identical results: True
rows: 1,000,000  batch: 8,192
```

Batching made the **numeric** score about five times faster in this run, because `np.exp` on a whole column runs in compiled
code. It made the **string** parsing more than twice as **slow**: pandas' `.str` methods still process one string at a time in
Python, plus the overhead of building Series. Two caveats keep this honest. These are handler timings on one machine, and
they leave out the cost of moving rows between Snowflake's query engine and the Python process, which this run can't
measure. Snowflake's documentation is careful here too: it promises vectorized UDFs "the potential for better performance if
your Python code operates efficiently on batches of rows". So measure your own function on a warehouse before deciding. The rule this run supports is narrower than "vectorize everything":
**vectorize when the handler's work is array math**. For string-heavy logic, first check whether built-in SQL
(`SPLIT_PART(email, '@', 2)`) can do it with no Python at all.

`max_batch_size` in the decorator only **limits** batch size, for handlers that can't take large batches. The documentation
says not to use it to request bigger batches.

## Table functions (UDTFs)

A Python UDTF is a **class**. Snowflake partitions the input with `OVER (PARTITION BY …)`, calls `__init__` once per
partition, `process` once per row (in `ORDER BY` order if you give one), and `end_partition` once at the end. Both `process`
and `end_partition` can yield output rows. Sessionizing clickstream events, a classic job that's awkward in SQL:

```python
import datetime as dt

class Sessionize:
    """UDTF handler: one instance per partition (customer), rows arrive in ORDER BY event_ts order."""
    GAP = dt.timedelta(minutes=30)

    def __init__(self):
        self.session_no, self.last_ts, self.events = 0, None, 0

    def process(self, event_ts, page):
        if self.last_ts is None or event_ts - self.last_ts > self.GAP:
            self.session_no += 1
        self.last_ts = event_ts
        self.events += 1
        yield (self.session_no, event_ts, page)           # one output row per input row

    def end_partition(self):
        yield (None, None, f"summary: {self.events} events in {self.session_no} sessions")
```

```sql
CREATE OR REPLACE FUNCTION web.mart.sessionize(event_ts TIMESTAMP_NTZ, page VARCHAR)
RETURNS TABLE (session_no NUMBER, event_ts TIMESTAMP_NTZ, page VARCHAR)
LANGUAGE PYTHON
RUNTIME_VERSION = '3.12'
HANDLER = 'Sessionize'
AS
$$
# (the class above)
$$;

SELECT e.customer_id, s.session_no, s.event_ts, s.page
FROM web.raw.events e,
     TABLE(web.mart.sessionize(e.event_ts, e.page) OVER (PARTITION BY e.customer_id ORDER BY e.event_ts)) s;
```

The handler's contract can be checked locally without Snowflake. A small driver calls it the way the documentation describes,
with a new instance per partition and rows in order:

```text
101 1 09:00 home
101 1 09:10 product
101 1 09:35 cart
101 2 11:00 home
101 2 11:05 checkout
101 summary: 5 events in 2 sessions
102 1 10:00 home
102 summary: 1 events in 1 sessions
```

Customer 101's 85-minute gap between 09:35 and 11:00 starts session 2. The `end_partition` summary row shows where
per-partition totals belong. (In this layout, summary rows have `NULL` session and timestamp. A real function would
usually emit one or the other, not both.)

## How It Actually Works

When a query calls a Python UDF, each warehouse node starts **Python worker processes** in a secure sandbox, with the
requested runtime and packages installed. The query engine sends rows to the workers, and the workers
send results back. For a scalar UDF the handler runs once per row. For a vectorized UDF, rows are grouped into batches and
converted to pandas DataFrames. For a UDTF, rows are routed so that each partition is processed by one handler instance, in the
requested order. Workers run in parallel across nodes and cores, so a Python UDF scales with the warehouse, but it's still
Python. A built-in SQL function runs inside the engine with no data movement or interpreter at all. That's why the best Python UDF is
often the one you didn't need. When you do need one, memory-heavy work (large models, big libraries) may need a
**Snowpark-optimized** warehouse, which has more memory per node.

## Common mistakes

- **Python for what SQL already does.** `SPLIT_PART`, `REGEXP_SUBSTR` and `TRY_TO_NUMBER` beat any handler.
- **Not handling `None`.** A `NULL` input will crash `email.split`.
- **Vectorizing string logic and expecting speed.** Measure. Vectorization pays off for array math.
- **Unpinned packages** in production functions. Pin versions (`'numpy==2.2.6'`) once a function is in use, so a rebuild
  doesn't silently change results.
- **Loading a model on every call.** Load it at module level or in `__init__` (once per process or partition), not inside the
  handler.
- **UDTFs without `ORDER BY`** when the logic depends on order, such as sessionization.

## Exercise

1. Create `email_domain` in a trial account and compare its run time on `SNOWFLAKE_SAMPLE_DATA.TPCH_SF10.CUSTOMER` (use
   `C_NAME || '@example.com'`) with `SPLIT_PART`, using the Query Profile.
2. Run `handler_bench.py` on your machine. Then add a third case (for example, a regex) and predict which version wins
   before running it.
3. Create the vectorized `churn_score` and a scalar version of it, run both on 10 million generated rows, and compare.
4. Extend `Sessionize` to emit, from `end_partition` only, one row per session with its start, end and number of pages.
   Test it with the local driver first.
