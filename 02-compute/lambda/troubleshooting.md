# 🛠️ Troubleshooting Guide

Common failure modes for both halves of this repo, with causes and fixes. Cross-references [`commands-cheatsheet.md`](./commands-cheatsheet.md) for diagnostic commands.

---

## Lambda Architecture (Data Pipeline) Issues

### 1. Batch view and speed view totals don't add up / double counting
**Symptom:** `Unified_Total_Sales` in the serving layer is higher than expected.
**Cause:** Records that were already absorbed into the batch layer are still sitting in the real-time stream and getting counted twice.
**Fix:** Once a batch cycle completes, purge (or mark as processed) any stream records with a timestamp earlier than the batch's cutoff before running the speed layer again.

### 2. Serving layer returns stale results
**Symptom:** Dashboard doesn't reflect recent batch updates.
**Cause:** The batch view cache/index wasn't refreshed after the last batch run completed.
**Fix:** Ensure the serving layer re-indexes the batch view immediately after each batch cycle finishes, not on a fixed unrelated schedule.

### 3. Master dataset growing unbounded
**Symptom:** Batch layer runs take longer every cycle; storage costs climbing.
**Cause:** Immutability means nothing is ever deleted — the raw log just keeps growing.
**Fix:** Partition the master dataset (e.g., by date), and archive/cold-tier older partitions that are no longer needed for recomputation, rather than deleting them outright.

### 4. Users confused by inconsistent numbers between refreshes
**Symptom:** Same query returns slightly different totals seconds apart.
**Cause:** This is expected — it's the *eventual consistency* trade-off of the speed layer.
**Fix:** This isn't really a bug, but UX-wise, show a "last updated" / "real-time estimate" indicator so users understand the number is a live approximation until the next batch cycle corrects it.

---

## AWS Lambda Service Issues

### 5. High cold start latency
**Symptom:** First invocation after idle time is consistently slow (seconds).
**Causes & Fixes:**
- Large deployment package → trim dependencies, use Layers, or switch to container images only if genuinely needed.
- Heavy initialization code running inside the handler → move DB/SDK client setup outside the handler (see README → Cold Start Best Practices).
- Historically slow-starting runtime (Java, .NET, larger Python apps) → enable **SnapStart**.
- Latency-sensitive, spiky traffic → use **Provisioned Concurrency** to keep instances warm.

### 6. `Task timed out after N seconds`
**Symptom:** Function is killed mid-execution.
**Cause:** `Timeout` config is lower than actual processing time needed.
**Fix:** Increase `Timeout` (max 15 min) via `update-function-configuration`, or profile and optimize the code path that's slow (e.g., an unindexed DB query).

### 7. `Process exited before completing request` / Out of Memory
**Symptom:** Function crashes partway through, especially on larger payloads.
**Cause:** `MemorySize` too low for the workload (remember: memory also scales CPU proportionally).
**Fix:** Increase `MemorySize`. Counter-intuitively, this can also *reduce* cost if the function is CPU-bound, since it finishes faster.

### 8. `TooManyRequestsException` / throttling
**Symptom:** Some invocations fail under load with a throttle error.
**Causes & Fixes:**
- Reserved concurrency set too low for real traffic → raise it, or remove it if it's not actually needed for protection.
- Account-wide concurrency limit reached → check with `aws lambda get-account-settings` and request a limit increase if needed.

### 9. VPC-attached Lambda can't reach the internet / times out
**Symptom:** Function works fine without a VPC, but times out once `VpcConfig` is added.
**Cause:** Placing a Lambda in private subnets cuts off default internet access.
**Fix:** Add a NAT Gateway for internet-bound calls, or use VPC Endpoints for AWS service calls (S3, DynamoDB, etc.) to avoid the extra hop entirely.

### 10. `AccessDeniedException` calling a downstream service
**Symptom:** Function runs but fails when calling S3/DynamoDB/another service.
**Cause:** The execution role's IAM policy doesn't grant the required action, or the target resource's own policy doesn't trust the Lambda's role.
**Fix:** Check both sides — the Lambda execution role (`get-policy`-adjacent IAM check) and, for cross-account/service triggers, the resource-based policy (`aws lambda get-policy`).

### 11. Deployment package too large
**Symptom:** `sam deploy` or `create-function` fails on package size.
**Cause:** Direct zip uploads are capped (250 MB unzipped via console/CLI; larger via container images).
**Fix:** Move shared/heavy dependencies into a **Layer**, strip unused files, or package as a container image if you genuinely need very large dependencies (e.g. big ML libraries).

### 12. Environment variable not found at runtime
**Symptom:** `KeyError` / `None` reading a config value that's clearly set in the console.
**Cause:** Old deployed version/alias is still being invoked, or a typo in the variable name (env vars are case-sensitive).
**Fix:** Re-check the exact key name, confirm you're invoking the alias/version you think you are, and redeploy if needed.

### 13. SnapStart returns stale connections after snapshot restore
**Symptom:** Intermittent DB connection errors only in production, only after idle periods, only with SnapStart enabled.
**Cause:** Connections established during the *original* init are frozen into the snapshot and may be stale when thawed later.
**Fix:** Use SnapStart's runtime hooks (`beforeCheckpoint` / `afterRestore`) to re-establish connections after restore, rather than assuming init-time connections are still valid.

---

## Diagnostic Commands Quick Reference

For the actual CLI syntax behind these checks, see [`commands-cheatsheet.md`](./commands-cheatsheet.md):
- **Logs & Init Duration** → Section 8 (Logs & Monitoring)
- **Concurrency limits** → Section 3 (Concurrency Controls)
- **Config verification** → Section 1 (`get-function-configuration`)
- **Permissions** → Section 6 (`get-policy`)
