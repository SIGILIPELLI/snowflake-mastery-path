---
description: "Signing in to Snowflake: the end of single-factor passwords, user types, MFA, key-pair JWTs built and verified locally, PATs, OAuth, SSO and policies."
---

# 02 · Authentication: MFA, Key Pairs, OAuth & SSO

Authentication answers *who is this?* before access control (the previous lesson) answers *what may they do?*. In
Snowflake this area has changed more in 2025–2026 than in the previous decade, because Snowflake has been **retiring
single-factor password sign-ins** account by account. This lesson covers the rules as they stand, the methods to use for
people and for programs, and what a key-pair sign-in actually sends, built and verified locally.

## The end of single-factor passwords

Snowflake's documentation ("Planning for the deprecation of single-factor password sign-ins") lays out a phased
timeline. Its estimated dates, as published in October 2026:

| Estimated date | Affected users | Phase |
|---|---|---|
| Sep 2025 – Jan 2026 | human users | mandatory MFA for all Snowsight users |
| May 2026 – Jul 2026 | human users, legacy service users | strong authentication for **new** users |
| Aug 2026 – Oct 2026 | human users, legacy service users | strong authentication for **all** users |

When the final phase is enforced, all human users must use a second factor with a password, and non-human users are
blocked from password authentication altogether. Snowflake notifies each account of its enforcement date, and the
documentation notes that some account types (reader accounts, trial accounts and Snowflake Postgres) are excluded.
Because the final phase falls in exactly the months this course was written, **check your account's notification and
the current page** rather than relying on these dates.

The distinction that drives everything is the user's **`TYPE`**:

| `TYPE` | Who | Allowed to authenticate with |
|---|---|---|
| `PERSON` (or unset) | humans | SSO, or password **plus MFA**, or key pair / PAT for programmatic tools |
| `SERVICE` | applications, pipelines | key pair, OAuth, programmatic access tokens, workload identity federation; **no password** |
| `LEGACY_SERVICE` | transitional | password temporarily; being phased out |

```sql
CREATE USER etl_service TYPE = SERVICE DEFAULT_ROLE = data_engineer DEFAULT_WAREHOUSE = etl_wh;
ALTER USER old_bi_tool SET TYPE = LEGACY_SERVICE;     -- buys time; plan the migration
SHOW USERS;                                           -- check the type column across the account
```

## MFA for people

Snowflake supports three second factors: **passkeys** (which Snowflake's documentation recommends for security and
usability), **authenticator apps** generating time-based one-time codes (TOTP), and **Duo**. Users enroll in Snowsight.
For SSO users, MFA is normally enforced by the identity provider, and an authentication policy decides whether Snowflake
additionally requires its own.

## Key pairs for programs, and what they actually send

Lesson 02 of Level 1 set up key-pair authentication for Snowflake CLI. Here is what a driver does with the key, built
locally from Snowflake's documented claim format and verified the way the server verifies it. Nothing is sent anywhere:

```python
import base64, hashlib, time, json
import jwt                                            # PyJWT
from cryptography.hazmat.primitives import serialization

private_key = serialization.load_pem_private_key(open("rsa_key.p8", "rb").read(), password=None)
public_der = private_key.public_key().public_bytes(serialization.Encoding.DER,
                                                   serialization.PublicFormat.SubjectPublicKeyInfo)
fingerprint = "SHA256:" + base64.b64encode(hashlib.sha256(public_der).digest()).decode()

account, user = "MYORG-MYACCOUNT", "ANA"              # upper case, '.' in the account replaced by '-'
now = int(time.time())
claims = {"iss": f"{account}.{user}.{fingerprint}", "sub": f"{account}.{user}", "iat": now, "exp": now + 3600}
token = jwt.encode(claims, private_key, algorithm="RS256")
print("fingerprint:", fingerprint)
print("JWT header :", json.dumps(jwt.get_unverified_header(token)))
print("JWT claims :", json.dumps({k: claims[k] for k in ("iss", "sub")}))
print("token length:", len(token), "characters")
decoded = jwt.decode(token, private_key.public_key(), algorithms=["RS256"])   # what the server does
print("verified with public key:", decoded["sub"])
```

```text
fingerprint: SHA256:xWy4qmg9a9Y2zY8a0g/qRNp2QGRbDv7emETjmB0BSSU=
JWT header : {"alg": "RS256", "typ": "JWT"}
JWT claims : {"iss": "MYORG-MYACCOUNT.ANA.SHA256:xWy4qmg9a9Y2zY8a0g/qRNp2QGRbDv7emETjmB0BSSU=", "sub": "MYORG-MYACCOUNT.ANA"}
token length: 571 characters
verified with public key: MYORG-MYACCOUNT.ANA
token from another key: InvalidSignatureError
```

The fingerprint is the same one computed in Level 1 · 02, because it's the same throwaway key. The token names the user
and the **fingerprint of the key that signed it**, so Snowflake knows which registered public key to check the signature
against. A token signed by any other key fails verification (the last line, from a second test in the same script).
Snowflake's documentation adds two rules seen in the code: the account and user must be **upper case**, and the token is
valid for **at most one hour** whatever `exp` says. Drivers create and refresh these tokens for you. Building one by hand
is only needed for raw HTTP clients such as the SQL API.

### Rotating keys without downtime

A user can have two registered public keys at once, which makes rotation a three-step change with no outage:

```sql
ALTER USER etl_service SET RSA_PUBLIC_KEY_2 = 'MIIBIjANBgkqh...new key...';   -- 1. add the new key
-- 2. deploy the new private key to the application and confirm it signs in (DESC USER shows both fingerprints)
ALTER USER etl_service UNSET RSA_PUBLIC_KEY;                                  -- 3. remove the old key
```

## Programmatic access tokens

A **programmatic access token (PAT)** is a secret string a program can use in place of a password: for drivers, the SQL
API and tools that only accept a "password" field.

```sql
ALTER USER etl_service ADD PROGRAMMATIC ACCESS TOKEN etl_pipeline
  ROLE_RESTRICTION = 'DATA_ENGINEER'
  DAYS_TO_EXPIRY = 30
  COMMENT = 'nightly ELT job';
```

From the documentation: tokens expire after **15 days by default**, can be given up to **365 days** (both adjustable with an
authentication policy), and by default a user must be subject to a **network policy** to generate or use one. For service
users that's mandatory, which ties the token to known network locations. `ROLE_RESTRICTION` limits what the token can do
to one role. The token value is shown **once**, at creation, so store it in a secret manager immediately.

## OAuth, SSO and workload identity

- **SSO for people.** SAML 2.0 (or OIDC) through your identity provider (Okta, Microsoft Entra ID and others), configured as a
  security integration. Users sign in through the identity provider, which also enforces its own MFA. Combined with SCIM
  provisioning, users and role memberships are managed in the identity provider rather than in Snowflake.
- **OAuth for applications acting for a user.** *Snowflake OAuth* (Snowflake issues the tokens, for partner tools like BI
  products) or *External OAuth* (your identity provider issues them).
- **Workload identity federation** lets workloads running in AWS, Azure or GCP authenticate with their cloud identity
  instead of a stored secret. It wasn't exercised here, so see Snowflake's documentation for the current providers.

## Authentication policies

An **authentication policy** restricts how users may sign in: which clients, which methods, and MFA requirements. Set it on
the account and override it for specific users:

```sql
CREATE AUTHENTICATION POLICY humans_sso_or_mfa
  CLIENT_TYPES = ('SNOWFLAKE_UI', 'DRIVERS', 'SNOWFLAKE_CLI')
  AUTHENTICATION_METHODS = ('SAML', 'PASSWORD')
  MFA_ENROLLMENT = 'REQUIRED';

CREATE AUTHENTICATION POLICY services_keypair_only
  AUTHENTICATION_METHODS = ('KEYPAIR', 'PROGRAMMATIC_ACCESS_TOKEN');

ALTER ACCOUNT SET AUTHENTICATION POLICY humans_sso_or_mfa;
ALTER USER etl_service SET AUTHENTICATION POLICY services_keypair_only;
```

Snowflake's guidance includes keeping a separate, less restrictive policy for a break-glass administrator, so that a
misconfigured account-level policy can't lock everyone out.

## How It Actually Works

Every sign-in goes to the cloud services layer, which applies, in order: **network policy** (is this IP or private
endpoint allowed? next lesson), **authentication policy** (is this client and method allowed for this user?), the
**method itself** (verify a password plus second factor, a JWT signature against the registered public keys, an OAuth or
SAML assertion against the integration's trusted issuer, a PAT against its stored hash), and the user's **type**
restrictions. Success creates a session token tied to the user, and from then on access control works on the session's
roles. Key pairs and workload identity are the strongest options for programs because **no reusable secret is ever sent**:
each sign-in carries a fresh, short-lived signed token.

## Common mistakes

- **Service accounts using passwords**, often with `TYPE` unset (treated as a human). Make them `TYPE = SERVICE` with key
  pairs, and plan ahead for the enforcement date.
- **Shared human logins for tools.** Each tool gets its own service user, with its own policy and role.
- **PATs without expiry discipline**, or stored in code. Use short expiry, a role restriction and a secret manager.
- **Rotating keys by replacing the only key.** Use `RSA_PUBLIC_KEY_2` and overlap.
- **An account-wide authentication policy with no break-glass exception.**
- **Assuming last year's rules.** This area changed through 2025–2026. Re-read the current pages.

## Exercise

1. List every user in a trial account with its `TYPE`, `has_password`, `has_rsa_public_key` and `has_mfa` columns from
   `SHOW USERS` (via `RESULT_SCAN`). Which users would be blocked under the final enforcement phase?
2. Create a `SERVICE` user with a key pair, then rotate it to a second key with zero failed sign-ins. Prove it with your
   pipeline's logs and `DESC USER`.
3. Extend `make_jwt.py` to read the account and user from your Snowflake CLI connection, then call the Snowflake SQL API
   with the token and `X-Snowflake-Authorization-Token-Type: KEYPAIR_JWT` to run `SELECT CURRENT_USER()`.
4. Write the authentication policies for your organization: humans via SSO, services via key pair or PAT, and one
   break-glass administrator. Explain the precedence.
