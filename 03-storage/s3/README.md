# AWS S3 Mastery — End-to-End Practical Learning Guide

A complete, practitioner-level walkthrough of Amazon S3: architecture, security, cost optimization, performance, and the newest AI/analytics extensions to the platform. Built as a study reference and a portfolio artifact — every concept below is paired with the commands, policies, and gotchas you need to actually operate S3 in production.

> 📌 This guide reflects S3 as of mid-2026, including recently released capabilities (S3 Tables, S3 Vectors, S3 Files, Account Regional Namespaces, and Object Annotations). Always cross-check pricing and regional availability against the official [S3 documentation](https://docs.aws.amazon.com/s3/) before relying on specifics in production.

## 📁 Repo Structure

| File | What's in it |
|---|---|
| **README.md** | This file — the full conceptual deep-dive, organized by topic |
| **commands-cheatsheet.md** | Every AWS CLI command, JSON policy, and Terraform snippet, grouped by topic for quick copy-paste |
| **hands-on-labs.md** | 15 step-by-step labs you can run in a sandbox AWS account to prove out each concept |
| **troubleshooting.md** | Real-world error messages, root causes, and fixes — organized as a lookup table |

## 🗺️ Table of Contents

1. [S3 Core Architecture](#1-s3-core-architecture)
2. [Bucket Policies & the PARC Model](#2-bucket-policies--the-parc-model)
3. [Cross-Account Bucket Access](#3-cross-account-bucket-access)
4. [Presigned URLs](#4-presigned-urls)
5. [Requester Pays](#5-requester-pays)
6. [Versioning, Delete Markers & MFA Delete](#6-versioning-delete-markers--mfa-delete)
7. [Multipart Upload](#7-multipart-upload)
8. [S3 Transfer Acceleration](#8-s3-transfer-acceleration)
9. [Encryption Fundamentals](#9-encryption-fundamentals)
10. [AWS KMS & Envelope Encryption](#10-aws-kms--envelope-encryption)
11. [Server-Side vs Client-Side Encryption](#11-server-side-vs-client-side-encryption)
12. [S3 Storage Classes](#12-s3-storage-classes)
13. [Lifecycle Configuration (Transitions & Expirations)](#13-lifecycle-configuration-transitions--expirations)
14. [S3 Replication (CRR & SRR)](#14-s3-replication-crr--srr)
15. [Centralized Log Aggregation on S3](#15-centralized-log-aggregation-on-s3)
16. [S3 Select & Glacier Select](#16-s3-select--glacier-select)
17. [S3 Object Lock (WORM Compliance)](#17-s3-object-lock-worm-compliance)
18. [Advanced & 2025–2026 Capabilities](#18-advanced--20252026-capabilities)
19. [Master Decision Cheat Sheet](#19-master-decision-cheat-sheet)

## ✅ Learning Map

- [x] Core hierarchy — buckets, keys, prefixes, objects
- [x] Security — IAM vs bucket policies, cross-account access, presigned URLs
- [x] Cost controls — requester pays, storage classes, lifecycle rules
- [x] Durability & recovery — versioning, replication, object lock
- [x] Cryptography — symmetric vs asymmetric, SSE vs CSE, KMS envelope encryption
- [x] Performance — multipart upload, transfer acceleration, prefix scaling
- [x] Query-in-place — S3 Select, Glacier Select
- [x] Operational patterns — log lakes, WORM compliance, batch operations
- [x] Frontier features — S3 Tables, S3 Vectors, S3 Files, Account Regional Namespaces, Object Annotations

---

## 1. S3 Core Architecture

S3 is not a filesystem — it's a flat, highly scalable key-value store. There is no real directory tree; "folders" are just a UI convention built from string prefixes.

```
[ AWS Global Infrastructure ]
       │
       ├───> [ AWS Region: e.g., ap-south-1 ]
                   │
                   └───> [ S3 Bucket: globally-unique-name ]
                               │
                               ├───> Prefix: `images/`
                               │        └───> Object (Key): `images/logo.png`
                               │
                               └───> Prefix: `logs/2026/`
                                        └───> Object (Key): `logs/2026/app.log`
```

**Key terms:**
- **Bucket** — the root container, created in a specific Region. The name is globally unique, but the data never leaves the chosen Region unless you explicitly configure replication.
- **Key** — the full unique string path identifying an object (e.g. `content/ui/index.html`).
- **Prefix** — the string before the last `/`. S3 uses prefixes for performance partitioning (up to 3,500 PUTs / 5,500 GETs per second, per prefix) and for lifecycle/replication filters.

### Anatomy of an object

| Component | Contents | Boundary |
|---|---|---|
| Data Payload | Raw byte stream | 0 bytes – 5 TB |
| System Metadata | Content-Type, Last-Modified, ETag, Version ID | Auto-managed by AWS |
| User Metadata | Custom `x-amz-meta-*` headers | Application-defined |

### The request evaluation flow (memorize this — it's the #1 debugging tool)

1. **Authentication check** — is the request signed? If anonymous and public access isn't enabled → rejected.
2. **Block Public Access guardrail** — if traffic comes from the internet and Block Public Access is on → `403 Forbidden`.
3. **Explicit Deny evaluation** — any `Deny` statement in IAM, bucket policy, or KMS key policy that matches → instant `403` (**Deny always wins**, regardless of any Allow elsewhere).
4. **Explicit Allow verification** — is there an `Allow` in either the IAM policy or the bucket policy? Yes → `200 OK`. No → `403 Forbidden` (default-deny).

---

## 2. Bucket Policies & the PARC Model

A **Bucket Policy** is a resource-based JSON policy attached directly to the bucket. Every statement follows the **PARC model**:

- **P**rincipal — who is asking? (IAM user/role, external account, or `*` for public)
- **A**ction — what API call? (`s3:GetObject`, `s3:PutObject`, `s3:ListBucket`, …)
- **R**esource — bucket-level actions (like `ListBucket`) target the bare bucket ARN (`arn:aws:s3:::my-bucket`); object-level actions (like `GetObject`) need the wildcard suffix (`arn:aws:s3:::my-bucket/*`)
- **C**ondition — optional fine-grained filter (IP, encryption, MFA, org ID, etc.)

### Production-hardened template

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowSpecificRoleReadOnly",
            "Effect": "Allow",
            "Principal": { "AWS": "arn:aws:iam::111122223333:role/application-read-role" },
            "Action": ["s3:GetObject", "s3:ListBucket"],
            "Resource": [
                "arn:aws:s3:::my-production-data-bucket",
                "arn:aws:s3:::my-production-data-bucket/*"
            ]
        },
        {
            "Sid": "DenyUnencryptedConnections",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::my-production-data-bucket",
                "arn:aws:s3:::my-production-data-bucket/*"
            ],
            "Condition": { "Bool": { "aws:SecureTransport": "false" } }
        }
    ]
}
```

### Best practices

- **Root lockout rule**: a badly configured `Deny` + `Principal: "*"` can lock everyone out. Only the root account can recover from this — be careful with wildcard denies.
- **Bucket vs object ARN mismatch** is the #1 cause of unexpected `Access Denied`. `s3:PutObject` needs the `/*` suffix; `s3:ListBucket` doesn't.
- **IAM policy vs bucket policy**: use IAM when you want to control what *one identity* can do across *many resources*; use a bucket policy when you want centralized perimeter rules for *one resource* regardless of who's asking.

---

## 3. Cross-Account Bucket Access

When a principal in **Account B** needs to read/write an S3 bucket owned by **Account A**, both sides must explicitly agree:

> **The core rule**: Same-account access needs an `Allow` in *either* the IAM policy or the bucket policy. Cross-account access needs an explicit `Allow` in **both** — the IAM policy in Account B (letting the principal go out) *and* the bucket policy in Account A (letting Account B in).

### Method 1 — Direct resource-based access (simplest)

```
[ Account B: IAM User/Role ] ══ Direct S3 API Call ══> [ Account A: S3 Bucket ]
     IAM Policy allows the call                    Bucket Policy trusts Account B
```

Use when an app in Account B calls the bucket in Account A directly with its own credentials. Requires a bucket policy in A trusting the Account B role, **and** an IAM policy in B allowing the call.

### Method 2 — STS AssumeRole (best for scaling & governance)

```
[Account B: User/App] --sts:AssumeRole--> [Account A: CrossAccountRole] --local access--> [Account A: Bucket]
```

Instead of updating the bucket policy every time a new consumer appears, Account B assumes a role that already lives inside Account A. Account B only needs `sts:AssumeRole` permission — everything else is governed centrally in Account A.

### The three production gotchas

1. **Object Ownership trap** — if Account B uploads an object into Account A's bucket, by default Account B still *owns* that object, and Account A can't read or modify it. **Fix:** set S3 Object Ownership to **Bucket Owner Enforced**, which strips ACLs and forces ownership onto the bucket owner.
2. **KMS pitfall** — if the bucket uses a Customer Managed Key (CMK), Account B gets `Access Denied` even with perfect S3 policies. **Fix:** explicitly grant `kms:Decrypt` and `kms:GenerateDataKey` to Account B's principal in the KMS Key Policy.
3. **Org-wide access at scale** — instead of listing account IDs one by one, use `aws:PrincipalOrgId` to trust an entire AWS Organization:
   ```json
   "Condition": { "StringEquals": { "aws:PrincipalOrgId": "o-xxxxxxxxxx" } }
   ```

---

## 4. Presigned URLs

A presigned URL grants **temporary, credential-free access** to a private object. It's cryptographically signed using the *creator's* IAM credentials — no separate API call to AWS is required to generate one; the signing happens entirely client-side in the SDK.

### The authentication chain

- **Permission mapping**: the URL is a proxy for the identity that signed it. If that identity loses access, the URL breaks — even if it hasn't expired yet.
- **Token expiration**: URLs signed with temporary credentials (EC2/Lambda/EKS roles) expire the moment the underlying session expires, regardless of the `ExpiresIn` you set.

### Generating one

```bash
aws s3 presign s3://my-secure-bucket/architecture-diagram.pdf --expires-in 900
```

```python
import boto3
s3 = boto3.client('s3')
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-secure-bucket', 'Key': 'architecture-diagram.pdf'},
    ExpiresIn=3600
)
```

Presigned URLs also work for **PUT** — letting a frontend upload directly to S3 without the payload ever touching your application servers.

### Boundaries

| Signed by | Max lifetime |
|---|---|
| IAM User (long-lived access keys) | Up to 7 days |
| IAM Role (Lambda / EC2 / ECS) | Bound to the role's max session duration (1–12 hrs) |

**Revocation:** you cannot selectively revoke a presigned URL once shared. To kill it immediately: delete the object, rotate/change its encryption key, or revoke the signing principal's permissions.

---

## 5. Requester Pays

By default, the **bucket owner pays for everything** — storage, requests, and egress. If you're sharing large datasets publicly, egress can spiral. **Requester Pays** shifts data-transfer and request costs to whoever downloads the data.

### Split of responsibility

| Cost | Who pays |
|---|---|
| Storage | Bucket owner (always) |
| Data transfer (egress) | Requester |
| API requests (GET/LIST) | Requester |

### Two hard prerequisites

1. **The requester must authenticate** — anonymous/public requests are rejected outright, since AWS needs to know who to bill.
2. **The requester must explicitly accept charges** — omitting the accept-charges flag returns `403 Forbidden`.

### Configuration

```bash
# Owner: enable it
aws s3api put-bucket-request-payment \
    --bucket your-shared-data-bucket \
    --request-payment-configuration '{"Payer": "Requester"}'

# Requester: must pass the flag on every call
aws s3 cp s3://your-shared-data-bucket/dataset.zip . --request-payer requester
aws s3 ls s3://your-shared-data-bucket/ --request-payer requester
```

SDK equivalent: pass `RequestPayer='requester'` in the call parameters.

### Gotchas

- **No static website hosting** — that requires anonymous public access, which conflicts with the auth requirement.
- **No BitTorrent distribution** while Requester Pays is active.
- **Same-Region access is free** — if the requester's compute is in the same Region as the bucket, transfer cost drops to $0 (only the tiny request fee applies).

---

## 6. Versioning, Delete Markers & MFA Delete

Bucket Versioning keeps multiple variants of an object, protecting against accidental overwrites and deletions.

### The three states

- **Unversioned (default)** — overwrites replace the object entirely.
- **Versioning-Enabled** — every write creates a new version with a unique Version ID.
- **Versioning-Suspended** — you can never fully disable versioning once enabled, only suspend it. New uploads get a `null` version ID; existing versions are untouched.

### Current vs. Noncurrent

Uploading the same key repeatedly doesn't overwrite — it stacks versions. The newest is the **Current Version**; everything older becomes **Noncurrent**.

### Delete Markers

Deleting an object in a versioned bucket does **not** erase data:

1. S3 inserts a zero-byte **Delete Marker** as the new current version.
2. A plain `GET` now returns `404 Not Found` — the object *appears* deleted.
3. To "undelete," remove the Delete Marker; the previous noncurrent version instantly becomes current again.
4. To **actually** purge data, you must pass the specific `VersionId` in the delete call.

### Managing cost: versioning + lifecycle

Unbounded version history on a bucket with frequent overwrites (e.g. streaming logs) causes runaway storage costs. Combine with lifecycle rules:

```json
{
  "Rules": [{
    "ID": "ManageNoncurrentVersions",
    "Status": "Enabled",
    "Filter": {"Prefix": "logs/"},
    "NoncurrentVersionExpiration": {
      "NoncurrentDays": 90,
      "NewerNoncurrentVersions": 2
    }
  }]
}
```
This purges noncurrent versions after 90 days, but always keeps the 2 newest as a rollback safety net.

### MFA Delete

Adds a hardware/virtual MFA requirement for two destructive operations:
- Permanently deleting an object version
- Suspending the bucket's versioning state

MFA Delete can **only** be toggled using the **root account's** credentials via CLI/API — not even IAM Administrators can bypass it.

---

## 7. Multipart Upload

Single-stream `PUT` operations cap out around 5 GB and are fragile over long transfers. Multipart Upload splits a file into chunks, uploads them (in parallel, if you want), and reassembles them server-side.

### Why use it

- **Bypasses the 5 GB single-PUT limit**, up to the 5 TB object max.
- **Parallelism** — multiple chunks stream concurrently over separate connections.
- **Fault tolerance** — a failed chunk means re-uploading just that chunk, not the whole file.
- **Pause/resume** — you can stop mid-transfer and continue later.

### Hard technical boundaries

| Constraint | Value |
|---|---|
| Object size range | 5 MB – 5 TB |
| Minimum part size | 5 MB (last part may be smaller) |
| Maximum part size | 5 GB |
| Maximum parts | 10,000 |

> **Sizing example:** a 100 GB file can't use 5 MB chunks (100GB / 5MB = 20,000 parts > the 10,000 cap). 25–100 MB chunks are the sweet spot for a 100 GB object.

### The 3-phase workflow

1. **Initiation** — `CreateMultipartUpload` → AWS returns a unique `UploadId`.
2. **Uploading chunks** — `UploadPart` with `UploadId` + sequential `PartNumber` (1–10,000) + raw bytes. Each successful part returns an `ETag`; your app must track the `PartNumber → ETag` mapping.
3. **Completion** — `CompleteMultipartUpload` with a sorted array of `PartNumber`/`ETag` pairs. S3 assembles the object in order and cleans up the temp chunks.

### The hidden cost trap: incomplete multipart uploads

If a transfer dies mid-stream, the already-uploaded parts sit in the bucket **indefinitely**, invisible to `aws s3 ls`, but still billed at standard storage rates. Fix it with a lifecycle rule:

```json
{
  "Rules": [{
    "ID": "AbortIncompleteMultipartUploadsAfter7Days",
    "Status": "Enabled",
    "Filter": {},
    "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
  }]
}
```

### Automation

You rarely hand-roll `UploadPart` loops — `aws s3 cp`/`aws s3 sync` and SDK Transfer Managers (Boto3's `S3Transfer`, etc.) auto-detect large files (>8 MB by default) and multipart them transparently.

---

## 8. S3 Transfer Acceleration

Speeds up long-distance uploads/downloads by routing traffic through CloudFront **Edge Locations** and AWS's private backbone instead of the public internet.

```
Without TA:  [ User in India ] ==== Public Internet / Multi-Hop ====> [ Bucket in USA ]

With TA:     [ User in India ] --(short local hop)--> [ AWS Edge Location ] --(private fiber)--> [ Bucket in USA ]
```

### Setup

Enabling TA provisions a distinct endpoint:
- Standard: `my-bucket-name.s3.amazonaws.com`
- Accelerated: `my-bucket-name.s3-accelerate.amazonaws.com`

Your application must target the `.s3-accelerate` domain to benefit.

### When to use it

✅ Geographically dispersed users uploading to a centralized bucket
✅ Large object batches (video, ML datasets) crossing continents
✅ Combined with multipart upload for maximum throughput

❌ Same-region traffic — zero benefit
❌ Small files — connection setup overhead outweighs the backbone speed gain

### Pricing: "No speed, no fee"

AWS charges an extra per-GB fee for the accelerated endpoint — **unless** AWS determines the accelerated path wouldn't actually be faster than standard S3 for that request, in which case you're not charged the acceleration fee at all.

---

## 9. Encryption Fundamentals

### The two states of data

| State | Protects against | S3 mechanism |
|---|---|---|
| **In Transit** | Network interception | TLS/HTTPS; enforced via `aws:SecureTransport` condition in a bucket policy |
| **At Rest** | Physical disk theft | Server-Side Encryption (SSE), applied automatically on write, decrypted automatically on authorized read |

### The two mathematical types

**Symmetric encryption** — one shared secret key encrypts *and* decrypts.
- Fast, ideal for bulk data.
- Key distribution is the weak point — if the key leaks, everything leaks.
- S3 uses **AES-256** (symmetric) for all default SSE options (SSE-S3, SSE-KMS).

**Asymmetric encryption** — a public/private key pair; the public key locks, only the private key unlocks.
- Slower, rarely used to encrypt bulk files directly.
- You can hand out the public key freely.
- S3 uses this during the **HTTPS handshake** to safely exchange a temporary symmetric session key.

| Type | # Keys | Strength | S3 Use Case |
|---|---|---|---|
| Symmetric | 1 (shared) | Speed | Data at rest (AES-256) |
| Asymmetric | 2 (public+private) | Secure key exchange | Data in transit (TLS handshake) |

---

## 10. AWS KMS & Envelope Encryption

KMS is the managed service behind the strongest S3 encryption tier — full audit trails, granular access governance, and centralized key control.

### Key types

| Type | Who manages it | Cost | Auditable/Configurable |
|---|---|---|---|
| AWS Owned Key | AWS, internally | Free | No visibility at all |
| AWS Managed Key (`aws/s3`) | AWS, auto-created on first use | Free | View-only, can't modify policy |
| Customer Managed Key (CMK) | You | $1/mo per key | Full control — rotation, policy, audit |

**Key Policies are the ultimate authority.** Even an IAM Administrator with `AdministratorAccess` cannot use or manage a KMS key unless the *key policy itself* explicitly trusts them.

### Envelope encryption — why it exists

KMS's `Encrypt` API caps payloads at **4 KB** — nowhere near enough for multi-GB S3 objects. So S3 uses envelope encryption: a fast symmetric data key does the heavy lifting, and KMS only ever touches that small key.

**Upload flow (SSE-KMS PUT):**
1. App sends a file to S3, requesting SSE-KMS.
2. S3 calls `GenerateDataKey(KMS_Key_ID)`.
3. KMS returns **two variants**: a Plaintext Data Key (used immediately) and an Encrypted Data Key (wrapped by your CMK).
4. S3 encrypts the file with the plaintext key (AES-256).
5. S3 stores the encrypted file + the *encrypted* data key as metadata.
6. S3 wipes the plaintext key from memory.

**Download flow (SSE-KMS GET):**
1. Authorized user requests the object.
2. S3 sends the stored encrypted data key to KMS: `Decrypt(Encrypted_Data_Key)`.
3. KMS checks IAM + Key Policy; if authorized, returns the plaintext data key.
4. S3 decrypts on the fly and streams the file back.

### Real-world gotchas

- **Cross-account KMS trap**: cross-account bucket access with a CMK-encrypted bucket fails with `Access Denied` even when S3 policies are perfect — you must separately grant `kms:Decrypt` + `kms:GenerateDataKey` to the other account's principal *in the KMS key policy*.
- **KMS API throttling**: every SSE-KMS object touch is a distinct KMS API call. Regional limits (10K–40K req/sec) can throttle a high-throughput pipeline. **Fix: enable S3 Bucket Keys** — a bucket-level data key cuts KMS API volume (and cost) by up to 99%.

---

## 11. Server-Side vs Client-Side Encryption

The core question: **where does the cryptographic computation happen** — on AWS's hardware, or on yours?

### Server-Side Encryption (SSE) — three tiers

| Option | Key management | Notes |
|---|---|---|
| **SSE-S3** (AES-256) | Fully AWS-managed | Default, free, zero config |
| **SSE-KMS** | AWS KMS | Full audit trail (who decrypted what, via CloudTrail), fine-grained key policies |
| **SSE-C** | You supply the raw key on every request | AWS never stores it — lose the key, lose the data forever. **Disabled by default on new buckets**; must be explicitly re-enabled via `PutBucketEncryption` |

### Client-Side Encryption (CSE)

Data is encrypted **before** it leaves your application — S3 only ever sees ciphertext. Built with the AWS Encryption SDK / S3 Encryption Client, using either a KMS-issued data key or a fully offline local/HSM master key.

**Use CSE when:** zero-trust compliance mandates that not even AWS employees may see cleartext (financial, military, healthcare).

### Comparison

| Feature | SSE | CSE |
|---|---|---|
| Where encryption happens | Inside AWS infrastructure | Inside your app/server |
| Who sees cleartext | AWS, momentarily | Only your application |
| CPU overhead | Free (AWS absorbs it) | Borne by your compute |
| S3 Select / Athena support | ✅ Fully supported | ❌ Blocked — AWS can't parse ciphertext |
| Key-loss risk | Low (AWS manages backup) | High — CSE key loss = permanent data loss |

### Enforcing HTTPS-only at the bucket level

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "EnforceHTTPSOnly",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::production-app-assets", "arn:aws:s3:::production-app-assets/*"],
    "Condition": { "Bool": { "aws:SecureTransport": "false" } }
  }]
}
```

---

## 12. S3 Storage Classes

The fundamental trade-off: **as monthly storage cost drops, retrieval cost and latency go up.**

### Frequently accessed

- **S3 Standard** — milliseconds access, ≥3 AZ replication, no retrieval fees, no minimums. Default choice for active workloads.
- **S3 Express One Zone** — single-digit ms latency (fastest tier), lives in one AZ next to your compute, purpose-built for high-throughput analytics/AI training — up to 10x faster than Standard.

### Infrequently accessed (IA)

Both carry a **per-GB retrieval fee**, a **30-day minimum storage duration**, and a **128 KB minimum billable object size**.

- **S3 Standard-IA** — millisecond access, ≥3 AZ. For backups/DR images that must be instantly available if needed.
- **S3 One Zone-IA** — single AZ (lower durability against a full-AZ loss), 20% cheaper than Standard-IA. For easily recreated data (thumbnails, transcode caches).

### Archival (Glacier tiers)

- **Glacier Instant Retrieval** — millisecond access even though it's archival; 90-day minimum. For rarely-touched compliance/medical records that must still load instantly.
- **Glacier Flexible Retrieval** — asynchronous: Expedited (1–5 min), Standard (3–5 hrs), Bulk (5–12 hrs, free). 90-day minimum. For standard backups/deep archives.
- **Glacier Deep Archive** — 12–48 hr retrieval, the cheapest storage in all of AWS, 180-day minimum. For 7–10 year regulatory retention.

### The automated tier: Intelligent-Tiering

For unpredictable access patterns. Monitors each object and shifts it automatically: Frequent Access → (30 days untouched) → Infrequent Access → (90 days) → Archive Instant Access. **No retrieval fees ever**, just a small flat monitoring fee per 1,000 objects.

### Summary matrix

| Class | Durability | Min Duration | Retrieval Fee | Access Speed |
|---|---|---|---|---|
| S3 Standard | 11 nines | None | None | Milliseconds |
| S3 Express One Zone | 11 nines (1 AZ) | None | None | Single-digit ms |
| Standard-IA | 11 nines | 30 days | Per GB | Milliseconds |
| One Zone-IA | 11 nines (1 AZ) | 30 days | Per GB | Milliseconds |
| Glacier Instant | 11 nines | 90 days | Per GB | Milliseconds |
| Glacier Flexible | 11 nines | 90 days | Per GB | Minutes–hours |
| Glacier Deep Archive | 11 nines | 180 days | Per GB | 12–48 hours |

---

## 13. Lifecycle Configuration (Transitions & Expirations)

Lifecycle Configurations automate storage-class migration and deletion — no cron jobs, no manual scripts.

### Two types of actions

- **Transition Actions** — move objects to a cheaper storage class as they age (e.g. Standard → Standard-IA at 30 days → Glacier at 90 days).
- **Expiration Actions** — permanently remove objects at a defined age (e.g. purge raw logs after 180 days).

### Anatomy of a rule

- `ID` — unique rule name
- `Status` — `Enabled`/`Disabled`
- `Filter` — entire bucket, a prefix, or an object tag
- Actions block — the transition/expiration schedule

### Transition pathways & rules

```
[ S3 Standard ] --(30d)--> [ Standard-IA ] --(90d)--> [ Glacier Flexible ] --(180d)--> [ Glacier Deep Archive ]
```

- **You cannot transition backward** (e.g. Glacier → Standard) via a lifecycle policy.
- **30-day floor**: S3 rejects a rule that transitions to Standard-IA/One Zone-IA before 30 days.
- **128 KB minimum billing**: a 10 KB file transitioned to Glacier is still billed as 128 KB.
- **S3 Express One Zone** directory buckets can't be lifecycle-transitioned into/out of.

### Expiration behavior depends on versioning

| Bucket state | What happens on expiration |
|---|---|
| Versioning disabled | Object is permanently, instantly deleted |
| Versioning enabled | A Delete Marker is added as the new current version; the data underneath survives |

### Versioned buckets need two independent rule sets

**Current version rules** — apply to the live object (e.g. transition to Glacier at 90 days, expire — i.e. add a delete marker — at 365 days).

**Noncurrent version rules** — apply to the historical stack underneath (e.g. transition noncurrent versions to Glacier 14 days after they go noncurrent; permanently expire them after 90 days). **This is how you actually free disk space in a versioned bucket.**

### Housekeeping toggles

- **Expired Object Delete Markers** — once all historical versions under a delete marker are purged, the marker itself becomes an orphan. This toggle auto-cleans those orphans.
- **Incomplete Multipart Uploads** — auto-purges abandoned upload chunks after N days.

### The timing engine

- **Age calculation**: based on `LastModified`, rounded **up** to the next midnight UTC.
- **Execution window**: S3 evaluates all lifecycle rules **once per day, at midnight UTC**.
- **Worked example**: upload Monday 10:00 AM UTC, rule = `Days: 1` → S3 rounds the start to Tuesday 00:00 UTC → the 1-day threshold crosses at Wednesday 00:00 UTC → object is logically expired (billing stops) at that instant, with physical deletion trailing 24–48 hrs behind.

### Production limits

- **1,000 rules max** per bucket, non-adjustable.
- **Small object penalty**: never write rules that transition tiny files individually to IA/Glacier — aggregate them first.

### Terraform (log bucket with full current + noncurrent handling)

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "log_bucket_lifecycle" {
  bucket = "my-company-production-logs"

  rule {
    id     = "archive-and-cleanup-application-logs"
    status = "Enabled"

    filter { prefix = "app-logs/" }

    transition { days = 30  storage_class = "STANDARD_IA" }
    transition { days = 90  storage_class = "GLACIER" }
    expiration { days = 365 }

    noncurrent_version_transition { noncurrent_days = 14  storage_class = "GLACIER" }
    noncurrent_version_expiration { noncurrent_days = 90 }
  }
}
```

---

## 14. S3 Replication (CRR & SRR)

A fully managed, **asynchronous** feature that duplicates objects, metadata, and tags from a source bucket to one or more destinations — the backbone of DR and compliance architectures.

### Two flavors

- **Cross-Region Replication (CRR)** — copies across Regions (e.g. `ap-south-1` → `us-east-1`). For geographic compliance, low-latency global delivery, cross-continent DR.
- **Same-Region Replication (SRR)** — copies within the same Region. For centralized log aggregation from multiple environments, or keeping staging/prod stores in sync.

### Prerequisites

1. **Versioning mandatory** on *both* source and every destination bucket.
2. **IAM execution role** — S3 needs permission to `s3:GetObjectVersion` on the source and `s3:PutObject` on the destination.
3. **Bucket policy trust** — for cross-account replication, the destination bucket policy must trust the source account's replication role.

### What replicates vs. what doesn't

| Replicated | Not Replicated |
|---|---|
| New objects uploaded after the rule is enabled | Existing objects present *before* the rule was enabled (needs S3 Batch Replication) |
| Metadata & tag updates | Daisy-chained replication (A→B→C doesn't cascade — an object landing in A stops at B) |
| Delete marker creation (if configured) | Objects in Glacier tiers (must be restored/"defrosted" first) |

**Fix for pre-existing data:** run an **S3 Batch Replication** job — a one-time bulk-scan of a historical manifest.

### Advanced options

- **Multi-destination replication** — fan out to up to 15 destination buckets across regions/accounts/storage classes.
- **Ownership override** — force the destination to strip source ownership and take control of replicas.
- **S3 Replication Time Control (RTC)** — guarantees 99.99% of new objects replicate within 15 minutes, backed by a financial SLA.

### Terraform (one-way CRR)

```hcl
resource "aws_s3_bucket_replication_configuration" "replication" {
  depends_on = [aws_s3_bucket_versioning.source_versioning]
  role   = aws_iam_role.replication_exec_role.arn
  bucket = aws_s3_bucket.source.id

  rule {
    id     = "disaster-recovery-sync"
    status = "Enabled"
    destination {
      bucket        = aws_s3_bucket.destination.arn
      storage_class = "STANDARD_IA"
    }
  }
}
```

---

## 15. Centralized Log Aggregation on S3

In multi-account AWS estates, S3 is the natural "log lake" — near-infinite scale, high durability, and very low storage cost.

### Typical ingestion sources

- **Infrastructure logs** (VPC Flow Logs, CloudTrail) — natively configurable to deliver straight to a centralized bucket in a dedicated Log Archive account.
- **Application/container logs** — Fluent Bit, Logstash, or AWS FireLens run as sidecars, buffering `stdout` and streaming to S3 or Kinesis Data Firehose.
- **EC2 instance logs** — the CloudWatch Agent tails local paths and ships to CloudWatch Logs, which can subscription-filter into S3.

### Two engineering traps

**A. The "small file" performance trap** — millions of tiny log files slow down Athena/S3 Select queries badly. **Fix:** buffer through Kinesis Data Firehose (e.g. flush every 5 minutes or at 128 MB) so S3 receives fewer, larger files.

**B. The lifecycle cost trap** — S3 lifecycle transitions charge a per-object transaction fee. Transitioning 10 million tiny daily logs to Glacier can cost more in API fees than it saves in storage. **Fix:** batch/aggregate logs into tarballs (via `s3tar` or a Lambda job) before letting lifecycle rules touch them.

### Hardening a production log bucket

1. **Structure prefixes for partition scanning:**
   ```
   s3://company-central-logs/account-id=123456789012/service=vpc-flow-logs/year=2026/month=07/day=05/
   ```
2. **Enable S3 Object Lock in Compliance Mode** (e.g. 365-day retention) — for SOC2/PCI-DSS, no one (not even root) can alter the logs during the window.
3. **Layer on SRR** — clone logs asynchronously into an isolated, restricted backup bucket in the same Region for extra disaster isolation.

---

## 16. S3 Select & Glacier Select

Server-side SQL filtering: instead of downloading a whole object to filter locally, you push a `WHERE` clause down to the storage layer and only the matching rows travel over the network.

### The value proposition

Traditional approach: `GET` a 10 GB CSV, pay full egress, burn local CPU/memory parsing it.

S3 Select approach:
```sql
SELECT * FROM s3object s WHERE s.error_id = 'ERR-999'
```
S3 filters on the storage nodes and returns a few KB — cutting network, API, and compute costs by 99%+.

### Supported formats & limits

- **Input formats:** CSV, JSON (lines or arrays), Apache Parquet — optionally GZIP/BZIP2-compressed, queryable without pre-extraction.
- **SQL support:** `SELECT`, `FROM`, `WHERE`, `AS`, basic string/math ops. **No** `JOIN`, `GROUP BY`, or multi-file subqueries.

### Glacier Select

Same SQL filtering, but against cold archival tiers — necessarily asynchronous:

1. Launch a `RestoreObject` job with your SQL expression and an output bucket.
2. S3 runs the query once the archive tier "defrosts" (Expedited 1–5 min / Standard 3–5 hrs / Bulk 5–12 hrs).
3. Filtered results land directly in your target bucket — no EC2/EMR cluster needed to parse raw archive data.

### Comparison

| Feature | S3 Select | Glacier Select |
|---|---|---|
| Target classes | Standard, Standard-IA, One Zone-IA | Glacier Flexible, Glacier Deep Archive |
| Response | Synchronous, immediate | Asynchronous, background job |
| Latency | Milliseconds | Minutes–hours |
| Use case | Real-time Lambda/Athena acceleration | Ad-hoc compliance/audit investigations |

---

## 17. S3 Object Lock (WORM Compliance)

A Write-Once-Read-Many mechanism preventing deletion or overwrite for a fixed period or indefinitely. Core tool for ransomware protection and regulatory compliance (e.g. SEC Rule 17a-4).

### Foundational guardrails

- **Versioning is mandatory** and, once Object Lock is enabled, versioning can **never** be suspended.
- **Creation-time only** — you cannot retrofit Object Lock onto an existing bucket. You must create a new bucket and migrate data in via S3 Batch Operations.

### Two independent lock mechanisms (can be combined)

**A. Retention Period** — a future `Retain-Until-Date` on the object version.
- **Governance Mode** — most users can't delete/overwrite/alter, but a principal with `s3:BypassGovernanceRetention` can override. Good for ransomware protection while leaving an admin trapdoor.
- **Compliance Mode** — absolute. **No one, including the root account, can shorten or delete the lock.** It can only be extended. Use for strict regulatory audits.

**B. Legal Hold** — same deletion block, but with **no expiration date**. A binary on/off switch, released only by a user with `s3:PutObjectLegalHold`. For open investigations/discovery.

### How overlapping protections resolve

The **most restrictive** rule always wins: if the Retention Period expires but a Legal Hold is still on, the object stays locked. If the Legal Hold is lifted but the Retention window hasn't lapsed, the object stays locked.

### Interaction with everyday operations

- **Standard delete** on a locked object → S3 just drops a Delete Marker; the protected version is untouched underneath.
- **Hard delete** (targeting a specific `VersionId`) on a locked object → blocked with `403 Access Denied`.
- **Lifecycle transitions** work fine on locked objects (e.g. Standard → Glacier); **lifecycle expirations cannot delete a locked object** until the retention window lapses and all legal holds are released.

### Terraform

```hcl
resource "aws_s3_bucket" "compliance_log_vault" {
  bucket              = "company-immutable-audit-vault"
  object_lock_enabled = true   # must be set at creation
}

resource "aws_s3_bucket_versioning" "vault_versioning" {
  bucket = aws_s3_bucket.compliance_log_vault.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_object_lock_configuration" "vault_lock_config" {
  depends_on = [aws_s3_bucket_versioning.vault_versioning]
  bucket     = aws_s3_bucket.compliance_log_vault.id
  rule {
    default_retention {
      mode = "COMPLIANCE"
      days = 365
    }
  }
}
```

---

## 18. Advanced & 2025–2026 Capabilities

S3 has expanded well beyond plain object storage. These are newer, purpose-built extensions worth knowing for interviews and modern architectures.

### S3 Tables — managed Apache Iceberg tables
A dedicated bucket type ("table bucket") purpose-built for tabular analytics data in the open Apache Iceberg format. Delivers up to 3x faster queries and up to 10x higher transactions-per-second than self-managed Iceberg-on-S3, with automatic background compaction, snapshot management, and unreferenced-file cleanup. Queryable from Athena, Redshift, EMR, Spark, and other Iceberg-compatible engines via a built-in Iceberg REST Catalog. Table buckets support only S3 Standard/Intelligent-Tiering (no Glacier tiers, no presigned URLs, no public access), and table names must be lowercase.

### S3 Vectors — native vector storage for AI/RAG
A "vector bucket" type with dedicated APIs to store, index, and query high-dimensional embeddings directly in S3 — no separate vector database cluster required. Now GA at up to 2 billion vectors per index (20 trillion per bucket) with ~100ms query latency, and up to 90% lower cost than specialized vector DBs. Integrates natively with Amazon Bedrock Knowledge Bases (RAG) and Amazon OpenSearch (tiered hot/cold vector strategies). Encrypted by default with SSE-S3, or SSE-KMS if you need customer-managed keys per index.

### S3 Files — mount S3 as a native file system
Lets you mount a general-purpose S3 bucket as a real NFS v4.1+ file system on EC2, ECS, EKS, Fargate, or Lambda — no FUSE hacks, no sync pipelines, no duplicated storage. Built on Amazon EFS technology as a high-performance cache layer in front of S3 (your source of truth stays in S3). Small/hot files are served from the ~1ms-latency cache; large sequential reads stream directly from S3. Writes batch-commit back to S3 roughly every 60 seconds ("stage and commit," borrowed from version-control systems). Both the file-system view and the classic S3 object API stay simultaneously valid against the same underlying data.

### S3 Batch Operations
A managed execution engine for large-scale actions across billions of objects at once — e.g. bulk-encrypting 500 million legacy unencrypted objects, retagging an entire data lake, or running a one-time S3 Batch Replication migration for objects that predate a replication rule. Runs entirely asynchronously in the background; avoids writing (and re-writing) custom pagination loops that will inevitably time out at scale.

### Cross-Region Access Points & Account Regional Namespaces
- **Cross-Region/Multi-Region Access Points** give you a single global endpoint that automatically routes requests to the closest of several identical buckets spread across Regions.
- **Account Regional Namespaces** let you reserve bucket names scoped to *your* account + Region combination (`prefix-{accountId}-{region}-an`), instead of competing in S3's shared global namespace. This eliminates bucket-name squatting/collisions and lets deleted names stay reserved to you — a strong default for new buckets in IaC-heavy, multi-account organizations. Enforceable org-wide via the `s3:x-amz-bucket-namespace` condition key in SCPs.

### S3 Object Annotations
An upgrade over standard user metadata (which is capped at ~10 tags). Object Annotations let you attach up to 1 GB of rich, mutable, *queryable* JSON/XML/YAML context directly to a single object — designed for autonomous AI agents and automated pipelines that need to discover and act on datasets without a side database.

---

## 19. Master Decision Cheat Sheet

| I need to... | Use |
|---|---|
| Let another AWS account read/write my bucket | Cross-account bucket policy (simple) or STS AssumeRole (scalable/governed) |
| Share a file temporarily without AWS credentials | Presigned URL |
| Make external consumers pay for large downloads | Requester Pays |
| Recover from an accidental overwrite/delete | Versioning + Noncurrent lifecycle rules |
| Prevent anyone from ever deleting compliance data | Object Lock, Compliance Mode |
| Upload files larger than 5 GB, or over flaky networks | Multipart Upload |
| Speed up long-distance global uploads | Transfer Acceleration |
| Encrypt with full audit trail of who decrypted what | SSE-KMS (+ S3 Bucket Keys for cost) |
| Guarantee AWS itself never sees cleartext | Client-Side Encryption |
| Cut storage cost for rarely-accessed data | Lifecycle transitions to IA/Glacier, or Intelligent-Tiering if access is unpredictable |
| Query a huge file without downloading it whole | S3 Select (hot) or Glacier Select (cold) |
| Keep a synchronous copy in another Region for DR | Cross-Region Replication |
| Build a data lake queried with SQL engines | S3 Tables |
| Store embeddings for RAG/semantic search | S3 Vectors |
| Give an app "normal" file read/write access to S3 | S3 Files |
| Bulk-modify billions of existing objects | S3 Batch Operations |
| Guarantee my bucket name is never squatted | Account Regional Namespaces |

---

## 📚 Where to go next

- [`commands-cheatsheet.md`](./commands-cheatsheet.md) — every CLI command, JSON policy, and Terraform block in one place
- [`hands-on-labs.md`](./hands-on-labs.md) — 15 guided labs to build muscle memory in a sandbox account
- [`troubleshooting.md`](./troubleshooting.md) — a lookup table of real error messages and their fixes

