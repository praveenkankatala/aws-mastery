# AWS S3 — Troubleshooting Guide

A lookup table of real-world S3 errors, their root causes, and fixes — organized by topic to match `README.md`. When in doubt, walk the [Core Evaluation Flow](./README.md#1-s3-core-architecture): Auth → Block Public Access → Explicit Deny → Explicit Allow.

---

## Access & Permissions

| Symptom | Root Cause | Fix |
|---|---|---|
| `403 Forbidden` on a request you're sure should be allowed | Missing `Allow` in *either* IAM or bucket policy for same-account access, or missing `Allow` in **both** for cross-account access | Check both policies. Cross-account requires an explicit `Allow` on both sides — one alone is not enough. |
| `403 Forbidden` even though your policy clearly has `Allow` | An **explicit `Deny`** exists somewhere (IAM, bucket policy, SCP, or KMS key policy) that matches the request | Deny always wins over Allow, no matter where it's defined. Search every attached policy for a matching `Deny` statement. |
| `s3:PutObject` fails with Access Denied despite a policy that "looks right" | The policy's `Resource` targets the bare bucket ARN (`arn:aws:s3:::my-bucket`) instead of the object-level ARN (`arn:aws:s3:::my-bucket/*`) | Object-level actions (`GetObject`, `PutObject`, `DeleteObject`) need the `/*` suffix. Bucket-level actions (`ListBucket`) need the bare ARN. Mixing them up is the #1 policy bug. |
| You locked yourself out of a bucket entirely after a policy change | A `Deny` with `Principal: "*"` and a bad `Condition` matched *everyone*, including admins | Only the AWS **root account** can fix this. Log in as root and correct/delete the offending statement. |
| Cross-account role can list objects but every `GetObject` fails with `Access Denied` | The bucket uses SSE-KMS with a Customer Managed Key, and the KMS key policy doesn't trust the other account's principal | Add a statement to the **KMS key policy** (not just the bucket policy) granting `kms:Decrypt` + `kms:GenerateDataKey` to the external account's role. |
| An IAM Administrator can't use a specific KMS key despite having `AdministratorAccess` | KMS key policies are the ultimate authority — IAM permissions alone are not enough | The Key Policy itself must explicitly trust the identity. Add the principal to the key policy. |
| Account A can't read/modify an object that Account B uploaded into Account A's bucket | Default Object Ownership means the uploader (Account B) still owns the object, even though it lives in Account A's bucket | Set S3 Object Ownership to **Bucket Owner Enforced**, which strips ACLs and forces bucket-owner ownership on every upload. |

---

## Presigned URLs

| Symptom | Root Cause | Fix |
|---|---|---|
| Presigned URL returns `Access Denied` even though it hasn't expired | The IAM identity that *generated* the URL lost permission to the object after the URL was created | Presigned URLs are a proxy for the signer's permissions at request time. Re-check the signer's current IAM/bucket policy access. |
| A presigned URL from a Lambda/EC2 role stops working well before your `ExpiresIn` value | It was signed with **temporary** credentials (an IAM role), so it dies the moment the underlying session/role credentials expire — regardless of `ExpiresIn` | Either shorten `ExpiresIn` to fit inside the role's session duration, or sign with longer-lived IAM user credentials if appropriate for your security posture. |
| You need to revoke a shared presigned URL immediately | Presigned URLs cannot be selectively revoked once issued | Delete the object, rotate/change its encryption key, or revoke the signing principal's permissions — these are the only ways to kill it early. |
| A presigned URL that should last 7 days stops working after a few hours | It was generated using an IAM **role**, not an IAM user with long-lived access keys | Only IAM User access-key signing supports up to the full 7-day maximum; role-based signing is capped at the role's max session duration. |

---

## Requester Pays

| Symptom | Root Cause | Fix |
|---|---|---|
| `403 Forbidden` downloading from a bucket you know you have permissions on | The bucket has Requester Pays enabled and you didn't pass the accept-charges flag | Add `--request-payer requester` to the CLI call, or `RequestPayer='requester'` in the SDK call. |
| Anonymous/public users can't access a Requester Pays bucket at all | Requester Pays fundamentally requires authenticated requests so AWS knows who to bill | This is by design — Requester Pays and anonymous public access are mutually exclusive. Use a normal bucket if you need public access. |
| You enabled Requester Pays and now Static Website Hosting won't turn on (or vice versa) | The two features are incompatible — static hosting needs anonymous access, which Requester Pays blocks | Pick one. If you need both a public website and cost-shifted downloads, use separate buckets. |
| Users in the same Region as your bucket are still being billed for transfer | Not a bug — this is exactly how it's supposed to work outside true same-region cases | Confirm the requester's compute is genuinely in the *same Region* as the bucket; if it is, transfer cost should already be $0 (only the tiny request fee applies). |

---

## Versioning & Delete Markers

| Symptom | Root Cause | Fix |
|---|---|---|
| A "deleted" file still shows up when listing versions, and storage costs haven't dropped | Deleting an object in a versioned bucket only adds a Delete Marker — the underlying data is still there | This is expected behavior, not a bug. To actually free space, permanently delete the specific `VersionId`, or let a `NoncurrentVersionExpiration` lifecycle rule handle it. |
| You "undeleted" a file but it's still returning `404` | You deleted the wrong version, or a *newer* Delete Marker was added after the one you removed | Run `list-object-versions` again and confirm you removed the Delete Marker that was actually the current version at the time of the 404. |
| Bucket storage costs keep climbing even though the live files aren't growing | Noncurrent versions are piling up indefinitely with no lifecycle rule managing them | Add a `NoncurrentVersionExpiration` rule (with `NewerNoncurrentVersions` as a rollback safety net) to purge old versions automatically. |
| You can't figure out how to disable versioning entirely | Versioning cannot be *disabled* once enabled — only suspended | Use `Status=Suspended` instead. New uploads get a `null` version ID; existing history is preserved. |
| MFA Delete won't toggle even for an IAM Administrator | MFA Delete can only be enabled/disabled using **root account** credentials | Log in as root (or use root API credentials) to change this specific setting — no IAM policy can grant this permission to a non-root identity. |

---

## Multipart Upload

| Symptom | Root Cause | Fix |
|---|---|---|
| `CompleteMultipartUpload` fails with an `InvalidPart` or ETag mismatch error | The `PartNumber`→`ETag` mapping sent in the completion manifest doesn't exactly match what `UploadPart` returned | Re-verify you recorded the exact ETag string (including quotes) returned by each `UploadPart` call, in the correct `PartNumber` order. |
| Lifecycle rule creation fails, or a bucket's storage bill has mysterious "phantom" usage | Failed/abandoned multipart uploads leave orphaned parts that don't show up in `aws s3 ls` but are still billed | Run `list-multipart-uploads` to find orphans, `abort-multipart-upload` to clear them, and add an `AbortIncompleteMultipartUpload` lifecycle rule going forward. |
| Multipart upload of a huge file fails validation before it even starts | You've exceeded 10,000 parts because your chosen part size is too small for the file size | Recalculate: `parts = file_size / part_size` must be ≤ 10,000. For a 100 GB file, use 25–100 MB chunks, not 5 MB. |
| A single `PutObject`/`UploadPart` call fails for a very large chunk | Part size exceeds the 5 GB per-part maximum | Split into smaller parts, staying within the 5 MB–5 GB range per part. |

---

## Encryption & KMS

| Symptom | Root Cause | Fix |
|---|---|---|
| `ThrottlingException` on high-volume S3 uploads/downloads with SSE-KMS | Every SSE-KMS object touch is a distinct KMS API call, and you've hit the region's KMS request-rate limit | Enable **S3 Bucket Keys** on the bucket — this cuts KMS API call volume (and cost) by up to 99% by using a bucket-level data key. |
| Cross-account read of a CMK-encrypted object fails even though S3 bucket policy is correct | The KMS key policy in the bucket-owner's account doesn't trust the external account/role | Add `kms:Decrypt` + `kms:GenerateDataKey` for the external principal directly in the KMS key policy — the bucket policy alone is not sufficient. |
| You lost an SSE-C encryption key and can no longer read the object | AWS never stores SSE-C keys — you are 100% responsible for key custody | There is no recovery path. This is why SSE-C is disabled by default on new buckets; only re-enable it if your application can guarantee robust key management. |
| SSE-C uploads fail with a configuration error on a brand-new bucket | SSE-C is disabled by default on new S3 buckets as a security default | Explicitly re-enable it via `PutBucketEncryption` if your use case genuinely requires customer-supplied keys. |
| S3 Select or Athena can't read objects that were uploaded fine | The objects were encrypted with **Client-Side Encryption** — AWS-side tools can't decrypt CSE ciphertext to parse it | This is an inherent CSE trade-off, not a bug. If you need S3 Select/Athena, use SSE (S3 or KMS) instead of CSE for those objects. |
| `PutObject` fails with a policy violation for unencrypted uploads | A bucket policy denies requests missing a specific `x-amz-server-side-encryption` header/condition | Explicitly set the required SSE header/parameter on the upload call to match what the bucket policy demands. |

---

## Storage Classes & Lifecycle

| Symptom | Root Cause | Fix |
|---|---|---|
| Lifecycle rule creation is rejected outright | A transition to Standard-IA or One Zone-IA is scheduled before the 30-day minimum | Set the transition's `Days` to 30 or greater. |
| Small files transitioned to Glacier/IA cost more than expected | All IA/Glacier tiers enforce a **128 KB minimum billable object size**, regardless of actual file size | Aggregate small files into larger archives before transitioning them, or accept the minimum-size billing floor. |
| Objects "disappeared" from an S3 Express One Zone directory bucket after a lifecycle policy ran | Express One Zone buckets don't support standard lifecycle transitions into/out of them | Manage Express One Zone data lifecycle at the application level instead of relying on standard S3 Lifecycle rules. |
| An object expired according to billing, but is still physically present hours later | S3 marks objects as logically expired (stops billing) immediately at the threshold, but physical deletion is asynchronous | This is expected — physical cleanup can trail by 24–48 hours. Don't rely on immediate physical removal for compliance purposes; use Object Lock expiration semantics if you need stronger guarantees. |
| A lifecycle rule you just applied doesn't seem to have run yet | Lifecycle rules are evaluated **once per day at midnight UTC**, not continuously | Wait for the next UTC midnight evaluation cycle, or double-check the rule's date math using the age-calculation rules in the README. |
| You've hit a wall adding more lifecycle rules to a bucket | A single bucket supports a hard maximum of 1,000 lifecycle rules | Consolidate rules using broader prefixes/tags rather than one rule per narrow use case. |
| Objects transition from Standard to Glacier fine, but you can't move them back | Lifecycle transitions are one-directional — you cannot transition "backward" to a warmer tier via a lifecycle policy | To move data back to Standard, perform an explicit `CopyObject` with the target storage class — lifecycle rules alone can't do it. |

---

## Replication

| Symptom | Root Cause | Fix |
|---|---|---|
| `PutBucketReplication` fails immediately | Versioning isn't enabled on the source and/or destination bucket | Enable versioning on **both** buckets first — replication is entirely dependent on Version IDs. |
| Objects that existed before you set up replication never show up in the destination | Replication only applies to objects created **after** the rule was enabled — it's not retroactive | Run an **S3 Batch Replication** job to bulk-copy the pre-existing historical objects. |
| A 3-bucket replication chain (A→B→C) stops working past the first hop | S3 Replication does not daisy-chain by default — an object landing in B via replication does not automatically re-trigger B's own replication rule to C | Design for this explicitly, or use S3 Batch Operations / an event-driven Lambda to bridge the gap if you truly need multi-hop replication. |
| Objects sitting in Glacier storage classes aren't replicating | Live replication can't evaluate objects that are archived/"frozen" in Glacier tiers | Restore ("defrost") the object first if it needs to be replicated, or reconsider your architecture so archival happens after replication, not before. |
| Cross-account replication fails with permission errors | The destination bucket policy doesn't trust the source account's replication IAM role | Add a statement to the destination bucket policy explicitly trusting the source account's replication role ARN. |
| Replication is technically working, but takes much longer than expected for compliance SLAs | Standard S3 Replication is asynchronous with variable timing based on bandwidth and object size | Enable **S3 Replication Time Control (RTC)** for a guaranteed 99.99%-within-15-minutes SLA (with a financial backing). |

---

## Object Lock

| Symptom | Root Cause | Fix |
|---|---|---|
| Can't enable Object Lock on an existing production bucket | Object Lock can only be turned on at **bucket creation time** | Create a new bucket with Object Lock enabled and migrate historical data into it via S3 Batch Operations. |
| A delete under Governance Mode retention fails unexpectedly | The calling principal lacks `s3:BypassGovernanceRetention`, or you didn't pass the bypass flag | Grant the permission and include `--bypass-governance-retention` in the delete call, if the business case genuinely justifies an override. |
| A delete under Compliance Mode retention fails, and no one — including root — can override it | This is Compliance Mode working exactly as designed: it is absolute and cannot be bypassed by any identity, including root | There is no fix or override. The Retain-Until-Date can only be **extended**, never shortened or removed. Plan retention periods carefully before using Compliance Mode. |
| An object stays locked even though its retention period already expired | An active **Legal Hold** is still applied independently of the retention timer | Explicitly turn off the Legal Hold (`s3:PutObjectLegalHold`) — retention and legal hold are evaluated independently, and the most restrictive one wins. |
| A lifecycle expiration rule isn't deleting objects you expect it to | The objects are still under an active retention period or Legal Hold | Lifecycle expiration cannot override Object Lock protections. Wait for the retention window to lapse and release any Legal Holds first. |
| Versioning got suspended unexpectedly, breaking assumptions elsewhere | Object Lock forces versioning to stay enabled permanently once it's turned on | This is expected — don't design workflows that assume you can suspend versioning on an Object Lock–enabled bucket, because you can't. |

---

## Transfer Acceleration

| Symptom | Root Cause | Fix |
|---|---|---|
| Enabling Transfer Acceleration made uploads slower or unchanged | Traffic is same-Region, or files are small enough that connection-setup overhead outweighs any backbone advantage | Only expect a meaningful gain for large files crossing long geographic distances. Benchmark before committing (see Lab 14). |
| You're being charged an acceleration fee even for a run that wasn't actually faster | This should not happen — AWS's "no speed, no fee" policy is supposed to waive the fee automatically when TA wouldn't help | Double-check your billing breakdown; if you consistently see acceleration fees on same-region traffic, verify you're not accidentally still targeting the `.s3-accelerate` endpoint for local traffic. |
| Uploads to the accelerated endpoint fail while the standard endpoint works fine | Acceleration wasn't actually enabled on the bucket, or the accelerated endpoint URL is malformed | Confirm with `get-bucket-accelerate-configuration`, and double check the endpoint format: `<bucket>.s3-accelerate.amazonaws.com`. |

---

## S3 Select / Glacier Select

| Symptom | Root Cause | Fix |
|---|---|---|
| SQL query fails with a syntax/unsupported-feature error | S3 Select only supports basic `SELECT`/`FROM`/`WHERE`/`AS` and simple operations — no `JOIN`, `GROUP BY`, or multi-file subqueries | Simplify the query, or pull the filtered results into a proper query engine (Athena, Redshift Spectrum) for anything relational. |
| Query against a Glacier object doesn't return results immediately | Glacier Select is inherently asynchronous — the archive must "defrost" first | Check your chosen retrieval tier's timing (Expedited 1–5 min / Standard 3–5 hrs / Bulk 5–12 hrs) and poll the output bucket rather than expecting a synchronous response. |
| S3 Select returns a "cannot parse" error on a valid-looking file | The input format/compression specified in the API call doesn't match the actual file (e.g. told it's CSV when it's JSON, or omitted a compression type) | Match `InputSerialization` exactly to the file's real format and compression. |

---

## Log Aggregation at Scale

| Symptom | Root Cause | Fix |
|---|---|---|
| Athena queries against your log bucket are painfully slow | Millions of tiny log files are hitting the "small file" performance trap | Buffer through Kinesis Data Firehose (time- or size-based flushing) to produce fewer, larger files before they land in S3. |
| Lifecycle transition costs exceed the storage savings you expected | Per-object transaction fees on millions of small files can outweigh the cost benefit of moving them to Glacier | Aggregate/tarball historical logs before transitioning them, rather than transitioning millions of individual small objects. |
| Compliance auditors flag that log files could theoretically be altered | The log bucket doesn't have Object Lock (WORM) protection | Enable S3 Object Lock in Compliance Mode with an appropriate retention window on the log bucket. |

---

## Quick Diagnostic Checklist

When something in S3 "just isn't working," walk this list top to bottom before assuming it's a bug:

1. **Is the request authenticated?** Anonymous calls fail unless the bucket explicitly allows public access.
2. **Is Block Public Access on, and is this an internet-origin request?** → likely `403`.
3. **Search every policy layer for an explicit `Deny`** — IAM, bucket policy, SCP, KMS key policy. Deny always wins.
4. **Is there an explicit `Allow` in at least one policy** (both, if cross-account)?
5. **Bucket-level vs object-level ARN** — did you use `/*` where needed?
6. **If KMS is involved**, does the **key policy** (not just IAM/bucket policy) trust the caller?
7. **If cross-account**, did you handle Object Ownership so the bucket owner actually owns uploaded objects?
8. **If versioned**, are you operating on the *current* version, or does a stray Delete Marker/Legal Hold/Retention lock explain the behavior?
9. **If it's a timing issue**, remember lifecycle rules run once daily at midnight UTC — not instantly.

---

For deeper conceptual background on any of these, jump back to [`README.md`](./README.md); for the exact commands to reproduce or fix an issue, see [`commands-cheatsheet.md`](./commands-cheatsheet.md).
