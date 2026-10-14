---
description: "Snowflake Level 3: roles and privileges, authentication, network policies, masking and row access, tags, secure views, sharing, encryption and auditing."
---

# Level 3 · Security, Governance & Sharing

Level 3 is about who may see and do what, and how you prove it. Snowflake gives you a deep toolbox (roles, policies, tags,
secure views, shares, audit views), and most real-world failures come from how the pieces combine rather than from any one of
them. Along the way you'll see administrators inherit raw PII through an innocent-looking role grant, a network policy change
modelled before it can lock anyone out, a key-pair sign-in token built and verified by hand, a view whose safety depends on
the optimizer's plan, and a grant script whose tests found three design faults.

!!! note "What was run"
    Nothing here was run on Snowflake. Effective access through role hierarchies was computed with a small RBAC model and
    tested with pytest. Network policy evaluation was modelled in Python, a key-pair JWT was built and verified locally with
    PyJWT, and the view-leak and audit queries ran in DuckDB 1.5.6 on synthetic data, always labelled as such. All Snowflake
    SQL was syntax-checked and verified against the October 2026 documentation, including areas that changed in 2025–2026
    (the end of single-factor passwords, programmatic access tokens, network rules).

**Prerequisites:** Levels 1 and 2.

## Modules

1. **[Access Control: Roles, Privileges & Hierarchies](01-rbac.md)** — system roles, access and functional roles, future
   grants, managed-access schemas, and effective access computed from a grant script.
2. **[Authentication: MFA, Key Pairs, OAuth & SSO](02-authentication.md)** — the end of single-factor passwords, user types,
   key-pair JWTs, PATs, SSO and authentication policies.
3. **[Network Policies & Private Connectivity](03-network-security.md)** — network rules and policies, precedence modelled
   offline, private connectivity and egress.
4. **[Masking & Row Access Policies](04-masking-row-access.md)** — writing policies, `CURRENT_ROLE` vs
   `IS_ROLE_IN_SESSION`, mapping tables and testing.
5. **[Object Tagging & Data Classification](05-tags-classification.md)** — allowed values, inheritance and propagation, tag-based masking and
   sensitive data classification.
6. **[Secure Views & Secure Functions](06-secure-views-udfs.md)** — how a normal view can leak, what `SECURE` changes and
   costs.
7. **[Secure Data Sharing & the Marketplace](07-data-sharing.md)** — shares, database roles, per-consumer rows, reader
   accounts and listings.
8. **[Encryption, Key Management & Compliance](08-encryption-compliance.md)** — the key hierarchy, rotation vs rekeying,
   Tri-Secret Secure and shared responsibility.
9. **[Auditing with ACCOUNT_USAGE & Access History](09-auditing-account-usage.md)** — login, query and access history,
   latency, and answering "who read this column?".
10. **[Project — A Governed Multi-Team Platform](10-project-governed-platform.md)** — tested role design, tag-based
    masking, region row access, network rules and a PII-holder alert.

## After Level 3

You can secure and govern a shared platform. Level 4 is about building on Snowflake and running it well: UDFs, stored
procedures and Snowpark, Cortex AI functions, cost management, warehouse scaling, replication and failover, CI/CD with dbt
and Snowflake CLI, and migrating from another warehouse, ending with a cost-aware platform capstone.
