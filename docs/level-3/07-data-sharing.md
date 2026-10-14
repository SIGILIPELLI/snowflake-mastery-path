---
description: "Snowflake Secure Data Sharing: shares and database roles, consumer databases, per-consumer rows, provider-paid reader accounts, listings, cross-region."
---

# 07 · Secure Data Sharing & the Marketplace

Sending data to another company usually means exports, files, pipelines and stale copies. In Snowflake, a provider can
**share** live tables and views with another Snowflake account, and the consumer queries them directly. Per Snowflake's
documentation, **no data is copied or transferred**: sharing works through the services layer and metadata store, shared
data takes no storage in the consumer's account, and the consumer pays only for the compute it uses to query. This
lesson covers shares from both sides, sharing different rows with different consumers, reader accounts, and listings.

## The provider side

A **share** is an object that bundles privileges on database objects and the list of consumer accounts:

```sql
USE ROLE ACCOUNTADMIN;                 -- or a role granted CREATE SHARE

CREATE SHARE sales_partner_share COMMENT = 'Daily sales for partner retailers';

-- option 1 (recommended): grant through a database role, which the share then receives
CREATE DATABASE ROLE IF NOT EXISTS sales.partner_reader;
GRANT USAGE ON SCHEMA sales.shared TO DATABASE ROLE sales.partner_reader;
GRANT SELECT ON VIEW sales.shared.partner_daily_sales TO DATABASE ROLE sales.partner_reader;

GRANT USAGE ON DATABASE sales TO SHARE sales_partner_share;
GRANT DATABASE ROLE sales.partner_reader TO SHARE sales_partner_share;

-- add consumers (organization-based account identifiers)
ALTER SHARE sales_partner_share ADD ACCOUNTS = partnerorg.retail_eu, partnerorg.retail_us;

SHOW SHARES;
DESC SHARE sales_partner_share;
```

The documentation also allows granting privileges **directly** to the share (`GRANT SELECT ON VIEW … TO SHARE …`).
Database roles are the better unit. A provider can define several (say `partner_reader` and `partner_finance`), and the
consumer can map each to different roles on their side.

Rules worth knowing:

- Shared objects are **read-only** for consumers: they can't insert, update or delete.
- A share accepts **secure views** (and other secure objects) by default. Shares created with `SECURE_OBJECTS_ONLY = FALSE`
  can include ordinary views, for performance. In that case the optimizer protections from lesson 06 no longer apply, so
  only do it when the consumer is allowed to see everything the view reads.
- A **direct share works only between accounts in the same region**. Across regions or clouds, use a **listing** with
  cross-cloud auto-fulfillment, or replicate the data to an account in the consumer's region and share from there.
- Revoking a grant or removing an account from the share takes effect immediately.

## The consumer side

The consumer sees the share as available and creates a **read-only database** from it:

```sql
SHOW SHARES;                                     -- inbound shares appear with kind = INBOUND

CREATE DATABASE partner_sales FROM SHARE providerorg.provider_account.sales_partner_share;

GRANT DATABASE ROLE partner_sales.partner_reader TO ROLE analyst;    -- map provider database roles to local roles

SELECT * FROM partner_sales.shared.partner_daily_sales WHERE sale_date >= CURRENT_DATE - 7;
```

(Shares built without database roles are granted locally with `GRANT IMPORTED PRIVILEGES ON DATABASE partner_sales TO ROLE
analyst`.) The consumer runs the queries on **its own warehouses** and pays for that compute. The provider pays for storage,
just as before sharing.

## Different rows for different consumers

A single share to many consumers usually needs each consumer to see only their own rows. Inside a shared secure view,
`CURRENT_ACCOUNT()` returns the **consumer's** account (its account locator), which turns into an entitlement filter.
Snowflake's own sharing example uses exactly this pattern:

```sql
CREATE TABLE IF NOT EXISTS sales.shared.partner_entitlements (consumer_account VARCHAR, partner_id NUMBER);
INSERT INTO sales.shared.partner_entitlements VALUES ('XY11111', 101), ('XY22222', 202);   -- consumer account locators

CREATE OR REPLACE SECURE VIEW sales.shared.partner_daily_sales AS
SELECT s.sale_date, s.partner_id, s.sku, s.units, s.revenue
FROM sales.mart.partner_daily_sales s
JOIN sales.shared.partner_entitlements e
  ON e.partner_id = s.partner_id
 AND e.consumer_account = CURRENT_ACCOUNT();
```

The view must be `SECURE`, or the entitlement logic could be bypassed (lesson 06), and the entitlement table stays in the
provider account, out of the share. Onboarding a partner becomes two steps: add their account to the share, and insert their
entitlement row. Test it as a consumer. Snowflake offers a simulation for exactly this, which per the documentation only
works when the current role **owns** the view:

```sql
ALTER SESSION SET SIMULATED_DATA_SHARING_CONSUMER = 'XY11111';
SELECT DISTINCT partner_id FROM sales.shared.partner_daily_sales;       -- expect only 101
ALTER SESSION UNSET SIMULATED_DATA_SHARING_CONSUMER;
```

Whatever identifier you store in the entitlement table, it must be the one the context function returns in the consumer's
account. Verify with the simulation before onboarding a real partner.

## Reader accounts: sharing with someone who has no Snowflake account

A **reader account** is a Snowflake account the provider creates and manages for a consumer without one:

```sql
CREATE MANAGED ACCOUNT partner_acme_reader
  ADMIN_NAME = acme_admin, ADMIN_PASSWORD = '<a strong generated password>',
  TYPE = READER;
```

Snowflake's documentation is direct about the cost: the **provider pays for all credits** consumed by warehouses in the reader
account, and they "can consume an unlimited number of credits", so set up a **resource monitor** on every reader-account
warehouse (Level 4 · 05). A reader account is primarily for querying the provider's shared data. It's not a general-purpose
account.

## Listings and the Marketplace

A **listing** wraps a share with product metadata (description, sample queries, refresh schedule, terms) and offers it
privately to named accounts or publicly on the **Snowflake Marketplace**. Listings are also how data reaches consumers in
other regions and clouds, through **cross-cloud auto-fulfillment**, which replicates the data where it's needed. Paid
listings, usage metrics and consumer analytics are part of the provider experience. Listings are managed mainly in Snowsight's
provider studio (with SQL and API equivalents). They weren't exercised for this course.

As a **consumer**, the Marketplace is a source of third-party data (weather, financial, demographic) that appears as a database
in your account, without a pipeline.

## How It Actually Works

A share is metadata: a list of objects, the privileges granted on them, and the consumer accounts. When a consumer creates a
database from a share, Snowflake creates a catalogue entry in the consumer's account that **points at the provider's
micro-partitions**. Queries from the consumer are compiled and pruned by the services layer exactly like local queries,
but the micro-partition files they read belong to the provider's tables, and they're executed on the consumer's warehouse.
That's why there's nothing to copy, why the data is always current (a provider commit creates a new table version that
consumers see immediately), and why revoking access is instant (the pointer is removed). Because both accounts must reach the
same storage, direct shares are limited to one region. Cross-region sharing needs the micro-partitions replicated to the
consumer's region first, which is what listing auto-fulfillment and database replication do (Level 4 · 07).

## Common mistakes

- **Sharing raw tables** when a secure view with an entitlement filter is what the contract allows.
- **Non-secure views in a share** without understanding what `SECURE_OBJECTS_ONLY = FALSE` gives up.
- **Entitlement filters with no test** for each consumer. Use the simulation parameter.
- **Reader accounts without resource monitors.** The provider pays for unlimited compute.
- **Expecting a direct share to reach another region.** Use a listing or replication.
- **Breaking consumers with schema changes.** Shared views are an API. Version them (`partner_daily_sales_v2`) and announce
  changes.

## Exercise

1. With two trial accounts in the same region (or one account using `SIMULATED_DATA_SHARING_CONSUMER`), share the course's
   daily revenue through a secure view and a database role, and query it from the consumer.
2. Implement the entitlement view for three partners and prove with the simulation that each sees only its own rows.
3. Create a reader account, put a 10-credit monthly resource monitor on its warehouse, and explain who receives the alerts.
4. Draft a listing description for the shared dataset: contents, refresh schedule, sample queries and change policy.
