# AWS KMS — A Complete, Practical Learning Guide

> A from-scratch, hands-on walkthrough of AWS Key Management Service (KMS) — built for learning, referencing, and showcasing real understanding of encryption on AWS.

**Category:** Security, Identity & Compliance
**Status:** ✅ Complete
**Last updated:** 2026-08-03

---

## 📚 What's in this repo

| File | What it covers |
|---|---|
| [`README.md`](./README.md) | You are here — concepts, architecture, and the "why" behind everything |
| [`commands-cheatsheet.md`](./commands-cheatsheet.md) | Every AWS CLI command you'll actually use, organized by task |
| [`hands-on-labs.md`](./hands-on-labs.md) | 7 guided labs — from creating your first key to a full encrypted application |
| [`troubleshooting.md`](./troubleshooting.md) | Real error messages and exactly how to fix them |

**Suggested order:** Read this README top to bottom once → do the labs in `hands-on-labs.md` → keep `commands-cheatsheet.md` open as a reference → bookmark `troubleshooting.md` for when things go sideways (they will, and that's fine).

---

## Table of Contents

1. [Why This Exists](#1-why-this-exists)
2. [Prerequisites](#2-prerequisites)
3. [High-Level Architecture & Service Flow](#3-high-level-architecture--service-flow)
4. [Core Concepts & Deep-Dive](#4-core-concepts--deep-dive)
5. [Step-by-Step Configuration & Implementation Guide](#5-step-by-step-configuration--implementation-guide)
6. [How to Use & Where to Use (Target Use Cases)](#6-how-to-use--where-to-use-target-use-cases)
7. [Limits, Quotas & Pricing](#7-limits-quotas--pricing)
8. [Security Best Practices](#8-security-best-practices)
9. [Glossary](#9-glossary)
10. [Further Reading](#10-further-reading)

---

## 1. Why This Exists

Almost every AWS workload eventually needs to answer the question: *"How do I encrypt this, and who's allowed to decrypt it?"*

AWS KMS is the service that answers that question for nearly the entire AWS ecosystem — S3 buckets, EBS volumes, RDS databases, Secrets Manager, Lambda environment variables, DynamoDB tables, and your own applications all lean on it. It's a small service conceptually, but it sits at the center of security architecture on AWS, which is exactly why it's worth understanding properly instead of just clicking "Enable encryption" and moving on.

This guide treats KMS as something to actually *understand* — not just a checkbox. Each concept is explained in plain language first, then backed by a hands-on lab so you build muscle memory, not just familiarity.

---

## 2. Prerequisites

Before starting the labs, make sure you have:

- **An AWS account** — a free tier account works fine; KMS has a small monthly cost per key (~$1), so labs use a handful of keys and clean up after themselves.
- **AWS CLI v2 installed** — verify with:
  ```bash
  aws --version
  ```
- **IAM credentials configured locally** with at least `kms:*`, `iam:*` (to create roles for later labs), and `sts:GetCallerIdentity` permissions. Use a sandbox/non-production account if possible.
  ```bash
  aws configure
  aws sts get-caller-identity
  ```
- **Basic command-line comfort** — you should be okay running commands, editing JSON files, and reading error output.
- **(Optional but useful)** `jq` installed for pretty-printing JSON CLI output:
  ```bash
  jq --version
  ```
- **A general idea of IAM** — you don't need to be an IAM expert, but knowing what a policy, role, and principal are will make Section 4 click faster.

No prior cryptography background is assumed. Every crypto term used here is explained the first time it appears.

---

## 3. High-Level Architecture & Service Flow

### 3.1 The one-sentence version

KMS creates and safeguards **keys** inside AWS-managed hardware, and instead of using those keys to encrypt your actual data directly, it uses them to encrypt much smaller, short-lived **data keys** — your data never has to leave your application to get encrypted.

### 3.2 Why not just encrypt everything with the KMS key directly?

Two reasons:

1. **Performance** — every call to KMS is a network round trip. If you encrypted every row of a database or every object in S3 directly through KMS, you'd hit rate limits and add latency to everything.
2. **A hard limit** — KMS's direct `Encrypt`/`Decrypt` API caps payloads at **4 KB**. It was never designed to encrypt bulk data directly.

The solution is a pattern called **envelope encryption**, and it's the single most important concept in this entire guide.

### 3.3 Envelope Encryption — Visual Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                         ENCRYPTING DATA                                │
└──────────────────────────────────────────────────────────────────────┘

  Your App                          AWS KMS
  ────────                          ───────
     │                                 │
     │  1. GenerateDataKey(keyId)      │
     │ ───────────────────────────────>│
     │                                 │  KMS key (lives only in KMS,
     │                                 │  never exported) generates a
     │                                 │  new AES-256 data key
     │                                 │
     │  2. Returns TWO things:         │
     │     - Plaintext data key        │
     │     - Same key, encrypted       │
     │ <───────────────────────────────│
     │                                 │
     │  3. Encrypt your file/data      │
     │     locally using the           │
     │     PLAINTEXT data key          │
     │     (fast — no network call)    │
     │                                 │
     │  4. Throw away the plaintext    │
     │     data key immediately        │
     │                                 │
     │  5. Store: [ciphertext] +       │
     │     [encrypted data key]        │
     │     together                    │


┌──────────────────────────────────────────────────────────────────────┐
│                         DECRYPTING DATA                                │
└──────────────────────────────────────────────────────────────────────┘

  Your App                          AWS KMS
  ────────                          ───────
     │                                 │
     │  1. Read the stored             │
     │     [encrypted data key]        │
     │                                 │
     │  2. Decrypt(encryptedDataKey)   │
     │ ───────────────────────────────>│
     │                                 │  KMS key decrypts it —
     │                                 │  only works if caller is
     │                                 │  authorized (key policy + IAM)
     │                                 │
     │  3. Returns plaintext data key  │
     │ <───────────────────────────────│
     │                                 │
     │  4. Use plaintext data key to   │
     │     decrypt the actual data     │
     │     locally                     │
     │                                 │
     │  5. Discard plaintext data key  │
```

The KMS key (called a "CMK" historically — Customer Master Key — now just "KMS key") never leaves AWS. It only ever wraps and unwraps small data keys. This is the pattern every AWS service integration uses under the hood.

### 3.4 Service Integration Flow (e.g., S3 with SSE-KMS)

```
You upload a file to S3 (encryption enabled with a KMS key)
        │
        ▼
S3 asks KMS: "Generate a data key using this KMS key,
              and use this grant you gave me earlier"
        │
        ▼
KMS returns a plaintext + encrypted data key to S3
        │
        ▼
S3 encrypts your object with the plaintext data key,
stores the encrypted data key as object metadata,
discards the plaintext data key
        │
        ▼
Later, on download: S3 sends the encrypted data key back to
KMS, gets the plaintext key back, decrypts your object,
streams it to you
```

You never see any of this happen — it's the exact same envelope encryption pattern from 3.3, just automated by the AWS service on your behalf using a **grant** (explained in Section 4).

### 3.5 Building Blocks at a Glance

| Component | What it is |
|---|---|
| **KMS key** | The logical key resource. Backed by real cryptographic key material stored in HSMs. |
| **Key policy** | A resource-based policy attached to the key — the root source of access control. |
| **IAM policy** | Identity-based policy (on users/roles) — works *together with* the key policy, not instead of it. |
| **Grant** | A temporary, narrowly-scoped permission, usually created by AWS services on your behalf. |
| **Alias** | A friendly name (`alias/my-app-key`) pointing at a key ID, so your code never hardcodes key IDs. |
| **Key material** | The actual secret cryptographic bytes. Generated by AWS, imported by you, or held in a custom HSM. |
| **CloudTrail** | Logs every single KMS API call — who called what, when, using which key. |

---

## 4. Core Concepts & Deep-Dive

### 4.1 Key Types (by usage)

| Key usage | What it does | Typical use case |
|---|---|---|
| **Symmetric encryption key** (default) | One key, used to both encrypt and decrypt | 95% of use cases — S3, EBS, RDS, app data, envelope encryption |
| **Asymmetric encryption key pair** (RSA) | Public key encrypts, private key decrypts | When the encrypting party shouldn't be able to decrypt (e.g., a third party encrypting data *for* you) |
| **Asymmetric signing key pair** (RSA or ECC) | Private key signs, public key verifies | Digital signatures, code signing, JWT signing where verification must happen outside AWS |
| **HMAC key** | Symmetric key used to generate/verify a message authentication code | API request signing, verifying message integrity between services |

**Rule of thumb:** if you're not sure which to pick, you want a **symmetric encryption key**. It's the default, the cheapest, the fastest, and what nearly every AWS service integration expects.

### 4.2 Key Material Origin

| Origin | Meaning |
|---|---|
| `AWS_KMS` (default) | AWS generates and manages the key material entirely. Simplest option. |
| `EXTERNAL` | You generate the key material yourself and import it ("Bring Your Own Key" / BYOK). You control its lifecycle and can delete the material to instantly "shred" the key. |
| `AWS_CLOUDHSM` | The key lives in a **custom key store** backed by your own dedicated CloudHSM cluster — single-tenant hardware, but still speaks the KMS API. |
| `EXTERNAL_KEY_STORE` | Key material lives entirely outside AWS, in your own on-premises or third-party key manager, via the External Key Store (XKS) feature. |

### 4.3 Types of KMS Keys by Ownership

This trips up almost everyone early on — there are **three** categories of keys, and they behave very differently:

| Type | Who manages it | Cost | You can... |
|---|---|---|---|
| **AWS owned keys** | Fully AWS — you never see them in your account | Free | Nothing — completely invisible, used internally by some AWS services for default encryption |
| **AWS managed keys** | AWS creates & rotates them, but they appear in your account as `aws/service-name` (e.g., `aws/s3`) | Free (no monthly key fee) | View key policy, view rotation status — but not edit the policy |
| **Customer managed keys (CMKs)** | You create, own, configure, and pay for | ~$1/month + usage | Full control — key policy, rotation settings, aliases, grants, deletion |

**In practice:** use **customer managed keys** whenever you need control over who can use the key, want custom rotation policies, need to grant cross-account access, or need audit-level clarity on exactly who's using what. Use AWS managed keys for quick, low-stakes default encryption where you don't need that control.

### 4.4 Key Policies vs. IAM Policies (the #1 source of confusion)

This is the concept that trips up the most people, so let's slow down.

- **Every KMS key must have a key policy.** No exceptions. It is a resource-based policy, similar in shape to an S3 bucket policy.
- **The key policy is the ultimate gatekeeper.** Even if an IAM user has `kms:Decrypt` with `"Resource": "*"` in their IAM policy, they **cannot** use the key unless the key policy also allows it.
- **The default key policy** (created automatically when you make a key via the console) includes a statement granting the account root full access, and — critically — delegates permission management to IAM:

  ```json
  {
    "Sid": "Enable IAM User Permissions",
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::111122223333:root" },
    "Action": "kms:*",
    "Resource": "*"
  }
  ```

  This single statement is what allows you to manage everything else through ordinary IAM policies afterward. **If you remove this statement**, you must manage 100% of access through the key policy directly — IAM policies alone won't do anything.

**Mental model:** think of the key policy as the *master switch*, and IAM as the *day-to-day dial*. Both have to say "yes" for access to be granted.

### 4.5 Grants

Grants are a way to delegate specific, narrow permissions on a key — programmatically, without editing the key policy JSON.

- Created via `CreateGrant`, referencing a principal, a key, and a specific set of allowed operations.
- Can include **constraints** (e.g., only allow `GenerateDataKey` if a specific `EncryptionContext` is present).
- This is exactly how AWS services get scoped, temporary access to your keys — e.g., when you enable SSE-KMS on an S3 bucket, S3 uses a grant, not a change to your key policy.
- Grants can be retired (`RetireGrant`) or revoked (`RevokeGrant`) — useful for cleaning up access after decommissioning something.

### 4.6 Encryption Context

An optional set of key-value pairs (non-secret) passed alongside `Encrypt`, `Decrypt`, and `GenerateDataKey` calls.

- Used as **Additional Authenticated Data (AAD)** — it doesn't get encrypted, but it does get cryptographically bound to the ciphertext. If you decrypt with a different (or missing) encryption context than what was used to encrypt, decryption fails.
- Great for binding ciphertext to metadata — e.g., `{"customer-id": "12345"}` — so a data key generated for one customer's data can't accidentally (or maliciously) be used to decrypt another's.
- You can also require specific encryption context values via key policy/IAM policy conditions for stronger authorization.
- Shows up in CloudTrail logs, so it's also great for auditing *what* was being encrypted/decrypted, not just *that* something was.

### 4.7 Aliases

- Human-friendly pointers to key IDs: `alias/prod-db-key` instead of `1234abcd-12ab-34cd-56ef-1234567890ab`.
- Let you **swap the underlying key** an alias points to without changing application code — useful for key rotation strategies or disaster recovery.
- Every alias is scoped to a single account and region.
- Note: deleting an alias does **not** delete the key it points to, and deleting a key doesn't clean up its aliases automatically — a very common source of orphaned aliases.

### 4.8 Automatic Key Rotation

- Available for **AWS-generated symmetric encryption keys** only.
- When enabled, AWS periodically generates new backing key material, but the key ID and ARN never change — your applications don't need to know rotation happened.
- Old key material is retained internally (not deleted) so **data encrypted before rotation can still be decrypted** — rotation only affects what new encryption operations use going forward.
- Rotation period is configurable (typically 90–2560 days), with an AWS default if unspecified.
- **Not supported** for: asymmetric keys, HMAC keys, or keys with imported (`EXTERNAL`) key material. For those, "rotation" means manually creating a new key (or importing new material) and updating your aliases/references.

### 4.9 Multi-Region Keys

- A "primary" key can be replicated into "replica" keys in other regions.
- Replicas share the **same key material and key ID** as the primary — but are still managed as independent resources (independent key policies, independent enable/disable state, independent deletion).
- This lets you encrypt data in one region and decrypt it in another **without** re-encrypting, which is a common requirement for cross-region disaster recovery, global applications, or replicated databases (e.g., DynamoDB Global Tables, Aurora Global Database).
- Important distinction: a *normal* (single-region) KMS key can **only** be used within the region it was created in. If you try to use a `us-east-1` key from `eu-west-1`, it will fail — this is one of the most common early mistakes.

### 4.10 Custom Key Store

- Lets a KMS key's material live in a **dedicated CloudHSM cluster** you control, rather than AWS's shared multi-tenant HSM fleet.
- Still exposes the standard KMS API — your application code doesn't change.
- Chosen when compliance requirements mandate single-tenant HSM control (some strict regulatory environments), while still wanting the operational convenience of the KMS API.
- Comes with real trade-offs: you now manage CloudHSM cluster availability, and it costs meaningfully more (hourly CloudHSM instance charges on top of KMS).

### 4.11 Auditing with CloudTrail

- Every KMS API call — `Encrypt`, `Decrypt`, `GenerateDataKey`, `CreateGrant`, key policy changes, everything — is logged to CloudTrail by default in most accounts.
- This is what makes KMS genuinely auditable: you can answer "who decrypted this specific piece of data, and when?" after the fact.
- Combine with the encryption context (4.6) for even richer audit trails — you can see not just *that* a decrypt happened, but *what* it was contextually for.

---

## 5. Step-by-Step Configuration & Implementation Guide

This is the condensed version — full guided labs with explanations live in [`hands-on-labs.md`](./hands-on-labs.md).

**Step 1 — Create a symmetric customer managed key**
```bash
aws kms create-key \
  --description "App-tier data encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT
```

**Step 2 — Give it a friendly alias**
```bash
aws kms create-alias \
  --alias-name alias/app-tier-key \
  --target-key-id <key-id-from-step-1>
```

**Step 3 — Review/adjust the key policy** so it delegates to IAM (see 4.4), then manage day-to-day access via IAM policies scoped to this key's ARN.

**Step 4 — Enable automatic rotation** (if it's a long-lived symmetric key)
```bash
aws kms enable-key-rotation --key-id alias/app-tier-key
```

**Step 5 — Use envelope encryption in your application**
```bash
# Generate a data key
aws kms generate-data-key \
  --key-id alias/app-tier-key \
  --key-spec AES_256

# ...encrypt your data locally with the plaintext key, discard it...

# Later, decrypt the stored encrypted data key
aws kms decrypt --ciphertext-blob fileb://encrypted-data-key.bin
```

**Step 6 — Confirm auditing is working** by checking CloudTrail for the calls you just made.

---

## 6. How to Use & Where to Use (Target Use Cases)

| Criteria | Choose AWS KMS | Choose an Alternative |
|---|---|---|
| Native encryption for S3, EBS, RDS, Lambda, DynamoDB, etc. with minimal setup | ✅ KMS | — |
| Managing app secrets (DB passwords, API tokens) with rotation *workflows* built in | — | **Secrets Manager** (which itself is backed by KMS under the hood) |
| Simple encrypted config values, low traffic, cost-sensitive | — | **SSM Parameter Store** (SecureString, KMS-backed) |
| Dedicated, single-tenant HSM required by strict compliance mandates | — | **CloudHSM** (or KMS custom key store as a hybrid) |
| Client-side, high-throughput bulk encryption without one API call per object | ✅ KMS via envelope encryption / AWS Encryption SDK | — |
| Same key usable to encrypt in one region, decrypt in another | ✅ KMS Multi-Region keys | — |
| Fully control your own key material lifecycle, generated outside AWS | ✅ KMS with BYOK (`EXTERNAL` origin) | Third-party/on-prem key manager via External Key Store |
| Hybrid or on-prem workloads needing a key manager entirely outside AWS's control plane | — | External Key Store (XKS) or a fully third-party KMS |
| Digital signatures verifiable outside AWS (e.g., by a partner system) | ✅ KMS asymmetric signing keys | — |

---

## 7. Limits, Quotas & Pricing

> Figures below are directional — always check the [AWS KMS pricing page](https://aws.amazon.com/kms/pricing/) and [service quotas](https://docs.aws.amazon.com/general/latest/gr/kms.html) for current numbers before relying on them for capacity planning.

**Quotas:**
- Direct `Encrypt`/`Decrypt` payload limit: **4 KB** (this is *the* reason envelope encryption exists).
- Key policy document size limit: **32 KB**.
- Grants per key: soft-limited in the low thousands, raisable via Service Quotas.
- Cryptographic operation requests are rate-limited **per account, per region, per API action** — `GenerateDataKey`/`Decrypt` have much higher default throughput than management actions like `CreateKey`.

**Pricing (illustrative):**
- Customer managed keys: **~$1/month per key** (prorated), plus a small monthly fee per Multi-Region replica.
- API requests: billed per 10,000 requests above a monthly free tier; asymmetric/HMAC operations cost more per-request than symmetric ones.
- AWS managed keys: **no monthly key fee**, only usage-based request charges.
- Custom key store (CloudHSM-backed): billed separately at CloudHSM's hourly cluster rate, on top of standard KMS request pricing.

---

## 8. Security Best Practices

- **Never use `"Resource": "*"` for `kms:Decrypt`/`kms:GenerateDataKey`** in production IAM policies — scope to specific key ARNs.
- **Use separate keys per application/data-classification**, not one giant shared key — it limits blast radius and keeps IAM/audit trails clean.
- **Always require encryption context** where the data's sensitivity warrants it — it's a nearly-free extra authorization layer.
- **Enable automatic rotation** on long-lived symmetric keys unless you have a specific reason not to.
- **Audit grants periodically** (`ListGrants`) — they're easy to create and easy to forget about.
- **Use a VPC interface endpoint for KMS** (`com.amazonaws.<region>.kms`) if your workloads run in private subnets without NAT, to keep KMS traffic off the public internet.
- **Tag your keys** — cost allocation and ownership tracking get painful fast without them.
- **Treat key deletion as a last resort** — disable a key first and observe for a while before scheduling deletion, since deletion (after the waiting period) is irreversible.

---

## 9. Glossary

| Term | Meaning |
|---|---|
| **KMS key** | The logical key resource (previously called CMK — Customer Master Key) |
| **CMK** | Legacy term for KMS key; still seen in older docs/APIs |
| **Envelope encryption** | Pattern of encrypting data with a local data key, then encrypting that data key with a KMS key |
| **Data key** | A short-lived symmetric key generated by KMS, used to encrypt actual data locally |
| **Key policy** | Resource-based policy attached to a KMS key; the root access control layer |
| **Grant** | Temporary, programmatic delegation of specific permissions on a key |
| **Encryption context** | Non-secret key-value metadata cryptographically bound to ciphertext (AAD) |
| **BYOK** | Bring Your Own Key — importing your own key material into KMS |
| **HSM** | Hardware Security Module — tamper-resistant hardware that stores/uses key material |
| **Custom key store** | A KMS key backed by your own dedicated CloudHSM cluster |
| **Multi-Region key** | A KMS key replicated across regions, sharing key material and key ID |
| **XKS** | External Key Store — key material lives entirely outside AWS |

---

## 10. Further Reading

- [AWS KMS Developer Guide](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html)
- [AWS KMS Cryptographic Details Whitepaper](https://docs.aws.amazon.com/kms/latest/cryptographic-details/)
- [AWS Encryption SDK](https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/introduction.html)
- Next: work through [`hands-on-labs.md`](./hands-on-labs.md) →

---

*Related docs in this repo: [`commands-cheatsheet.md`](./commands-cheatsheet.md) · [`hands-on-labs.md`](./hands-on-labs.md) · [`troubleshooting.md`](./troubleshooting.md)*
