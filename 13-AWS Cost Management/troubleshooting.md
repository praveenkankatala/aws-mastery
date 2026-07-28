# AWS Billing & Cost Management — Troubleshooting Guide

> Common issues, exact error messages, root causes, and fixes — organized by service.

---

## Table of Contents
1. [Access & Permissions](#1-access--permissions)
2. [Cost Explorer](#2-cost-explorer)
3. [Cost Allocation Tags](#3-cost-allocation-tags)
4. [AWS Budgets](#4-aws-budgets)
5. [CUR / Data Exports & Athena](#5-cur--data-exports--athena)
6. [QuickSight / CUDOS Dashboards](#6-quicksight--cudos-dashboards)
7. [Cost Anomaly Detection](#7-cost-anomaly-detection)
8. [Rightsizing & Compute Optimizer](#8-rightsizing--compute-optimizer)
9. [Trusted Advisor](#9-trusted-advisor)
10. [Service Quotas](#10-service-quotas)
11. [API & CLI Issues](#11-api--cli-issues)

---

## 1. Access & Permissions

### ❌ "You Need Permissions" / blank Billing console even with AdministratorAccess
- **Cause:** Root has not activated *IAM user and role access to Billing information* (off by default), OR your policy lacks billing actions.
- **Fix:** Root user → **Account** page → *IAM user and role access to Billing information* → **Activate**. Then attach `Billing` job-function policy or the relevant `aws-portal:*` / fine-grained `billing:*`, `ce:*`, `budgets:*` actions.
- **Note:** Accounts created/migrated after the 2023 billing-permissions migration use **fine-grained actions** (`billing:GetBillingData`, `payments:ListPaymentPreferences`, `ce:GetCostAndUsage`, …). Old `aws-portal:*` policies may silently stop matching — update them.

### ❌ `AccessDeniedException` on `ce:GetCostAndUsage` from a member account
- **Cause:** In an Organization, member-account access to Cost Explorer data can be restricted by payer-account preferences (Linked account access settings).
- **Fix:** Management account → Cost Management **Preferences** → enable *Linked account access*. Then grant `ce:*` IAM permissions in the member account.

---

## 2. Cost Explorer

### ❌ Cost Explorer is empty / "data is being processed"
- **Cause:** First enablement takes up to **24 hours** to backfill.
- **Fix:** Wait. Data then refreshes at least every 24h — Cost Explorer is never real-time.

### ❌ Today's / yesterday's spend looks too low
- **Cause:** Normal ingestion lag (up to ~24–48h for some services to finalize line items; some charges like data transfer post late).
- **Fix:** Treat the last 1–2 days as provisional; use month-to-date trends instead.

### ❌ Numbers in Cost Explorer ≠ numbers on the Invoice
- **Causes:**
  - Different **metric**: Unblended vs Amortized vs Blended vs Net.
  - Invoice includes **tax**; Cost Explorer charts often filtered to exclude `Tax`/`Credit`/`Refund` record types.
  - Refunds/credits applied after month close (`RefreshClosedReports` behavior in CUR).
- **Fix:** Match the metric and include/exclude the same charge types before comparing.

### ❌ Can't see hourly granularity beyond 14 days
- **Cause:** By design — hourly data is limited to the trailing 14 days and is an opt-in paid feature.
- **Fix:** For long-term hourly analysis, use **CUR** (hourly, permanent).

### ❌ Forecast missing or band is huge
- **Cause:** Not enough consistent history (new accounts), or highly volatile spend.
- **Fix:** Forecasts need stable history; the 80% band widens with volatility — that's expected behavior, not an error.

---

## 3. Cost Allocation Tags

### ❌ My tag doesn't appear in Cost Explorer's tag list
- **Causes (in order of likelihood):**
  1. Tag was never **activated** in Billing → Cost Allocation Tags.
  2. Activated < **24 hours** ago.
  3. Tag key has zero *billed* usage yet.
- **Fix:** Activate, wait 24h, generate some tagged usage. Check status: `aws ce list-cost-allocation-tags`.

### ❌ Huge "No Tag" bucket
- **Causes:**
  1. **Non-retroactive activation** — days before activation show as No Tag.
  2. **Untaggable charges** — Support fees, subscriptions, some data-transfer line items.
  3. Engineers deploying untagged resources.
- **Fixes:** Run a **tag backfill** (`aws ce start-cost-allocation-tag-backfill`, once per quarter) for #1; accept #2 as structural; for #3 enforce Tag Policies + SCP (`Deny` create actions when `aws:RequestTag/<key>` is absent) and audit with AWS Config rule `required-tags` or `resourcegroupstaggingapi`.

### ❌ Same tag appears three times (`Environment`, `environment`, `Env`)
- **Cause:** Tags are **case-sensitive**; each casing is a separate dimension.
- **Fix:** Standardize via Organizations **Tag Policies**; re-tag offenders; only the canonical key stays activated.

### ❌ SCP tag enforcement blocks Auto Scaling / service-linked creations
- **Cause:** Services creating resources on your behalf don't pass your mandatory tags.
- **Fix:** Add condition exceptions for service principals/roles (e.g., allow `autoscaling.amazonaws.com`), or enforce via IaC pipelines instead of hard SCP denies.

---

## 4. AWS Budgets

### ❌ Budget alert never fired though threshold was crossed
- **Causes:**
  1. Budgets evaluates roughly **daily** (up to 3 evaluations/day) — not real-time; a fast spike can land before evaluation.
  2. SNS topic policy doesn't allow `budgets.amazonaws.com` to publish.
  3. Email subscription never **confirmed**.
- **Fix (SNS policy):**
  ```json
  {"Sid":"AllowBudgets","Effect":"Allow",
   "Principal":{"Service":"budgets.amazonaws.com"},
   "Action":"SNS:Publish","Resource":"arn:aws:sns:us-east-1:111122223333:budget-alerts"}
  ```
  Confirm the email link; for spikes rely on **Cost Anomaly Detection**, not budgets.

### ❌ Forecasted alert seems wrong / never triggers early in month
- **Cause:** Forecast alerts need enough month-to-date signal; brand-new budgets/accounts can't forecast reliably (AWS suppresses forecast alerts until ~5 weeks of usage history exists).
- **Fix:** Keep an *actual* alert alongside every forecasted one.

### ❌ Budget Action fails with `AccessDenied`
- **Cause:** The **execution role** isn't assumable by `budgets.amazonaws.com` or lacks `iam:AttachRolePolicy` / `ec2:StopInstances` / `ssm:StartAutomationExecution` for the chosen action.
- **Fix:** Trust policy principal = `budgets.amazonaws.com`; grant only the specific action permissions; re-run via `execute-budget-action`.

### ❌ Tag filter on a budget matches nothing
- **Cause:** The tag isn't an **activated cost allocation tag**, or the filter syntax is wrong (`user:Key$Value`).
- **Fix:** Activate the tag first; verify with a Cost Explorer filter before scoping the budget.

---

## 5. CUR / Data Exports & Athena

### ❌ No files in S3 after creating the export
- **Causes:** first delivery takes up to **24 hours**; bucket policy missing service permissions; wrong bucket region typed.
- **Fix:** Verify bucket policy allows `billingreports.amazonaws.com` / `bcm-data-exports.amazonaws.com` (`s3:PutObject`, `s3:GetBucketAcl`/`GetBucketPolicy`) with `aws:SourceAccount` condition; wait a full day.

### ❌ `HIVE_BAD_DATA` / schema mismatch in Athena
- **Cause:** Mixing CSV and Parquet deliveries in one table path, or CUR schema evolved (columns appear when first used) and the Glue table is stale.
- **Fix:** One format per prefix; re-run the **Glue Crawler** after month boundaries; prefer Parquet.

### ❌ Athena: `Query has no output location`
- **Fix:** Athena → Settings → set query result location (`s3://bucket/athena-results/`), or use a workgroup with one enforced.

### ❌ Duplicate rows / double-counted costs in Athena
- **Cause:** `CREATE`-versioned reports keep every delivery snapshot; querying all of them duplicates data.
- **Fix:** Use **OVERWRITE_REPORT** versioning, or filter to the latest `assembly_id` per billing period.

### ❌ CUR shows resource IDs for some services but not others
- **Cause:** "Include resource IDs" only covers services that emit them; some line items (support, tax) never carry one.
- **Fix:** Expected; aggregate those rows by `line_item_line_item_type` instead.

### ❌ No pod/task-level rows for EKS
- **Cause:** **Split Cost Allocation Data** wasn't enabled on the export, or the EKS cluster isn't opted in to split cost allocation in Billing preferences.
- **Fix:** Enable both, then wait 24h — data is forward-only.

---

## 6. QuickSight / CUDOS Dashboards

### ❌ CloudFormation stack fails: QuickSight user/ARN invalid
- **Cause:** Wrong QuickSight principal ARN (namespace/region mismatch), or QuickSight isn't Enterprise Edition in that region.
- **Fix:** `aws quicksight list-users --aws-account-id <id> --namespace default --region <qs-identity-region>` and paste the exact ARN; confirm Enterprise subscription in the deployment region.

### ❌ Dashboard deployed but empty
- **Causes:** data export hasn't delivered yet (up to 24h); QuickSight lacks **S3/Athena permissions**; SPICE dataset refresh failed.
- **Fix:** QuickSight → Manage QuickSight → **Security & Permissions** → enable Athena + tick the specific export bucket; then Datasets → Refresh; check SPICE capacity isn't exhausted.

### ❌ `Access Denied` when QuickSight queries Athena
- **Cause:** QuickSight service role can't read the Glue catalog/S3 data or write Athena results.
- **Fix:** Re-run the Security & Permissions manage flow (it rewrites the service role policy); include the Athena results bucket too.

---

## 7. Cost Anomaly Detection

### ❌ No anomalies ever detected
- **Causes:** model still baselining (needs ~10 days of history); threshold too high; monitor scoped to a dimension with no spend.
- **Fix:** Lower the absolute-impact threshold; prefer one broad **AWS services** monitor; give it time.

### ❌ Too many false positives
- **Fix:** Submit feedback on each anomaly (*planned activity*); raise the threshold; use weekly summaries for low-stakes segments.

### ❌ Alerts not arriving via SNS
- **Fix:** Topic policy must allow `costalerts.amazonaws.com` to publish; confirm subscription.

---

## 8. Rightsizing & Compute Optimizer

### ❌ "You don't have any rightsizing recommendations"
- **Causes:**
  1. Feature not enabled in Cost Explorer **Preferences**.
  2. Resources younger than the **14-day** lookback (or <30h of metrics for Compute Optimizer).
  3. Everything is genuinely `Optimized`.
  4. Net savings would be **< $0** after commitment math — suppressed by design.
- **Fix:** Enable prefs, wait out the lookback, check Compute Optimizer directly for the finding classification.

### ❌ `OptInRequiredException` from Compute Optimizer API
- **Fix:** `aws compute-optimizer update-enrollment-status --status Active` (add `--include-member-accounts` from the management account for org view).

### ❌ Memory metrics missing from recommendations
- **Causes:** CloudWatch Agent not installed/running; agent config lacks `mem_used_percent`; metrics in a custom namespace Compute Optimizer doesn't read; memory ingestion preference off.
- **Fix:** Install agent via SSM, use the standard `CWAgent` namespace with `InstanceId` dimension, enable memory utilization in Compute Optimizer preferences; verify in CloudWatch → Metrics → CWAgent.

### ❌ Recommendation suggests Graviton but our AMIs are x86-only
- **Fix:** Set **recommendation preferences** to exclude ARM64/Graviton families (account- or org-level), or invest in multi-arch builds to capture the ~40% price-performance gain.

### ❌ Downsized instance now throttling (CPU credits)
- **Cause:** Moved onto a burstable `t`-family without watching credit balance; or lookback missed a monthly peak.
- **Fix:** Check the **Performance Risk** score before acting; enable **Enhanced Infrastructure Metrics** (3-month lookback) for cyclical workloads; prefer intra-family first.

---

## 9. Trusted Advisor

### ❌ Cost Optimization category shows "Upgrade your support plan"
- **Cause:** Basic/Developer support only exposes Service Quotas + core Security checks.
- **Fix:** Business/Enterprise Support unlocks the 100+ check suite and the `support`/`trustedadvisor` APIs.

### ❌ `SubscriptionRequiredException` calling `aws support ...`
- **Cause:** Same support-tier gating; or wrong region.
- **Fix:** Business+ plan; always call with `--region us-east-1`.

### ❌ Check data is stale
- **Cause:** Cost checks refresh on a ~**weekly** cycle automatically.
- **Fix:** Manual refresh: `aws support refresh-trusted-advisor-check --check-id <id>` (some checks are refresh-rate-limited; wait for `describe-trusted-advisor-check-refresh-statuses` to show `success`).

### ❌ EventBridge rule never triggers
- **Causes:** Rule created outside **us-east-1**; event pattern's `check-name` doesn't exactly match; status filter excludes the transition.
- **Fix:** Recreate rule in us-east-1; copy check names verbatim from `describe-trusted-advisor-checks`; include both `WARN` and `ERROR`.

---

## 10. Service Quotas

### ❌ Deployment fails: `You have exceeded your maximum vCPU limit` / `LimitExceededException` / `Throttling: Rate exceeded`
- **Cause:** Applied quota hit (resource count or API rate).
- **Fix:** For resource quotas, request an increase (`request-service-quota-increase`); for API throttling, implement **exponential backoff + jitter** (SDKs do this—tune retries) and spread bursts.

### ❌ Quota shows "Not available" utilization
- **Cause:** Not every quota is CloudWatch-instrumented; utilization graphs exist only for supported quotas.
- **Fix:** Track those manually (e.g., count resources on a schedule) or rely on failure alarms.

### ❌ Increase request stuck in `PENDING` or `CASE_OPENED`
- **Cause:** Larger increases route to human review via a support case.
- **Fix:** Watch `list-requested-service-quota-change-history`; respond to the support case; justify with usage data. Non-adjustable quotas will be auto-denied — check `Adjustable: false` first.

### ❌ Quota fine in us-east-1 but deployment fails in DR region
- **Cause:** Quotas are **per-region**.
- **Fix:** Request in every region you operate; bake increases into the **org quota request template** so new accounts inherit them.

---

## 11. API & CLI Issues

### ❌ `Could not connect to the endpoint URL: "https://ce.<region>.amazonaws.com"`
- **Cause:** Billing APIs (`ce`, `cur`, `budgets`, `support`) are global-but-hosted-in-us-east-1.
- **Fix:** Add `--region us-east-1`.

### ❌ Unexpected Cost Explorer API charges on the bill
- **Cause:** `ce` calls cost **$0.01 per paginated request**; dashboards polling frequently add up.
- **Fix:** Cache responses; reduce polling; move heavy analytics to CUR + Athena (pay only Athena scan costs — and partition/compress to minimize those).

### ❌ `ValidationException: Start date must be before end date` / date confusion
- **Cause:** `End` is **exclusive** in `ce` time periods.
- **Fix:** For July, use `Start=2026-07-01,End=2026-08-01`.

### ❌ CLI JSON filter errors (`Invalid type for parameter Filter`)
- **Cause:** Shell quoting mangling the JSON.
- **Fix:** Use single-quoted JSON blobs (Linux/macOS) or `file://filter.json` — the most reliable option in scripts and on Windows.

---

## Quick Reference — "Why is my data missing?" cheat table

| Symptom | Most likely cause | Wait time |
|---|---|---|
| Cost Explorer empty | Just enabled | up to 24h |
| Tag missing in CE | Not activated / just activated | up to 24h |
| Pre-activation days = "No Tag" | Non-retroactive tags | backfill (quarterly) |
| CUR bucket empty | First delivery pending | up to 24h |
| CUDOS dashboard blank | Export + SPICE not loaded | up to 24h |
| No rightsizing recs | Lookback not elapsed | 14 days |
| Compute Optimizer "no data" | <30h metrics / not opted in | 30h+ |
| Anomaly detection silent | Model baselining | ~10 days |
| Trusted Advisor stale | Weekly refresh cycle | ≤7 days or manual refresh |
