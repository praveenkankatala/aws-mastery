# CloudTrail — Troubleshooting Guide

Common issues, real error messages, root causes, and fixes.

---

## Table of Contents

1. [Events Missing / Not Appearing](#1-events-missing--not-appearing)
2. [Trail Creation & S3 Delivery Errors](#2-trail-creation--s3-delivery-errors)
3. [CloudWatch Logs Integration Errors](#3-cloudwatch-logs-integration-errors)
4. [KMS Encryption Errors](#4-kms-encryption-errors)
5. [Integrity Validation Issues](#5-integrity-validation-issues)
6. [Athena Query Problems](#6-athena-query-problems)
7. [Insights & Data Events Issues](#7-insights--data-events-issues)
8. [Organization Trail Issues](#8-organization-trail-issues)
9. [Cost Surprises](#9-cost-surprises)
10. [Reading Events — Interpretation Pitfalls](#10-reading-events--interpretation-pitfalls)

---

## 1. Events Missing / Not Appearing

### "I performed an action but can't find it in Event History"
1. **Ingestion delay** — events typically take **5–15 minutes** to appear (up to ~20). Wait before concluding it's missing.
2. **It's a data event** — `GetObject`, `PutObject`, Lambda `Invoke` **never** appear in Event History; they only reach a trail with data events enabled.
3. **Wrong region selected** — Event History in the console is region-scoped; a `us-east-1` action won't show while you're viewing `ap-south-1`. Global services (IAM, STS, CloudFront) log to `us-east-1`.
4. **90-day window** — Event History only covers the rolling last 90 days; older events exist only in a trail's S3 archive.
5. **Not all services/actions are logged** — a few services and read-heavy data-plane calls aren't covered; check the service's CloudTrail documentation page.

### "My trail bucket has no files"
1. **Logging never started** — `create-trail` does **not** start logging. Check `get-trail-status` → `IsLogging` must be `true`; run `start-logging`.
2. Delivery cadence — files land roughly every 5 minutes *when there is activity*; a silent account produces few files.
3. Delivery failing silently — `get-trail-status` shows `LatestDeliveryError` (usually a bucket-policy or KMS issue; see §2/§4).

### "Events from another region are missing"
- **Cause:** single-region trail.
- **Fix:** `update-trail --is-multi-region-trail`; also `--include-global-service-events` for IAM/STS/CloudFront.

---

## 2. Trail Creation & S3 Delivery Errors

### `InsufficientS3BucketPolicyException: Incorrect S3 bucket policy is detected`
- **Cause:** the bucket policy doesn't grant the CloudTrail service principal `s3:GetBucketAcl` + `s3:PutObject` on the exact `AWSLogs/<account-id>/*` path.
- **Fix:** apply the policy from the cheatsheet §8. Verify the account ID and (if used) `s3:x-amz-acl: bucket-owner-full-control` condition. If you added an `aws:SourceArn` condition, it must match the trail ARN **exactly** — including region and trail name.

### `S3BucketDoesNotExistException`
- **Cause:** typo, or bucket in a different partition/deleted.
- **Fix:** bucket must exist before trail creation; CloudTrail doesn't create it via CLI (console can).

### `LatestDeliveryError: AccessDenied` after things worked before
- **Likely causes:** someone tightened the bucket policy, enabled default-deny SCPs, changed bucket ownership controls, or rotated/disabled the KMS key.
- **Fix:** re-apply the service-principal policy; check `get-trail-status` timestamps to pinpoint when delivery broke.

### `MaximumNumberOfTrailsExceededException`
- **Cause:** limit of 5 trails per region.
- **Fix:** delete unused trails, or use one multi-region trail instead of many single-region ones (also cheaper — first copy of management events is free once).

### `TrailAlreadyExistsException`
- Trail names are unique per account+region; pick a new name or update the existing trail.

---

## 3. CloudWatch Logs Integration Errors

### `InvalidCloudWatchLogsRoleArnException` / `InvalidCloudWatchLogsLogGroupArnException`
- **Cause:** malformed ARNs — the log-group ARN must end with `:*`, and the role must exist in the same account.
- **Fix:**
  `arn:aws:logs:<region>:<acct>:log-group:CloudTrail/SecurityEvents:*`

### CloudTrail wired to CW Logs but the log group stays empty
1. **Trust policy wrong** — the role must trust `cloudtrail.amazonaws.com` (not your user).
2. **Role permissions** — needs `logs:CreateLogStream` and `logs:PutLogEvents` on that log group.
3. Wait 10–15 minutes and generate fresh activity — only *new* events stream; historical S3 files are never backfilled into CW Logs.

### Metric filter never matches (alarm stays quiet during tests)
1. **JSON filter syntax** — CloudTrail events are JSON; patterns must use `{ $.field = value }` syntax, not plain-text patterns.
2. Field path typos: it's `$.userIdentity.type`, `$.eventName`, `$.errorCode` — case-sensitive.
3. Test offline:
   ```bash
   aws logs test-metric-filter \
     --filter-pattern '{ $.eventName = StopLogging }' \
     --log-event-messages '{"eventName":"StopLogging","userIdentity":{"type":"IAMUser"}}'
   ```
4. Alarm side: use `--treat-missing-data notBreaching` — security metrics are (hopefully) zero most of the time, otherwise the alarm parks in `INSUFFICIENT_DATA`.

---

## 4. KMS Encryption Errors

### `InsufficientEncryptionPolicyException` when setting `--kms-key-id`
- **Cause:** the key policy doesn't allow CloudTrail (`kms:GenerateDataKey*` with the CloudTrail encryption-context condition).
- **Fix:** add the key-policy statement from cheatsheet §7. The key must be in the **same region as the trail's home region** and symmetric.

### "Access Denied" when *reading* delivered log files
- **Cause:** your IAM principal can reach the S3 object but lacks `kms:Decrypt` on the CMK.
- **Fix:** grant the security team `kms:Decrypt` in the key policy — S3 permissions alone are not enough for SSE-KMS objects. (This is also the *feature*: misconfigured bucket ≠ readable logs.)

### Delivery stops after key rotation/disable
- `get-trail-status` → `LatestDeliveryError` mentioning KMS. Re-enable the key or point the trail at a valid key.

---

## 5. Integrity Validation Issues

### `validate-logs` reports files as `INVALID`
- **Meaning:** the file's hash no longer matches the signed digest — it was **modified, truncated, or replaced** after delivery. Treat as a security incident: preserve the bucket (versioning helps — the original object version may still exist), check S3 access logs / CloudTrail S3 data events for who touched it.

### "No digest files found" / validation returns nothing
1. **Validation wasn't enabled** when the logs were written — `--enable-log-file-validation` only applies going forward.
2. Digests are generated **hourly** — a trail younger than an hour has none yet.
3. Digest files were moved/deleted from `.../CloudTrail-Digest/` — the chain breaks; validation can't vouch for that window.

### Files reported as "skipped"
- Files delivered while validation was disabled, or delivered outside the requested time window. Narrow/adjust `--start-time`.

---

## 6. Athena Query Problems

### Query returns zero rows but the S3 data exists
1. **`LOCATION` path wrong** — must point at `.../AWSLogs/<account-id>/CloudTrail/` (not the bucket root, not including region).
2. **Partition projection template mismatch** — `storage.location.template` must mirror the actual key layout exactly (`${region}/${date}` with `yyyy/MM/dd`).
3. Querying partitioned columns you never populated — with projection, filter on the projected `region`/`date` columns; without projection you must `MSCK REPAIR TABLE` / add partitions manually (which is why projection is recommended).

### `HIVE_BAD_DATA` / JSON parse errors
- **Cause:** wrong SerDe or the table pointed at digest files too.
- **Fix:** use `org.apache.hive.hcatalog.data.JsonSerDe`; ensure `LOCATION` excludes the `CloudTrail-Digest/` and `CloudTrail-Insight/` prefixes.

### Queries are slow and expensive
- **Cause:** full-bucket scans (Athena bills per TB scanned).
- **Fix:** always filter on `region` and `date` partitions first; use `LIMIT` while iterating; consider converting hot ranges to Parquet, or move interactive querying to CloudTrail Lake.

### `Access Denied` from Athena
- Athena needs: read on the CloudTrail bucket, write on the query-results bucket, and `kms:Decrypt` if logs are CMK-encrypted.

---

## 7. Insights & Data Events Issues

### Insights enabled but never fires
- **Cause:** Insights flags deviation **from your account's learned baseline** — it needs days of history, and a quiet account may never deviate enough. Also it analyzes **write** management APIs' call rates, not everything.
- **Fix:** patience in production; don't expect fireworks in a fresh lab account. Verify enablement with `get-insight-selectors`.

### Data events cost exploded overnight
- **Cause:** enabling S3 data events with `"Values": ["arn:aws:s3"]` (ALL buckets) or Lambda `Invoke` on all functions — every object read in the account became a billed event.
- **Fix:** scope with **advanced event selectors** to specific buckets/prefixes/event names; use `ReadWriteType: WriteOnly` if reads don't matter; consider Data Events Aggregation summaries for high-volume access-pattern monitoring.

### Data events not visible in Event History
- Expected behavior — data events only go to trail destinations (S3/CW Logs/Lake), never to the free Event History view.

---

## 8. Organization Trail Issues

### `OrganizationNotInAllFeaturesModeException` / can't create org trail
- **Cause:** AWS Organizations must be in **All Features** mode (not consolidated-billing-only), and trusted access enabled.
- **Fix:**
  ```bash
  aws organizations enable-aws-service-access --service-principal cloudtrail.amazonaws.com
  ```
  Run trail creation from the **management account** (or a delegated administrator).

### Member accounts see the trail but get `AccessDenied` modifying it
- **Expected & by design** — members cannot stop/modify an org trail. That's the anti-insider control.

### Member-account events missing in the central bucket
- Bucket policy must allow writes for `AWSLogs/<org-id>/*` (the org path), not just the management account's path; re-check after adding new member accounts.

---

## 9. Cost Surprises

| Symptom | Root cause | Fix |
|---|---|---|
| Sudden CloudTrail line item | Data events enabled broadly (all S3 buckets / all Lambda invokes) | Scope with advanced event selectors; WriteOnly where possible |
| Paying for management events | Second+ trail delivering the same management events (only the first copy is free) | Consolidate into one multi-region trail |
| S3 storage creep | Audit logs kept forever in Standard class | S3 lifecycle: transition to Glacier after 90 days, expire per retention policy (respect compliance minimums / Object Lock) |
| CW Logs ingestion cost | Full trail streamed to CW Logs with long retention | Stream for alerting only, keep CW retention short (30–90 d) — S3 is the archive |
| Athena bill spikes | Unpartitioned full-history scans | Partition projection + date filters (see §6) |
| Lake more expensive than expected | Per-event ingestion + retention pricing | Compare against S3+Athena at your volume; use Lake for management events, S3 for bulk data events |

---

## 10. Reading Events — Interpretation Pitfalls

### "The sourceIPAddress is an AWS domain, not an IP"
- A value like `cloudformation.amazonaws.com` means an **AWS service made the call on your behalf** (e.g., CloudFormation creating resources from your stack). Trace the human via the originating `CreateStack` event instead.

### "userIdentity is an assumed role — who's the actual person?"
- Look at `userIdentity.sessionContext.sessionIssuer` for the role, and the role-session name in the `arn` (often the username or tool). For SSO, the session name typically embeds the user's identity. Correlate with the earlier `AssumeRole`/`AssumeRoleWithSAML` event for the full chain.

### "I see events with errorCode — did the action happen?"
- **No.** `errorCode` present (e.g., `AccessDenied`, `Client.UnauthorizedOperation`) means the call **failed**. Bursts of these = someone probing permissions — alert-worthy, but the resource wasn't changed.

### "responseElements is null — is the event broken?"
- Normal for **read-only** calls and some async operations; `requestParameters` still shows what was asked.

### "Timestamps don't line up with my local logs"
- `eventTime` is always **UTC**. Convert before building incident timelines (IST = UTC+5:30).

---

## Quick Diagnostic Flowchart

```
Missing event?
 ├── < 15 min old?            -> wait, ingestion delay
 ├── Data-plane call?         -> needs data events on a trail (never in Event History)
 ├── Different region?        -> check that region / multi-region trail / global events
 └── > 90 days old?           -> only in the S3 archive (Athena), not Event History

Trail not delivering?
 ├── get-trail-status         -> IsLogging? LatestDeliveryError?
 ├── AccessDenied             -> S3 bucket policy (§2) or KMS key policy (§4)
 └── CW Logs empty            -> role trust + permissions (§3), new events only

Alert never fires?
 ├── test-metric-filter with a sample event (JSON syntax!)
 ├── treat-missing-data notBreaching on the alarm
 └── SNS subscription confirmed?
```
