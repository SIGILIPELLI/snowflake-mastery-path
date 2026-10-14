---
description: "Snowflake encryption at rest and in transit, the key hierarchy, 30-day rotation vs yearly rekeying, Tri-Secret Secure, and your side of compliance."
---

# 08 · Encryption, Key Management & Compliance

Security reviews of a Snowflake deployment almost always ask the same questions. Is the data encrypted? With whose keys?
How are keys rotated? Can we revoke Snowflake's access? Which compliance programmes does it support? Most of the answers
are "Snowflake does it by default", and a few depend on your **edition** and on choices you make. This lesson collects what
Snowflake's documentation says, so you can answer accurately rather than from memory.

## Encrypted by default, end to end

From Snowflake's documentation on end-to-end encryption:

- **All customer data at rest is encrypted**, and data is encrypted **in transit** (TLS) to and from the Snowflake service.
  There's nothing to turn on.
- Data loaded into tables is stored in Snowflake's file format, encrypted. When it's processed, it's decrypted for the
  operation and re-encrypted afterwards.
- Files in **internal stages** are encrypted by Snowflake automatically, on upload by the client.
- Files in **external stages** (your own bucket) are your storage, encrypted as your bucket is configured. Snowflake
  recommends **client-side encryption** for them, and supports it for loading and unloading with a stage-level
  `MASTER_KEY` (a 128- or 256-bit AES key, Base64-encoded), or with the cloud provider's server-side encryption options:

```sql
CREATE STAGE learn_db.raw.s3_encrypted
  URL = 's3://my-company-landing/encrypted/'
  STORAGE_INTEGRATION = s3_landing_int
  ENCRYPTION = (TYPE = 'AWS_SSE_KMS' KMS_KEY_ID = 'arn:aws:kms:eu-west-1:123456789012:key/1a2b3c4d-0000-0000-0000-000000000000');
```

- Results **unloaded** to a stage are encrypted per the stage's settings.

## The key hierarchy

Snowflake's documentation describes a **hierarchical key model** with four levels, each encrypting ("wrapping") the
keys below it:

```text
root key            (in a cloud-provider hardware security module, HSM)
 └─ account master key    (one hierarchy per customer account)
     └─ table master key  (per table)
         └─ file keys     (per micro-partition file)
```

Two consequences matter. Every account has its own hierarchy, so one customer's keys never protect another's data. And
each key protects only a small amount of data, which limits the impact of any single key and makes rotation practical.

## Rotation vs rekeying

These are different operations, and security questionnaires often confuse them:

| | Key rotation | Periodic rekeying |
|---|---|---|
| What happens | keys older than **30 days** are retired, and new keys are created for new data | data protected by a retired key older than **one year** is **re-encrypted** with a new key |
| Old key | kept, used only to **decrypt** existing data, destroyed when no longer needed | destroyed once its data has been re-encrypted |
| Edition | all | **Enterprise and higher**, enabled by `ACCOUNTADMIN` |
| Cost to you | none | Snowflake documents a possible additional storage charge during the transition |

```sql
ALTER ACCOUNT SET PERIODIC_DATA_REKEYING = TRUE;     -- Enterprise+, ACCOUNTADMIN
```

Rotation is cheap because creating a new key affects only data written afterwards. Rekeying is a real rewrite of stored
data, which is why it runs on a yearly cycle, and why a policy that demands "all data re-encrypted under a fresh key
regularly" is a rekeying requirement, not a rotation one.

## Tri-Secret Secure: your key in the chain

On **Business Critical** Edition and above, **Tri-Secret Secure** combines a Snowflake-maintained key with a
**customer-managed key** held in your cloud provider's key management service (AWS KMS, Azure Key Vault or Google Cloud KMS)
into a composite master key that protects your data. The practical effect: if you **revoke** Snowflake's access to your
key, Snowflake can no longer decrypt your data. That's a strong control for regulated data, and also an availability risk.
A mistaken key-policy change or a deleted key makes your data unreadable, by you as well. Enabling it is a coordinated
process involving Snowflake Support and your key-management team. It wasn't run for this course, so follow the current
documentation step by step.

## Compliance

Snowflake publishes its certifications and attestations (such as SOC reports, ISO certifications and PCI DSS) and supports
regulated workloads (such as HIPAA) on qualifying editions and regions. The current list and its conditions live on
Snowflake's Trust Center and compliance pages, and they change, so this course doesn't reproduce it. What matters for
practitioners is the **shared responsibility** split:

| Snowflake is responsible for | You are responsible for |
|---|---|
| encryption at rest and in transit, key hierarchy, rotation | choosing the edition and region your obligations require |
| physical and infrastructure security, service availability | users, roles, MFA and authentication policies (lessons 01–02) |
| providing audit data (access history, login history) | network policies and private connectivity (lesson 03) |
| | classifying and masking sensitive data (lessons 04–05) |
| | reviewing and retaining audit logs (next lesson) |
| | external stage buckets and their encryption |

An auditor asking "is personal data protected?" will want evidence from your side of that table: who can read the
`customers` table, who did read it last month, and how you know.

## How It Actually Works

Each micro-partition file is encrypted with its own **file key**. The file key is stored only in encrypted form, wrapped by
the table's master key, which is wrapped by the account master key, which is wrapped by the root key in the HSM. To read a
file, the service unwraps the chain from the top, which requires the HSM. That's why keys never leave hardware protection in
plain form at the root. Rotating a key means new data gets wrapped under a new key, while old wrapped keys remain readable by
their retired parent until they're no longer needed. Rekeying re-encrypts the affected data files themselves under fresh keys,
so it's proportional to data volume.

With Tri-Secret Secure, the account-level key material depends on both Snowflake's key and the key in your KMS. Snowflake must
call your KMS to obtain it, so revoking that permission breaks the chain at the account level, and every file below becomes
undecryptable.

## Common mistakes

- **Unencrypted external stage buckets** because "Snowflake encrypts everything". Your bucket is your responsibility.
- **Confusing rotation with rekeying** in policy documents and questionnaires.
- **Promising customer-managed keys on Standard or Enterprise.** Tri-Secret Secure requires Business Critical or higher.
- **Tri-Secret Secure without key-lifecycle discipline.** A deleted or mis-scoped key is a self-inflicted outage.
- **Copying a compliance list from a blog post.** Check Snowflake's current Trust Center pages for your edition and region.

## Exercise

1. Write the one-page answer to a customer's security questionnaire section on encryption for your account: at rest, in
   transit, key ownership, rotation, rekeying, and external stages. Cite the documentation pages you used.
2. Check whether periodic rekeying is enabled in your account (`SHOW PARAMETERS LIKE 'PERIODIC_DATA_REKEYING' IN ACCOUNT`),
   and decide whether it should be.
3. Configure an external stage on a bucket with server-side KMS encryption, load the course's order files through it, and
   confirm that loading still works when the bucket requires encryption.
4. List the controls from lessons 01–07 that an auditor would ask you to evidence for a table containing personal data,
   and the query or screen you would show for each.
