---
description: "Cortex AI functions in SQL: AI_CLASSIFY, AI_FILTER, AI_SENTIMENT, AI_COMPLETE with structured output, model access, data residency and token costs."
---

# 04 · Cortex AI Functions

Support tickets, reviews, call notes, contracts: most companies have more text than anyone reads. **Cortex AI functions** run
large language models **from SQL**, inside Snowflake's service perimeter, so classifying a million tickets is a `SELECT` rather than
a pipeline to an external API. This lesson covers the functions, who may call them, where the data goes, and the part that
surprises teams on their first bill: how tokens add up.

!!! warning "Not run for this course"
    Cortex functions need a Snowflake account in a supported region, so none were called here and no model outputs are
    shown. Syntax, privileges, billing rules and limits come from Snowflake's documentation as of October 2026. This area
    changes monthly: functions move from preview to GA, and models are added and retired. Check the function reference and
    the regional availability page before relying on any detail.

## The functions

Per the documentation, the task-specific functions include:

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

Some are preview features. The documentation marks each one, and only GA functions are meant for production.

## Examples on support tickets

```sql
-- route tickets: one label each, with descriptions to sharpen the boundaries
SELECT ticket_id,
       AI_CLASSIFY(body,
         [ {'label': 'billing', 'description': 'charges, refunds, invoices, payment failures'},
           {'label': 'bug',     'description': 'a feature that worked before or should work does not work'},
           {'label': 'how-to',  'description': 'questions about how to use an existing feature'},
           {'label': 'account', 'description': 'users, admins, access, sign-in and workspace settings'} ],
         {'task_description': 'Route customer support tickets for a SaaS product'}
       ):labels[0]::VARCHAR AS queue
FROM support.raw.tickets;

-- a semantic WHERE clause
SELECT ticket_id, body
FROM support.raw.tickets
WHERE created_at >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND AI_FILTER(PROMPT('The customer says they are considering cancelling: {0}', body));

-- aspect-based sentiment
SELECT ticket_id, AI_SENTIMENT(body, ['pricing', 'support', 'reliability']) AS sentiment
FROM support.raw.tickets;

-- structured extraction with a typed response format
SELECT ticket_id,
       AI_COMPLETE(
         model => 'llama3.3-70b',
         prompt => 'Extract the order number and the product mentioned in this ticket: ' || body,
         response_format => TYPE OBJECT(order_number STRING, product STRING),
         model_parameters => {'temperature': 0}
       ) AS extracted
FROM support.raw.tickets;

-- one answer across many rows (not limited by a single context window, per the documentation)
SELECT AI_AGG(body, 'List the three most common reasons for refund requests, with one example each')
FROM support.raw.tickets
WHERE created_at >= DATEADD(month, -1, CURRENT_TIMESTAMP());
```

Two behaviours to plan for. **Errors return `NULL` by default**, row by row, and the query still succeeds. Pass
`return_error_details => TRUE` (the last argument) to get an object with the value or the error, and count the failures.
And outputs are **probabilistic**. `temperature` defaults to 0 for `AI_COMPLETE`, but a model or function version change can
still change results, so store the outputs you depend on rather than recomputing them.

## Who may call them

From the documentation on privileges:

- A role needs **both** the account privilege `USE AI FUNCTIONS` **and** one of the database roles `SNOWFLAKE.CORTEX_USER` or
  `SNOWFLAKE.AI_FUNCTIONS_USER`.
- By default, `USE AI FUNCTIONS` and `CORTEX_USER` are **granted to `PUBLIC`**, so **every user can call them**.
  `AI_FUNCTIONS_USER` isn't.
- **Which models** a role can use is controlled by **model RBAC**: application roles such as
  `SNOWFLAKE."CORTEX-MODEL-ROLE-LLAMA3.3-70B"`. The older account parameter `CORTEX_MODELS_ALLOWLIST` is being deprecated
  (since August 2026 it can only be set to `'None'`).
- `ACCOUNTADMIN` always has access to all models, including as a **secondary** role. Test restrictions with another role and
  `USE SECONDARY ROLES NONE`.

```sql
USE ROLE ACCOUNTADMIN;
REVOKE USE AI FUNCTIONS ON ACCOUNT FROM ROLE PUBLIC;
REVOKE DATABASE ROLE SNOWFLAKE.CORTEX_USER FROM ROLE PUBLIC;

CREATE ROLE IF NOT EXISTS ai_user;
GRANT USE AI FUNCTIONS ON ACCOUNT TO ROLE ai_user;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE ai_user;
GRANT APPLICATION ROLE SNOWFLAKE."CORTEX-MODEL-ROLE-LLAMA3.3-70B" TO ROLE ai_user;
GRANT ROLE ai_user TO ROLE support_analytics;

SHOW APPLICATION ROLES LIKE 'CORTEX-MODEL%' IN APPLICATION SNOWFLAKE;
```

Whether to lock it down is a governance decision, not a technical one. Leaving it open means anyone who can read a table can
send it to a model, and spend credits doing so. Masking and row access policies (Level 3) still apply to what a role can read,
and so to what it can send.

## Where the data goes

The models run inside Snowflake's service perimeter, but not necessarily in **your region**. If a model isn't available
locally, **cross-region inference** (account parameter `CORTEX_ENABLED_CROSS_REGION`) can route the request elsewhere. Per
the documentation, data at rest stays in your account's region. The prompt and response travel encrypted to the processing
region and aren't persisted there, and billing happens in your region. The default depends on the account: new
organizations created after March 9, 2026 default to `ANY_REGION`. If your obligations require in-region processing, check the
parameter and set it explicitly:

```sql
SHOW PARAMETERS LIKE 'CORTEX_ENABLED_CROSS_REGION' IN ACCOUNT;
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'DISABLED';      -- or a list of allowed regions
```

## What it costs

Cortex AI functions are billed **per token** (credits per million tokens, by function and model, in Snowflake's Service
Consumption Table), **plus** the warehouse that runs the query. The documented rules that matter most:

- For generating functions (`AI_COMPLETE`, `AI_CLASSIFY`, `AI_FILTER`, `AI_AGG`, `AI_SUMMARIZE`, `AI_TRANSLATE`), **input and
  output tokens** are both billed. For embeddings and similarity, input only.
- Task-specific functions **add their own prompt** to your text, so billed tokens exceed your text.
- For `AI_CLASSIFY`, the **labels, descriptions and examples are billed as input for every row**, not once per query.
- Snowflake recommends a warehouse **no larger than MEDIUM**. A larger one doesn't make these functions faster.

The third rule is easy to overlook, because the labels look like a constant in the query. A lower-bound model of input
tokens for the routing query above over one million tickets, using the documented four-characters-per-token convention:

```python
# Lower-bound input-token model for AI_CLASSIFY over 1,000,000 tickets, using the documented ~4 characters per token
# convention and the documented rule that labels, descriptions and examples are billed as input FOR EVERY ROW.
# The function's own hidden prompt (also billed, per the docs) is unknown and left out, so real numbers are higher.
import json
tok = lambda s: len(s) / 4
tickets = ["My invoice for September shows a double charge for the Pro plan, please refund one of them.",
           "The export to CSV button does nothing since yesterday's update.",
           "How do I add a second admin to our workspace?",
           "Card declined again although the bank says everything is fine. Order 88231."]
avg_ticket = sum(map(tok, tickets)) / len(tickets)

labels = ["billing", "bug", "how-to", "account"]
described = [{"label": "billing", "description": "charges, refunds, invoices, payment failures"},
             {"label": "bug", "description": "a feature that worked before or should work does not work"},
             {"label": "how-to", "description": "questions about how to use an existing feature"},
             {"label": "account", "description": "users, admins, access, sign-in and workspace settings"}]
examples = [{"input": t, "labels": [l], "explanation": "matches the category description"}
            for t, l in zip(tickets[:3], ["billing", "bug", "how-to"])]

ROWS = 1_000_000
variants = {"labels only": tok(json.dumps(labels)),
            "+ descriptions": tok(json.dumps(described)),
            "+ descriptions + 3 examples": tok(json.dumps(described)) + tok(json.dumps(examples))}
print(f"average ticket: {avg_ticket:.0f} tokens")
for name, extra in variants.items():
    per_row = avg_ticket + extra
    print(f"{name:<28} {per_row:6.0f} tokens/row  {per_row * ROWS / 1e6:7.1f} M input tokens  ({per_row / avg_ticket:.1f}x the tickets alone)")
```

```text
average ticket: 17 tokens
labels only                      27 tokens/row     26.9 M input tokens  (1.6x the tickets alone)
+ descriptions                  107 tokens/row    106.9 M input tokens  (6.2x the tickets alone)
+ descriptions + 3 examples     222 tokens/row    222.1 M input tokens  (13.0x the tickets alone)
```

With short tickets, the **category definitions dominate**: descriptions multiply input tokens about six times, and three
few-shot examples about thirteen times, before the function's own prompt is counted. Descriptions and examples often improve
accuracy, so the answer isn't to drop them. Measure accuracy with and without them on a labelled sample, and pay for the
version that earns it. Then cut the volume:

- **Classify each ticket once.** Run the function from a stream-driven task (Level 2 · 06–07) and store the result, so
  dashboards read a column instead of calling a model.
- **Filter with SQL first.** A `WHERE` on date, language or product before `AI_FILTER` sends fewer rows to the model.
- **Measure tokens before scaling.** `AI_COUNT_TOKENS` returns the token count for a function and input, at warehouse cost
  only.

Track actual spend in ACCOUNT_USAGE (with its usual latency, Level 3 · 09):

```sql
SELECT function_name, model_name, SUM(tokens) AS tokens, SUM(token_credits) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_QUERY_USAGE_HISTORY
GROUP BY function_name, model_name
ORDER BY credits DESC;
```

## How It Actually Works

An AI function call is part of an ordinary query. The warehouse reads and filters the rows, and for each input that reaches the
function, Snowflake builds a request (your text plus the function's prompt, labels or schema) and sends it to the model-serving
service, which Snowflake optimizes for throughput. Results come back as values in the query's output. That's why the documentation
recommends batch use over interactive calls, why a larger warehouse doesn't help (the model service is the bottleneck), and why
billing has two parts: warehouse time for the query, and tokens for the model. Access checks happen at both ends. Privileges
decide whether the role may call the function and use the model, and the usual table privileges and policies decide which rows
and values can be read and sent at all.

## Common mistakes

- **Leaving the default PUBLIC grants without deciding to.** Every user can send any table they can read to a model.
- **Recomputing model outputs in dashboards.** Classify once, store the result.
- **Ignoring `NULL` results.** Errors are silent by default. Count them.
- **Long label descriptions and many examples on large tables** without measuring their value. They're billed per row.
- **Large warehouses for AI queries.** Medium or smaller, per the documentation.
- **Assuming in-region processing.** Check `CORTEX_ENABLED_CROSS_REGION`.
- **Treating outputs as deterministic facts.** Evaluate against a labelled sample, and re-evaluate when a model changes.

## Exercise

1. In a trial account, check who can call AI functions (`SHOW GRANTS OF DATABASE ROLE SNOWFLAKE.CORTEX_USER`) and what
   `CORTEX_ENABLED_CROSS_REGION` is set to. Write down whether either should change for your organization.
2. Label 50 synthetic tickets by hand. Run `AI_CLASSIFY` with labels only, with descriptions, and with examples, and record
   accuracy and `TOKENS` from the usage view for each. Which version would you pay for?
3. Build the "classify once" pipeline: a stream on `tickets`, a task that inserts `AI_CLASSIFY` results for new rows into
   `ticket_labels`, and a check that re-running the task classifies nothing.
4. Use `AI_COUNT_TOKENS` on your ticket sample and compare it with `classify_tokens.py`'s four-characters estimate.
