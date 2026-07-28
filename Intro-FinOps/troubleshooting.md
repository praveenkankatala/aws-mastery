# AWS Billing & Cost Management — Troubleshooting Guide

> Common issues organized by area. Each entry: **Symptom → Root cause → Fix**.

---

## Table of Contents

1. [Access & IAM Issues](#1-access--iam-issues)
2. [Cost Explorer Issues](#2-cost-explorer-issues)
3. [Tags & Cost Categories Issues](#3-tags--cost-categories-issues)
4. [Budgets & Alerts Issues](#4-budgets--alerts-issues)
5. [Budget Actions Issues](#5-budget-actions-issues)
6. [Cost Anomaly Detection Issues](#6-cost-anomaly-detection-issues)
7. [CUR / Data Exports & Athena Issues](#7-cur--data-exports--athena-issues)
8. [Organizations / Consolidated Billing Issues](#8-organizations--consolidated-billing-issues)
9. [Savings Plans & RI Issues](#9-savings-plans--ri-issues)
10. [Unexpected Charges — Diagnosis Playbook](#10-unexpected-charges--diagnosis-playbook)

---

## 1. Access & IAM Issues

### 1.1 "You need permissions" on Billing pages — even as an admin

- **Symptom:** IAM user with `AdministratorAccess` sees *"You need permissions to access this page"* or blank billing pages.
- **Root cause:** Root has not enabled **IAM user and role access to Billing information**. This account-level switch is separate from IAM policies, and it is **off by default**.
- **Fix:** Sign in as **root** → Account page → *IAM user and role access to Billing information* → Edit → **Activate IAM Access** → Update. Then confirm the identity also has billing permissions (`Billing` job-function policy or equivalent).

### 1.2 `AccessDeniedException` on `aws ce get-cost-and-usage`

- **Root cause:** Missing `ce:GetCostAndUsage` (and friends) in the identity's policy — `Billing` job-function policy covers console but check for explicit denies; SCPs can also block `ce:*`.
- **Fix:** Attach a policy with `ce:Get*`, `ce:List*`, `ce:Describe*`. In an org, verify no SCP denies Cost Explorer actions in your account/OU. Also make sure you're calling **us-east-1**.

### 1.3 `aws-portal:*` policies stopped working / migration confusion

- **Symptom:** Old policies using `aws-portal:ViewBilling` behave unexpectedly; new consoles reference `billing:*`, `payments:*`, `account:*` actions.
- **Root cause:** AWS replaced coarse `aws-portal` actions with **fine-grained billing actions**. Accounts created after the migration only honor the new actions.
- **Fix:** Update policies to the fine-grained actions (`billing:GetBillingData`, `invoicing:GetInvoicePDF`, `payments:ListPaymentPreferences`, `ce:*`, `budgets:*`, `cur:*`, `freetier:*`, `tax:*` as needed). AWS's affected-policies tool / managed policies (`AWSBillingReadOnlyAccess`) already use them.

### 1.4 Member account can't see its own costs

- **Root cause:** In an organization, the **management account can restrict** member access to billing views; also member IAM access activation is per-account.
- **Fix:** Activate IAM billing access **in the member account** (its own root), grant IAM permissions there, and check payer-level settings/SCPs.

---

## 2. Cost Explorer Issues

### 2.1 Cost Explorer is empty / "being prepared"

- **Root cause:** First enablement takes up to **24 hours** to ingest data.
- **Fix:** Wait. Also note new accounts have little data to show.

### 2.2 Today's spend is missing or wrong

- **Root cause:** Cost Explorer refreshes **at least once per 24 hours** — it is *not* real time; some services report usage with additional delay (data transfer, support fees, marketplace).
- **Fix:** Treat intra-day numbers as estimates; verify after 24–48h. For close-to-real-time spike detection use **Cost Anomaly Detection** (evaluates ~3x/day).

### 2.3 Numbers don't match the Bills page / invoice

- **Root causes (pick your mismatch):**
  - Comparing **unblended vs amortized vs blended** cost types
  - Cost Explorer default excludes/includes **credits, refunds, taxes** differently than filters you set
  - Bills finalize after month close — mid-month restatements are normal
  - Org view (payer) vs single-account view
- **Fix:** For invoice reconciliation use **Unblended** with refunds/credits included and a **closed** month. For trend analysis, use Amortized.

### 2.4 Can't see resource-level (instance ID) costs

- **Root cause:** Hourly & resource-level granularity is an **opt-in paid feature**, and only retains **14 days**.
- **Fix:** Cost Management → Preferences → enable hourly/resource granularity, or use **CUR with resource IDs** for unlimited history.

### 2.5 Can only see 13 months of history

- **Root cause:** Default retention is 13 months.
- **Fix:** Enable **multi-year data** (up to 38 months, monthly granularity) in Cost Management preferences; for longer history, keep CUR in S3 forever.

---

## 3. Tags & Cost Categories Issues

### 3.1 Activated tag doesn't appear in Cost Explorer / CUR

- **Root causes:**
  1. **< 24h** since activation
  2. Tag applied to resources **after** usage occurred — cost allocation is **not retroactive**
  3. Tag activated in a **member** account — activation must happen in the **management (payer) account**
  4. The resource type doesn't propagate tags to billing (a few don't)
- **Fix:** Wait 24h; activate at the payer; use **tag backfill** (`aws ce start-cost-allocation-tag-backfill`) to reprocess up to 12 months of history; verify the tag exists on the resource (`resourcegroupstaggingapi get-resources`).

### 3.2 Large "No tag key: X" / untagged bucket in reports

- **Root cause:** Untagged resources, un-taggable charges (taxes, support, some data transfer), or tags added late.
- **Fix:** Tag enforcement: **Tag Policies** + IaC checks + AWS Config rule `required-tags`; use a **Cost Category** default value `Unallocated` and drive it down as a KPI; use `aws:createdBy` to find who created untagged resources.

### 3.3 Tag values case mismatch splits my data (`Prod` vs `prod`)

- **Root cause:** Tags are **case-sensitive** in cost data.
- **Fix:** Standardize (Tag Policies can enforce case), re-tag resources; use a Cost Category rule matching both values to merge historical reporting.

### 3.4 Cost category shows costs in the wrong bucket

- **Root cause:** Rules evaluate **in order — first match wins**; a broad early rule swallows everything.
- **Fix:** Reorder rules from most-specific to least-specific; remember category changes apply to the **current month onward** (retroactive to the start of the month).

---

## 4. Budgets & Alerts Issues

### 4.1 Budget alert never fired though I crossed the threshold

- **Root causes:**
  1. Budget data refreshes only **~3 times a day** — alerts aren't instant
  2. Alert set on **Forecasted** but the budget period is nearly over (forecast alerts need enough of the period remaining/history to forecast)
  3. Notification threshold/comparison misconfigured
  4. Email went to spam / SNS subscription **never confirmed**
- **Fix:** Verify with `describe-budgets` (check `CalculatedSpend`); use **Actual** thresholds for testing; confirm SNS subscriptions (`aws sns list-subscriptions-by-topic` — `PendingConfirmation` = not confirmed); check spam.

### 4.2 SNS notification not delivered though email works

- **Root cause:** The SNS **topic access policy** doesn't allow `budgets.amazonaws.com` to publish. The console shows a validation error, but CLI-created budgets silently fail delivery.
- **Fix:** Add to the topic policy:

```json
{
  "Sid": "AllowBudgets",
  "Effect": "Allow",
  "Principal": {"Service": "budgets.amazonaws.com"},
  "Action": "SNS:Publish",
  "Resource": "arn:aws:sns:us-east-1:<ACCOUNT_ID>:budget-alerts"
}
```

For Anomaly Detection, the principal is `costalerts.amazonaws.com`. If the topic is KMS-encrypted, the key policy must also allow the service (`kms:GenerateDataKey`, `kms:Decrypt`).

### 4.3 Tag filter on a budget matches nothing

- **Root causes:** Tag not **activated** for cost allocation; wrong CostFilters syntax (`user:Key$Value`); usage predates tagging.
- **Fix:** Activate the tag first (wait 24h), then create the budget; verify the tag shows values in `aws ce get-tags`.

### 4.4 "Budget limit exceeded" — can't create more budgets

- **Root cause:** Account budget quota reached (soft limit).
- **Fix:** Delete unused budgets or request a quota increase (Service Quotas / support).

### 4.5 Budget shows $0 spend but resources are running

- **Root cause:** Filters too narrow (wrong service name string, wrong account); or the ~8–12h data lag.
- **Fix:** Get exact dimension values from `aws ce get-dimension-values --dimension SERVICE` — filter strings must match exactly (e.g., `Amazon Elastic Compute Cloud - Compute`, not `EC2`).

---

## 5. Budget Actions Issues

### 5.1 Action stuck in `STANDBY` / never executes

- **Root cause:** Threshold not reached yet (remember 3x/day evaluation), or approval mode is **Require approval** and nobody approved.
- **Fix:** Check `describe-budget-action-histories`; approve pending executions with `execute-budget-action --execution-type APPROVE_BUDGET_ACTION`.

### 5.2 Action fails with `EXECUTION_FAILURE`

- **Root causes:**
  1. Execution role missing trust for `budgets.amazonaws.com` or missing permissions (`ec2:StopInstances`, `iam:AttachUserPolicy`, `organizations:AttachPolicy`…)
  2. Target instance IDs no longer exist / wrong region
  3. SCP blocks the action's API calls
- **Fix:** Attach `AWSBudgetsActionsWithAWSResourceControlAccess` to the role; verify the trust policy; re-select live targets.

### 5.3 IAM-policy action didn't stop new spending

- **Root cause:** The deny policy attached only to the selected users/roles/groups — other identities (or already-running resources) keep spending. Stopping *new* launches doesn't stop *existing* resources.
- **Fix:** Combine actions: apply deny policy **and** stop EC2/RDS action; for hard org-level stops use an SCP action.

---

## 6. Cost Anomaly Detection Issues

### 6.1 No anomalies detected despite obvious spikes

- **Root causes:**
  1. Monitor created recently — needs **~10 days** of history to learn a baseline
  2. Spike is below your **alert threshold** (it may be detected but not alerted — check Detection history)
  3. Gradual growth isn't an "anomaly" — the model tracks pattern deviations, not slow creep
- **Fix:** Check Detection history (detected ≠ alerted); lower the threshold; for slow creep use forecast budgets instead.

### 6.2 Too many false-positive alerts

- **Fix:** Raise the subscription threshold ($ or %), give **feedback** on each anomaly (`provide-anomaly-feedback`) so the model tunes, and split monitors (per-service monitors give tighter baselines than one broad custom monitor).

### 6.3 Anomaly SNS alerts not arriving

- **Root cause:** Topic policy missing `costalerts.amazonaws.com` publish permission (see §4.2); or subscription frequency is DAILY/WEEKLY (summary emails only) while you expected instant SNS — **IMMEDIATE frequency is required for SNS**.
- **Fix:** Fix the topic policy; set Frequency = Individual alerts/IMMEDIATE for SNS delivery.

---

## 7. CUR / Data Exports & Athena Issues

### 7.1 No report files in S3 after creating the export

- **Root causes:**
  1. First delivery takes **up to 24h**
  2. **Bucket policy** missing/edited — `billingreports.amazonaws.com` / `bcm-data-exports.amazonaws.com` can't `PutObject`
  3. Bucket has SSE-KMS with a key policy that blocks the billing service
  4. Report created in a member account instead of the payer (member CURs only contain that member's data — which may be why it "looks empty")
- **Fix:** Recreate via console so the policy is auto-applied, or paste the documented bucket policy; check KMS key policy; wait 24h.

### 7.2 Athena: `HIVE_BAD_DATA` / zero rows / schema mismatch

- **Root causes:**
  1. Table `LOCATION` points at the wrong prefix (must be the `.../data/` folder for CUR 2.0, not metadata folders)
  2. DDL columns don't match the columns actually selected in the export
  3. CSV export queried with a Parquet SerDe (or vice-versa)
  4. Crawler picked up metadata/manifest files as tables
- **Fix:** Align LOCATION and format; exclude non-data prefixes in the crawler; simplest: delete and let a fresh Glue crawler infer from `data/` only.

### 7.3 Numbers change after I already reported them

- **Root cause:** CUR is **restated multiple times** during the month and finalized after invoice issuance (`RefreshClosedReports`/refresh cadence also re-delivers prior months when credits/refunds land).
- **Fix:** Treat mid-month data as provisional; snapshot finalized months for reporting; use `bill_invoice_id` presence to detect finalized data.

### 7.4 Athena query costs creeping up

- **Fix:** Use **Parquet** (columnar, compressed) not CSV; SELECT only needed columns; partition-prune by `billing_period`; set workgroup query limits.

### 7.5 Legacy CUR vs Data Exports confusion

- **Symptom:** Docs mention `aws cur`, console shows "Data Exports".
- **Explanation:** **Data Exports (CUR 2.0)** is the successor — SQL column selection, nested resource tags map, better schema stability. Legacy CUR still works but new builds should use CUR 2.0. The Athena/QuickSight ecosystem (CUDOS) supports both.

---

## 8. Organizations / Consolidated Billing Issues

### 8.1 Member account shows $0 in its own Cost Explorer

- **Root cause:** Historical data before joining the org stays with the old payer; also check payer-side visibility settings.
- **Explanation:** When an account joins/leaves an organization, cost history **doesn't move** with it in the new payer's view.

### 8.2 RI/SP discount applied to the "wrong" account

- **Root cause:** **Discount sharing** is on by default — unused RI/SP discounts float to any matching usage in the org, and where they land is not controllable per-hour.
- **Fix:** This is usually *good* (maximizes utilization). To restrict: Billing preferences → turn **off** RI/SP discount sharing for specific accounts — accepting possibly lower overall utilization. For clean chargeback despite sharing, report on **amortized + effective savings** columns in CUR, or use Billing Conductor.

### 8.3 Tag activation missing in member accounts

- **Root cause:** Cost allocation tags are **activated by the management account only**.
- **Fix:** Activate at the payer; member-created tag *keys* appear in the payer's activation list once used.

### 8.4 Can't remove/close a member account

- **Root causes:** Member lacks standalone billing info (payment method) to leave; or waiting periods for newly created accounts; org deletion requires zero members.
- **Fix:** Add payment info to the member before removal, or **close** the account from the management account (quota: limited % of accounts closable per month); then delete the org.

### 8.5 SCP didn't block the API call

- **Root causes:** SCPs **don't apply to the management account**; policy attached to the wrong OU; `FullAWSAccess` interplay misunderstood (SCPs are filters, deny wins).
- **Fix:** Test from a member account; check effective policies with the Organizations console policy viewer.

---

## 9. Savings Plans & RI Issues

### 9.1 Savings Plan utilization below 100%

- **Root cause:** You committed to more $/hour than your eligible usage in some hours (nights/weekends dips, workload moved/terminated, moved to non-eligible services/regions for EC2 Instance SPs).
- **Fix:** Utilization is hourly — investigate with SP utilization report at daily granularity; shift eligible workloads onto covered hours; next time commit to the **trough**, not the average; set a utilization budget alert (<90%).

### 9.2 On-Demand charges despite owning an SP/RI

- **Root causes:**
  1. Usage exceeds the commitment (SP covers first $X/hour; excess is On-Demand — expected)
  2. RI attribute mismatch: region/AZ, OS/platform, tenancy, **size-flexibility only applies to regional Linux/shared-tenancy RIs**
  3. EC2 Instance SP scoped to a different family/region than the running instances
  4. In an org: discount consumed by another account (sharing)
- **Fix:** Check coverage reports grouped by instance family/region; align instance attributes; consider Convertible RI exchange or additional Compute SP.

### 9.3 Bought the wrong RI / over-committed

- **Fixes by type:**
  - **Convertible RI** → `get-reserved-instances-exchange-quote` + exchange to what you need (equal or greater value)
  - **Standard RI** → modify (AZ/scope/size within family for Linux) or **sell on the RI Marketplace** (US bank account required)
  - **Savings Plan** → 7-day return window for qualifying plans (`return-savings-plan`); otherwise it rides to term — grow usage into it
  - Genuine purchase mistakes within a short window: open an AWS Support billing case

### 9.4 Recommendation seems too high/low

- **Root cause:** Lookback window (7/30/60 days) captured an unrepresentative period (spike, migration, holidays).
- **Fix:** Use 60-day lookback for stability; exclude known-temporary workloads mentally; buy incrementally (multiple smaller SPs quarterly rather than one big one).

---

## 10. Unexpected Charges — Diagnosis Playbook

### 10.1 "I got billed but I thought I was in free tier"

Most common culprits:

| Charge | Why |
|--------|-----|
| **EBS storage / snapshots** | Free tier covers limited GB; volumes bill even when the instance is **stopped** |
| **Elastic IP** | Billed when **not associated** with a running instance |
| **NAT Gateway** | ~$0.045/hr + per-GB processing — never free, frequently forgotten |
| **Non-free instance type** | Only specific micro types are free-tier eligible (t2.micro/t3.micro depending on region) |
| **Two micro instances** | 750 hrs/month is shared across instances (2 × 24×31 > 750) |
| **RDS left running** | Free tier is 750 hrs of specific classes; Multi-AZ doubles hours |
| **Data transfer out** | Free allowance is small; inter-AZ/region traffic also bills |
| **Route 53 hosted zone** | $0.50/zone/month — no free tier |
| **Support plan** | Developer/Business subscription auto-renews monthly |
| **CloudWatch** | Custom metrics/dashboards/log ingestion beyond free allowance |

**Playbook:** Bills page → expand each service → note usage type → Cost Explorer daily view filtered to that service → find start date → locate & delete the resource (check **all regions** — use the region selector or `aws resourcegroupstaggingapi get-resources` per region). For genuinely accidental first-time charges, AWS Support sometimes issues goodwill credits — ask politely via a billing support case (free to open for all accounts).

### 10.2 "Charges from a region I never use"

- **Likely causes:** default-region resources created by a tool/tutorial; a global service billing in us-east-1 (CloudFront, Route 53, some certs); **or compromised credentials** (crypto-mining typically appears as large EC2/GPU usage in unusual regions).
- **Fix:** If unexplained big compute in strange regions → **assume compromise**: rotate/deactivate access keys, check CloudTrail `RunInstances`/`CreateUser` events, delete rogue resources in every region, enable MFA, open an AWS Support security case. Then set up anomaly detection (Lab 6) so it never goes unnoticed again.

### 10.3 "Bill doubled this month" — structured drill-down

1. **Cost Explorer** → monthly → group by **Service** → which service moved?
2. Re-group by **Usage type** → compute vs storage vs transfer?
3. Re-group by **Linked account** (org) / **Region** / **Tag: Team**
4. Switch to **daily** granularity → exact start date of the change
5. **Anomaly Detection history** → ready-made root cause?
6. **CUR/Athena** → `line_item_resource_id` level → the exact resource
7. Correlate the start date with deployments (CloudTrail, CI/CD history)

### 10.4 "Charged after closing my account"

- **Root cause:** Closure doesn't always immediately terminate everything; final invoice covers usage up to closure; some subscriptions (RIs, support) bill through their cycle. Accounts are also reopenable for ~90 days, during which resources may persist.
- **Fix:** Before closing: terminate resources in **all regions**, delete RIs/SP obligations understanding they bill to term, cancel support plan, remove marketplace subscriptions. Post-closure charges → billing support case.

---

## 🧯 Golden Rules

1. **Root activates IAM billing access first** — half of all "billing broken" reports are this.
2. **All billing CLIs → us-east-1.**
3. **Nothing is real-time**: Cost Explorer ≤24h lag, Budgets 3x/day, CUR restated all month.
4. **Tags: activate at the payer, wait 24h, never retroactive without backfill.**
5. **SNS topics need service-principal publish permissions** (`budgets.amazonaws.com`, `costalerts.amazonaws.com`).
6. **Filters need exact dimension strings** — always look them up with `get-dimension-values`.
7. When in doubt about a charge: **billing support cases are free for every account.**
