# AWS S3 — Hands-On Labs

15 guided labs to build real muscle memory. Each lab lists a **goal**, **prerequisites**, **numbered steps**, a **validation check**, and a **cleanup** step so you don't leave billable resources running in your sandbox account.

> ⚠️ Run these in a personal sandbox/dev AWS account, never in production. Replace every placeholder (`YOUR_ACCOUNT_ID`, bucket names, etc.) with your own values — S3 bucket names must be globally unique.

---

## Lab 1 — Bucket Policy Fundamentals & the PARC Model

**Goal:** Create a bucket, lock down public access, and grant read-only access to one IAM role via a bucket policy.

**Prerequisites:** AWS CLI configured, an existing IAM role to test with.

**Steps:**
1. Create the bucket: `aws s3 mb s3://parc-lab-<your-initials>-<random>`
2. Confirm Block Public Access is on by default: `aws s3api get-public-access-block --bucket <bucket>`
3. Write a bucket policy granting `s3:GetObject` + `s3:ListBucket` to one specific role ARN (see `commands-cheatsheet.md` → Bucket Policy).
4. Apply it: `aws s3api put-bucket-policy --bucket <bucket> --policy file://policy.json`
5. Upload a test object, then try to `GetObject` using credentials **not** covered by the policy — confirm you get `403`.
6. Assume the allowed role and confirm the same `GetObject` now succeeds.

**Validation:** The disallowed identity gets `403 Access Denied`; the allowed role gets `200 OK`.

**Cleanup:** `aws s3 rm s3://<bucket> --recursive && aws s3 rb s3://<bucket>`

---

## Lab 2 — Cross-Account Access via STS AssumeRole

**Goal:** Set up Account B assuming a role in Account A to read a bucket, without ever touching the bucket policy again for new consumers.

**Prerequisites:** Two AWS accounts (or two IAM users simulating them), admin access to both.

**Steps:**
1. **In Account A:** create bucket `cross-account-lab-a-<random>` and upload a test file.
2. **In Account A:** create IAM role `CrossAccountS3AccessRole` with a trust policy trusting Account B's account ID, and a permissions policy allowing `s3:GetObject`/`s3:ListBucket` on the bucket.
3. **In Account B:** create an IAM policy allowing only `sts:AssumeRole` targeting that role's ARN, attach it to a test user/role.
4. **In Account B:** run `aws sts assume-role --role-arn arn:aws:iam::ACCOUNT_A_ID:role/CrossAccountS3AccessRole --role-session-name lab2`
5. Export the returned temporary credentials and run `aws s3 ls s3://cross-account-lab-a-<random>/` using them.

**Validation:** The list/get succeeds using only the temporary STS credentials — Account B never needed direct bucket-policy access.

**Cleanup:** Delete the role in Account A, empty and delete the bucket.

---

## Lab 3 — Object Ownership & the Cross-Account Upload Trap

**Goal:** Reproduce and fix the classic "Account A can't read what Account B uploaded" bug.

**Prerequisites:** Completed Lab 2's cross-account setup, with `s3:PutObject` added to the role's permissions.

**Steps:**
1. Confirm the bucket's Object Ownership is **not** yet `Bucket Owner Enforced`: `aws s3api get-bucket-ownership-controls --bucket <bucket>`
2. From Account B (via the assumed role), upload a file: `aws s3 cp test.txt s3://<bucket>/from-account-b.txt`
3. From Account A, try to read it directly with the bucket owner's own root/admin credentials — if ACLs are in play, note any permission friction.
4. Apply the fix: `aws s3api put-bucket-ownership-controls --bucket <bucket> --ownership-controls '{"Rules":[{"ObjectOwnership":"BucketOwnerEnforced"}]}'`
5. Re-upload from Account B and confirm Account A now has full, unambiguous ownership.

**Validation:** After the fix, `head-object` from Account A shows the bucket owner as the sole owner, with no ACL complications.

**Cleanup:** Same as Lab 2.

---

## Lab 4 — Presigned URLs for Direct Browser Upload

**Goal:** Let a "frontend" upload a file straight to S3 using a presigned PUT URL, bypassing any backend server for the transfer itself.

**Prerequisites:** Python + boto3 installed locally.

**Steps:**
1. Create bucket `presigned-lab-<random>`, keep Block Public Access on (no public bucket needed).
2. Write a small Python script using `generate_presigned_url('put_object', ...)` with a 5-minute expiry (see cheatsheet).
3. Run the script to print a presigned PUT URL.
4. From your terminal, upload a file with plain `curl` (no AWS credentials involved):
   `curl -X PUT -T ./testfile.txt "<presigned-url>"`
5. Confirm the object landed: `aws s3 ls s3://presigned-lab-<random>/`
6. Wait past the expiry window, retry the same URL, and confirm it now fails.

**Validation:** Upload succeeds within the window and fails with `AccessDenied`/expired-request errors afterward.

**Cleanup:** `aws s3 rm s3://<bucket> --recursive && aws s3 rb s3://<bucket>`

---

## Lab 5 — Requester Pays

**Goal:** Configure a Requester Pays bucket and prove that unauthenticated/no-flag requests fail while authenticated + flagged requests succeed.

**Steps:**
1. Create bucket `requester-pays-lab-<random>` and upload a moderately sized test file.
2. Enable Requester Pays: `aws s3api put-bucket-request-payment --bucket <bucket> --request-payment-configuration '{"Payer":"Requester"}'`
3. Try downloading **without** the flag: `aws s3 cp s3://<bucket>/testfile.zip .` → expect `403 Forbidden`.
4. Retry **with** the flag: `aws s3 cp s3://<bucket>/testfile.zip . --request-payer requester` → succeeds.
5. Try enabling Static Website Hosting on the same bucket and observe that AWS rejects the combination.

**Validation:** Step 3 fails, step 4 succeeds, step 5 is rejected by AWS as an invalid combination.

**Cleanup:** Disable Requester Pays, empty and delete the bucket.

---

## Lab 6 — Versioning, Delete Markers & Recovery

**Goal:** Simulate an accidental delete and recover the file using version history.

**Steps:**
1. Create bucket `versioning-lab-<random>` and enable versioning.
2. Upload `report.txt` three times with slightly different content each time (three versions stack up).
3. List all versions: `aws s3api list-object-versions --bucket <bucket> --prefix report.txt`
4. Delete the object normally: `aws s3 rm s3://<bucket>/report.txt`
5. Confirm a plain `aws s3 ls` shows it "gone," but `list-object-versions` shows a new Delete Marker as the current version.
6. Recover it: `aws s3api delete-object --bucket <bucket> --key report.txt --version-id <DELETE_MARKER_VERSION_ID>`
7. Confirm `aws s3 cp s3://<bucket>/report.txt -` streams the most recent real content again.

**Validation:** The file "reappears" after removing the delete marker, with the correct prior content.

**Cleanup:** Permanently delete every version (loop `delete-object` with each `--version-id`), then delete the bucket.

---

## Lab 7 — Multipart Upload (Manual Walkthrough)

**Goal:** Perform a multipart upload by hand, end-to-end, to internalize the 3-phase lifecycle.

**Steps:**
1. Create a ~30 MB test file: `dd if=/dev/urandom of=bigfile.bin bs=1M count=30` (Linux/macOS) or an equivalent.
2. Split it into 3 parts of 10 MB each: `split -b 10m bigfile.bin part_`
3. Initiate: `aws s3api create-multipart-upload --bucket <bucket> --key bigfile.bin` → note the `UploadId`.
4. Upload each part with `upload-part`, recording the returned `ETag` for each `PartNumber`.
5. Build `parts-manifest.json` from the recorded ETags.
6. Complete: `aws s3api complete-multipart-upload --bucket <bucket> --key bigfile.bin --upload-id <ID> --multipart-upload file://parts-manifest.json`
7. Download the reassembled file and diff it against the original to confirm byte-for-byte integrity.

**Validation:** `diff bigfile.bin downloaded-bigfile.bin` shows no differences.

**Bonus:** Deliberately abandon a multipart upload (initiate, upload one part, never complete). Run `list-multipart-uploads` to see the orphan, then apply the "abort incomplete multipart uploads after 7 days" lifecycle rule and explain why it matters for cost.

**Cleanup:** Abort any leftover multipart uploads, delete the object and bucket.

---

## Lab 8 — SSE-KMS & the Cross-Account KMS Trap

**Goal:** Encrypt a bucket with a Customer Managed Key, then reproduce and fix the cross-account KMS `Access Denied` gotcha.

**Steps:**
1. Create a CMK: `aws kms create-key --description "S3 lab CMK"` → note the Key ID.
2. Create bucket `kms-lab-<random>` and set default encryption to SSE-KMS using that key (see cheatsheet).
3. Upload a file and confirm via `head-object` that `ServerSideEncryption: aws:kms` is set.
4. Repeat the cross-account setup from Lab 2, this time granting the Account B role `s3:GetObject` in the bucket policy — **but do not yet touch the KMS key policy.**
5. From Account B, try to download the object → expect `Access Denied`, even though the S3-level policy is correct.
6. Fix it: add a statement to the KMS key policy granting Account B's role `kms:Decrypt` and `kms:GenerateDataKey`.
7. Retry the download from Account B → succeeds.

**Validation:** Step 5 fails with `Access Denied`; step 7 succeeds after the key-policy fix.

**Cleanup:** Delete the bucket contents/bucket, schedule the CMK for deletion (`aws kms schedule-key-deletion`).

---

## Lab 9 — Enabling S3 Bucket Keys and Observing the Cost Impact

**Goal:** Understand why S3 Bucket Keys matter for high-throughput SSE-KMS workloads.

**Steps:**
1. Reuse the bucket from Lab 8 (SSE-KMS enabled, `BucketKeyEnabled: false`).
2. Upload ~50 small objects in a loop, noting (via CloudTrail, if enabled) the number of distinct KMS `Decrypt`/`GenerateDataKey` API calls generated.
3. Enable S3 Bucket Keys on the bucket (see cheatsheet).
4. Upload another ~50 small objects the same way.
5. Compare KMS API call volume between the two batches in CloudTrail Event History (filter by `eventSource: kms.amazonaws.com`).

**Validation:** The second batch (Bucket Keys enabled) shows dramatically fewer direct KMS API calls for the same object count.

**Cleanup:** Same as Lab 8.

---

## Lab 10 — Storage Classes & Lifecycle Transitions

**Goal:** Build a full lifecycle policy that ages data from Standard → IA → Glacier → expiration, and understand the 30-day/128 KB rules.

**Steps:**
1. Create bucket `lifecycle-lab-<random>`, upload a handful of small text files under prefix `logs/`.
2. Try applying a lifecycle rule that transitions to `STANDARD_IA` at `Days: 10` → observe AWS reject it (must be ≥30 days).
3. Correct it to `Days: 30`, add a second transition to `GLACIER` at `Days: 90`, and an `Expiration` at `Days: 365`.
4. Apply the full `lifecycle.json` from the cheatsheet (adjust the prefix).
5. Use `aws s3api get-bucket-lifecycle-configuration` to confirm it saved correctly.
6. (Optional, for a longer-running exercise) Backdate the `LastModified` conceptually by reasoning through the timing engine: if you uploaded at Monday 10:00 AM UTC with a `Days: 1` rule, work out the exact UTC timestamp at which the object logically expires. Write your answer down, then check it against the README's worked example.

**Validation:** The rejected 10-day rule confirms the 30-day floor; the applied policy round-trips correctly via `get-bucket-lifecycle-configuration`.

**Cleanup:** Remove the lifecycle configuration, empty and delete the bucket.

---

## Lab 11 — Same-Region Replication (SRR)

**Goal:** Stand up a working SRR pipeline and observe replication lag + what does/doesn't replicate.

**Steps:**
1. Create two buckets in the **same** Region: `srr-lab-source-<random>` and `srr-lab-dest-<random>`.
2. Enable versioning on both.
3. Create an IAM role for replication with `s3:GetObjectVersion` on the source and `s3:PutObject`/`s3:ReplicateObject` on the destination.
4. Apply a replication configuration on the source pointing at the destination.
5. Upload a **new** object to the source and wait ~1–2 minutes; confirm it appears in the destination.
6. Upload an object, then immediately check the destination — note the asynchronous lag.
7. Take an object that existed in the source **before** step 4 (upload one before enabling replication) and confirm it does **not** appear in the destination — proving existing objects aren't retroactively replicated.

**Validation:** New objects replicate within a couple of minutes; pre-existing objects do not, without S3 Batch Replication.

**Cleanup:** Remove the replication rule, empty and delete both buckets, delete the IAM role.

---

## Lab 12 — Log Aggregation Prefix Design & Athena-Ready Structure

**Goal:** Design a partition-friendly prefix layout and prove it queries efficiently with S3 Select (a lightweight stand-in for Athena).

**Steps:**
1. Create bucket `log-lake-lab-<random>`.
2. Upload a handful of small CSV "log" files under a partitioned prefix scheme, e.g.
   `account-id=111122223333/service=vpc-flow-logs/year=2026/month=07/day=05/log1.csv`
3. Upload a second batch under a **flat**, non-partitioned prefix (`raw-logs/log1.csv`, `raw-logs/log2.csv`, …) for comparison.
4. Run an `s3api select-object-content` query against one of the partitioned files, filtering on a column value.
5. Discuss (in your lab notes) how the partitioned layout would let Athena skip scanning irrelevant prefixes entirely, while the flat layout forces a full scan.

**Validation:** The S3 Select query returns filtered rows correctly; your notes clearly explain the partition-pruning benefit.

**Cleanup:** Empty and delete the bucket.

---

## Lab 13 — S3 Object Lock in Governance Mode

**Goal:** Prove that Governance Mode blocks ordinary deletes but allows an authorized bypass — without risking a real Compliance-Mode lock you can't undo.

**Steps:**
1. Create a **new** bucket with Object Lock enabled at creation (`--object-lock-enabled-for-bucket`).
2. Confirm versioning was auto-enabled.
3. Upload an object and apply a Governance-mode retention of a few days: `put-object-retention` with `Mode: GOVERNANCE`.
4. Try a normal delete of that specific version → expect it to be blocked.
5. Retry with `--bypass-governance-retention`, using a principal that has `s3:BypassGovernanceRetention` → succeeds.
6. Separately, apply a Legal Hold to another object, confirm deletion is blocked, then turn the hold off and confirm deletion succeeds.

**Validation:** Step 4 fails, step 5 succeeds, Legal Hold blocks/unblocks deletion exactly as toggled.

**Cleanup:** Once all retention windows lapse and holds are released, empty and delete the bucket. **Do not test Compliance Mode in a bucket you might need to delete soon — it cannot be shortened or bypassed by anyone, including root.**

---

## Lab 14 — Transfer Acceleration Benchmark

**Goal:** Measure whether Transfer Acceleration actually helps for your location and file size — don't just take the marketing claim on faith.

**Steps:**
1. Create bucket `ta-lab-<random>` in a Region geographically distant from you (e.g. if you're in Asia, pick `us-east-1`).
2. Enable Transfer Acceleration on it.
3. Create a ~200 MB test file.
4. Upload it to the **standard** endpoint, timing the transfer: `time aws s3 cp bigfile.bin s3://ta-lab-<random>/standard/bigfile.bin`
5. Upload the same file to the **accelerated** endpoint, timing it: `time aws s3 cp bigfile.bin s3://ta-lab-<random>/accelerated/bigfile.bin --endpoint-url https://ta-lab-<random>.s3-accelerate.amazonaws.com`
6. Repeat both 2–3 times and compare average durations.
7. Now repeat the whole comparison with a bucket in your **own** Region and a small (<1 MB) file — confirm the gap disappears or reverses, matching the "poor use cases" guidance.

**Validation:** Cross-region + large file shows a measurable speedup with acceleration; same-region + small file shows little to no benefit.

**Cleanup:** Disable acceleration, empty and delete the bucket.

---

## Lab 15 — S3 Select Cost/Performance Comparison

**Goal:** Quantify the difference between downloading a whole file and querying it in place.

**Steps:**
1. Create bucket `select-lab-<random>` and upload a CSV with a few thousand rows and a column like `status` (values: `OK`/`FAILED`).
2. **Baseline:** download the whole file and grep for `FAILED` locally, timing it and noting the bytes transferred.
3. **S3 Select:** run `select-object-content` with `SELECT * FROM s3object s WHERE s.status = 'FAILED'`, timing it and noting the bytes returned.
4. Compare wall-clock time and data transferred between the two approaches.
5. Try a query using `JOIN` or `GROUP BY` and confirm S3 Select rejects it — reinforcing the "no relational operations" limit.

**Validation:** S3 Select returns a smaller payload for the same logical result; the unsupported SQL construct is rejected with a clear error.

**Cleanup:** Empty and delete the bucket.

---

## Suggested Order & Time Budget

| Labs | Focus | Rough time |
|---|---|---|
| 1, 4, 6 | Security & data-protection basics | 1–1.5 hrs |
| 2, 3, 8, 9 | Cross-account + KMS depth | 1.5–2 hrs |
| 7, 14 | Performance & transfer mechanics | 1 hr |
| 5, 10, 12 | Cost optimization & lifecycle | 1–1.5 hrs |
| 11 | Replication & DR | 45 min |
| 13, 15 | Compliance & query-in-place | 1 hr |

Push your policy JSONs, Terraform files, and lab notes into this repo as you go — a `labs/` folder with your actual working configs turns this from a study guide into a genuine portfolio piece.
