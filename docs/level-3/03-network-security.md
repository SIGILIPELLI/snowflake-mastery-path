---
description: "Snowflake network rules and policies, blocked-first evaluation and precedence modelled offline, private connectivity, and egress for external access."
---

# 03 · Network Policies & Private Connectivity

Authentication proves who someone is. A **network policy** decides **where they may connect from**. A stolen key used from
an attacker's laptop is still a valid key, but if the account only accepts connections from your VPN and your pipeline's
NAT gateway, it doesn't get in. Network policies are one of the cheapest, highest-value controls in Snowflake, and since
2025–2026 they're also a **prerequisite** for features such as programmatic access tokens for service users (Level 3 · 02).

## Network rules and network policies

Snowflake separates *what* an address is from *what you do with it*:

- A **network rule** is a schema-level object that names a set of network identifiers: IPv4/IPv6 ranges, AWS VPC endpoint
  IDs, Azure private-link IDs, GCP Private Service Connect IDs (for inbound), or host:port pairs (for outbound).
- A **network policy** combines rules into an **allowed list** and a **blocked list**, and is attached to the account, a
  user, or a security integration.

```sql
USE ROLE SECURITYADMIN;
CREATE DATABASE IF NOT EXISTS security_db;
CREATE SCHEMA IF NOT EXISTS security_db.network;

CREATE NETWORK RULE security_db.network.corp_vpn      TYPE = IPV4 MODE = INGRESS VALUE_LIST = ('203.0.113.0/24');
CREATE NETWORK RULE security_db.network.office_lisbon TYPE = IPV4 MODE = INGRESS VALUE_LIST = ('198.51.100.16/28');
CREATE NETWORK RULE security_db.network.etl_nat       TYPE = IPV4 MODE = INGRESS VALUE_LIST = ('192.0.2.10/32');
CREATE NETWORK RULE security_db.network.known_bad     TYPE = IPV4 MODE = INGRESS VALUE_LIST = ('203.0.113.66/32');

CREATE NETWORK POLICY account_default
  ALLOWED_NETWORK_RULE_LIST = ('security_db.network.corp_vpn', 'security_db.network.office_lisbon')
  BLOCKED_NETWORK_RULE_LIST = ('security_db.network.known_bad');

CREATE NETWORK POLICY etl_only
  ALLOWED_NETWORK_RULE_LIST = ('security_db.network.etl_nat');

ALTER ACCOUNT SET NETWORK_POLICY = account_default;
ALTER USER etl_service SET NETWORK_POLICY = etl_only;
```

(The older `ALLOWED_IP_LIST` and `BLOCKED_IP_LIST` properties still work, but Snowflake's documentation says new policies
should use network rules, and advises against mixing both in one policy.)

## How a request is judged

From Snowflake's documentation:

1. **The blocked list is applied first.** An address that matches a blocked rule is refused, even if it's also in the allowed
   list.
2. Otherwise the address must match the **allowed list**.
3. When policies exist at several levels, **the most specific one wins**: a user's policy overrides the account's
   (security-integration policies have their own precedence rules).
4. For requests arriving over **private connectivity**, if the allowed list contains private-endpoint rules (`AWSVPCEID`,
   `AZURELINKID`), IPv4/IPv6 rules are ignored for those requests.

These rules can be modelled exactly, which lets you test a policy change before applying it to an account where a mistake
locks everyone out. A small evaluator (Python, `ipaddress` module, run locally), configured with the rules and policies
above:

```python
"""Evaluate Snowflake-style network policies offline (a model of the documented rules):
- a policy has allowed and blocked network rules (IPv4 CIDR lists here);
- the blocked list is applied first; then the request must match the allowed list;
- the most specific policy wins: user > account (security-integration policies are not modelled)."""
import ipaddress

RULES = {   # network rule name -> CIDR list (as in CREATE NETWORK RULE ... VALUE_LIST)
    "corp_vpn":       ["203.0.113.0/24"],
    "office_lisbon":  ["198.51.100.16/28"],
    "etl_nat":        ["192.0.2.10/32"],
    "known_bad":      ["203.0.113.66/32"],
}
POLICIES = {
    "account_default": {"allowed": ["corp_vpn", "office_lisbon"], "blocked": ["known_bad"]},
    "etl_only":        {"allowed": ["etl_nat"],                   "blocked": []},
}
ACCOUNT_POLICY = "account_default"
USER_POLICY = {"ETL_SERVICE": "etl_only"}          # ALTER USER etl_service SET NETWORK_POLICY = etl_only

def in_rules(ip, rule_names):
    return any(ipaddress.ip_address(ip) in ipaddress.ip_network(c) for r in rule_names for c in RULES[r])

def decide(user, ip):
    name = USER_POLICY.get(user, ACCOUNT_POLICY)
    p = POLICIES[name]
    if in_rules(ip, p["blocked"]):
        return name, "DENY (blocked list)"
    return name, "ALLOW" if in_rules(ip, p["allowed"]) else "DENY (not in allowed list)"

for user, ip, note in [("ANA", "203.0.113.25", "on the VPN"),
                       ("ANA", "203.0.113.66", "VPN address on the blocked list"),
                       ("ANA", "198.51.100.20", "Lisbon office"),
                       ("ANA", "198.51.100.40", "outside the office /28"),
                       ("ETL_SERVICE", "192.0.2.10", "pipeline's NAT gateway"),
                       ("ETL_SERVICE", "203.0.113.25", "stolen key used from the VPN")]:
    policy, verdict = decide(user, ip)
    print(f"{user:12} {ip:15} {note:34} policy={policy:16} {verdict}")
```

```text
ANA          203.0.113.25    on the VPN                         policy=account_default  ALLOW
ANA          203.0.113.66    VPN address on the blocked list    policy=account_default  DENY (blocked list)
ANA          198.51.100.20   Lisbon office                      policy=account_default  ALLOW
ANA          198.51.100.40   outside the office /28             policy=account_default  DENY (not in allowed list)
ETL_SERVICE  192.0.2.10      pipeline's NAT gateway             policy=etl_only         ALLOW
ETL_SERVICE  203.0.113.25    stolen key used from the VPN       policy=etl_only         DENY (not in allowed list)
```

Two lines are worth a second look. `203.0.113.66` is inside the VPN range **and** on the blocked list, and blocked wins.
And `ETL_SERVICE` is denied from the VPN, an address the account policy allows, because its own user-level policy is the
most specific one and allows only the NAT gateway. That's exactly the protection you want for a service user's key: it's
useless outside the pipeline's network.

## Don't lock yourself out

A network policy on the account applies to **every** user, administrators included. Before `ALTER ACCOUNT SET
NETWORK_POLICY`:

- confirm your own current address is allowed (`SELECT CURRENT_IP_ADDRESS();`);
- set it first on a test user (`ALTER USER test_user SET NETWORK_POLICY = …`) and sign in as that user;
- keep a break-glass administrator with a user-level policy that allows a known, controlled location;
- remember that BI tools, ETL services and Snowflake Native Apps connecting from vendor IP ranges need rules too.

## Private connectivity

Business Critical Edition (and above) supports private connectivity, such as **AWS PrivateLink**, **Azure Private Link**
and **Google Cloud Private Service Connect**, so traffic from your cloud network reaches Snowflake without crossing the
public internet. Combined with network rules of type `AWSVPCEID`/`AZURELINKID`/`GCPPSCID` and a policy that allows only those,
the account becomes unreachable from the internet entirely. Setting it up involves your cloud account's networking and DNS,
and Snowflake's system functions for authorizing endpoints, which weren't exercised for this course. Follow the
provider-specific guides.

Internal stages can be restricted to private endpoints as well (network rules in `INTERNAL_STAGE` mode, with an account
parameter enabled), which matters because `PUT`/`GET` traffic goes to the stage's cloud storage, not to Snowflake's
service endpoint.

## Egress: controlling what Snowflake can reach

Rules with `MODE = EGRESS` and `TYPE = HOST_PORT` describe **outbound** destinations. They're used with **external
access integrations**, which let Python UDFs and procedures call external APIs, so code in Snowflake can only reach hosts
you've explicitly listed:

```sql
CREATE NETWORK RULE security_db.network.fx_api_egress
  MODE = EGRESS TYPE = HOST_PORT VALUE_LIST = ('api.exchangerate.example:443');

CREATE EXTERNAL ACCESS INTEGRATION fx_api_access
  ALLOWED_NETWORK_RULES = (security_db.network.fx_api_egress)
  ENABLED = TRUE;
```

Level 4 · 03 uses this from a Python function.

## How It Actually Works

Network policy checks happen in the cloud services layer at **authentication time**, before credentials are even evaluated
(Level 3 · 02). The service determines the request's source (the client's public IP, or the private endpoint ID it arrived
through), finds the most specific policy for the user or integration involved, evaluates blocked rules, then allowed rules,
and refuses the connection if they don't permit it. (How a policy change affects sessions that are already open isn't
covered here. Test it in your own account before relying on it in an incident.) Egress rules are enforced differently: the sandbox that runs UDFs and procedures can only
open network connections to destinations listed in an external access integration the function was created with.

## Common mistakes

- **Applying an untested account policy** and locking out administrators and every integration.
- **`0.0.0.0/0` in an allowed list**, "temporarily".
- **No user-level policies for service users**, which leaves stolen keys usable from anywhere the account allows.
- **Mixing old IP lists and network rules** in one policy.
- **Forgetting vendor IPs** for SaaS tools that connect to Snowflake.
- **Securing the service endpoint but not internal stages** in a private-connectivity design.

## Exercise

1. Extend `netpolicy.py` to support IPv6 rules and a security-integration level, and add a test where a user policy is
   *less* restrictive than the account policy. Which one applies?
2. In a trial account, create a network rule for your current IP (from `CURRENT_IP_ADDRESS()`), a policy, and apply it to a
   test user. Sign in as that user from another network (a phone hotspot) and observe the refusal.
3. Write a query over `SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY` that lists failed sign-ins in the last 7 days by
   `client_ip` and `error_message`. Which ones look like network-policy blocks?
4. Design the network policies for an account used by an office, a VPN, two SaaS BI tools and one ETL service, including a
   break-glass path.
