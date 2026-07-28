# AWS Billing & Cost Management — Hands-On Labs

> 12 progressive labs taking you from a fresh account to a complete FinOps setup. Labs 1–6 work in **any single account** (mostly free). Lab 7 needs an **AWS Organization**. Labs 8–11 add reporting, optimization, and automation.
>
> 💰 **Cost of these labs:** near zero if you stay in free tier — the only paid items are S3 storage for CUR (cents), Athena queries (cents), any test EC2 instances you launch, and Cost Explorer API calls ($0.01/request). **Lab 12 tears everything down.**

---

## Lab Index

| # | Lab | Skills |
|---|-----|--------|
| 1 | Account & IAM foundation for billing | IAM activation, billing policies, preferences |
| 2 | Tagging strategy, cost allocation tags & cost categories | Tags, categories, split charges |
| 3 | Cost Explorer deep-dive | Filters, group-bys, cost types, forecast |
| 4 | Budgets with email + SNS (Slack-ready) alerts | Budgets, SNS |
| 5 | Budget Actions — auto-stop EC2 on overspend | Budget actions, IAM roles |
| 6 | Cost Anomaly Detection | ML monitors, subscriptions |
| 7 | Consolidated billing with AWS Organizations | Org, OUs, SCP cost guardrails |
| 8 | CUR 2.0 → S3 → Athena → dashboard | Data Exports, Glue, Athena |
| 9 | Waste hunt: Trusted Advisor, Compute Optimizer, Cost Optimization Hub | Rightsizing |
| 10 | Savings Plans analysis & (simulated) purchase | SP recommendations, utilization budgets |
| 11 | Automation: scheduled shutdown of dev resources | EventBridge, Lambda |
| 12 | Cleanup | Teardown |

---

## Lab 1 — Account & IAM Foundation for Billing

**Goal:** stop using root, give an IAM identity full billing visibility, and switch on the basic alerting preferences.

### Step 1.1 — Activate IAM access to billing (root, one time)

1. Sign in as the **root user**
2. Top-right account menu → **Account**
3. Scroll to **IAM user and role access to Billing information**
4. **Edit** → check **Activate IAM Access** → **Update**

> Without this, billing pages show *"You need permissions"* to every IAM user — even admins. This is the single most common billing gotcha.

### Step 1.2 — Create a FinOps IAM role/user

```bash
# Create a group and attach the AWS-managed Billing job-function policy
aws iam create-group --group-name FinOps
aws iam attach-group-policy --group-name FinOps \
  --policy-arn arn:aws:iam::aws:policy/job-function/Billing
# Add Cost Explorer / CUR / budgets read-write as needed
aws iam attach-group-policy --group-name FinOps \
  --policy-arn arn:aws:iam::aws:policy/AWSBillingReadOnlyAccess
```

For a **read-only analyst**, use `AWSBillingReadOnlyAccess` alone. For engineers, a custom policy with `ce:Get*`, `ce:Describe*`, `ce:List*`, `budgets:View*` is typical.

### Step 1.3 — Billing preferences

1. Billing console → **Billing preferences**
2. Enable **PDF invoices delivered by email**
3. Enable **AWS Free Tier alerts** (add an email)
4. Enable **Receive CloudWatch billing alerts** (needed later for Lab 4's optional legacy alarm) → **Save**

### Step 1.4 — Explore the Bills page

Billing console → **Bills** → expand the current month:
- Note the hierarchy: **Service → Region → Usage type**
- Find **Taxes**, **Credits**, and (if org) the per-account tabs
- Compare with **Invoices** for a closed month

✅ **Checkpoint:** an IAM identity (not root) can open Bills, Cost Explorer, Budgets without permission errors.

---

## Lab 2 — Tagging Strategy, Cost Allocation Tags & Cost Categories

**Goal:** make spend attributable.

### Step 2.1 — Define a minimal tag standard

| Key | Example values | Purpose |
|-----|----------------|---------|
| `Environment` | dev / qa / prod | Env split |
| `Team` | platform / payments | Chargeback owner |
| `Project` | phoenix | Product mapping |
| `AutoStop` | true / false | Automation hook (Lab 11) |

### Step 2.2 — Launch two tagged test resources

```bash
aws ec2 run-instances --image-id $(aws ssm get-parameter \
    --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
    --query Parameter.Value --output text) \
  --instance-type t3.micro --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Environment,Value=dev},{Key=Team,Value=platform},{Key=AutoStop,Value=true}]'

aws s3 mb s3://finops-lab-$RANDOM-demo
aws s3api put-bucket-tagging --bucket <bucket-name> \
  --tagging 'TagSet=[{Key=Environment,Value=dev},{Key=Team,Value=payments}]'
```

### Step 2.3 — Activate the tags for cost allocation

1. Billing console → **Cost allocation tags**
2. **User-defined** tab → select `Environment`, `Team`, `Project` → **Activate**
3. **AWS-generated** tab → activate `aws:createdBy` (optional but useful)

```bash
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Environment,Status=Active TagKey=Team,Status=Active
```

> ⏳ Tags take up to **24h** to appear in Cost Explorer/CUR and only apply to usage **after** activation. Use the **backfill** feature to reprocess past months: `aws ce start-cost-allocation-tag-backfill --backfill-from 2026-01-01T00:00:00Z`

### Step 2.4 — Create a Cost Category

1. Cost Management → **Cost categories** → **Create**
2. Name: `Teams`; rules (ordered, first match wins):
   - `Platform` ⇐ Tag `Team = platform`
   - `Payments` ⇐ Tag `Team = payments`
   - Default value: `Unallocated`
3. (Optional) **Split charges:** send `Shared` category costs to Platform/Payments **proportionally**

✅ **Checkpoint (next day):** Cost Explorer → Group by → *Cost category: Teams* shows your split, and untagged spend lands in `Unallocated`.

---

## Lab 3 — Cost Explorer Deep-Dive

**Goal:** be able to answer "why did cost change?" in under 5 minutes.

### Step 3.1 — Enable

Billing console → **Cost Explorer** → launch. First-time enablement processes ~24h of data before charts appear.

### Step 3.2 — Core drills (do each one)

1. **Monthly trend:** Date = last 6 months, granularity = Monthly, Group by = **Service** → identify your top 3 services
2. **Daily spike hunt:** last 14 days, Daily, filter Service = EC2, Group by = **Usage type** → note that `BoxUsage` (compute), `EBS:VolumeUsage`, and `DataTransfer` are separate lines
3. **Tag view:** Group by = Tag: `Environment` (after Lab 2 activates)
4. **Purchase option:** Group by = **Purchase option** → On-Demand vs Spot vs SP/RI split
5. **Cost type switch:** toggle Unblended ↔ **Amortized** and observe RI/SP upfronts smoothing out
6. **Forecast:** extend the date range into the future → shaded forecast band (±)

### Step 3.3 — Save a report

Save the daily-EC2-by-usage-type view as a **report** — saved reports are your reusable dashboards.

### Step 3.4 — Same answers via CLI

```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query "ResultsByTime[0].Groups[?Metrics.UnblendedCost.Amount>'0'].[Keys[0],Metrics.UnblendedCost.Amount]" \
  --output table
```

✅ **Checkpoint:** you can name your top 3 services, the top EC2 usage type, and the month-end forecast.

---

## Lab 4 — Budgets with Email + SNS Alerts

**Goal:** never be surprised by a bill again.

### Step 4.1 — SNS topic for alerts

```bash
aws sns create-topic --name budget-alerts --region us-east-1
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:$ACCOUNT_ID:budget-alerts \
  --protocol email --notification-endpoint you@example.com --region us-east-1
# Confirm the subscription from your inbox!
```

**Allow Budgets to publish** (critical — see troubleshooting §4):

```bash
aws sns set-topic-attributes --region us-east-1 \
  --topic-arn arn:aws:sns:us-east-1:$ACCOUNT_ID:budget-alerts \
  --attribute-name Policy \
  --attribute-value '{
    "Version":"2012-10-17",
    "Statement":[{"Sid":"AllowBudgets","Effect":"Allow",
      "Principal":{"Service":"budgets.amazonaws.com"},
      "Action":"SNS:Publish",
      "Resource":"arn:aws:sns:us-east-1:'"$ACCOUNT_ID"':budget-alerts"}]
  }'
```

### Step 4.2 — Create three budgets

**Console:** Billing → Budgets → Create budget → *Customize*:

1. **`monthly-total`** — Cost budget, $50/month, alerts at **80% actual**, **100% forecasted** → email + SNS
2. **`dev-environment`** — Cost budget, $20/month, filter **Tag: Environment = dev**
3. **`ec2-hours`** — Usage budget, 700 hrs/month of *EC2: Running Hours* (free-tier tripwire)

(CLI versions are in the cheatsheet §3.)

### Step 4.3 — Test it

Set a temporary $0.01 budget → the next budget evaluation (data refreshes ~3x/day) fires the alert → verify the email and SNS message → delete the test budget.

### Step 4.4 — (Optional) Legacy CloudWatch billing alarm

Because you enabled the preference in Lab 1, create the alarm from cheatsheet §13 — good to know for certification questions.

✅ **Checkpoint:** you receive an alert email + SNS notification from the test budget.

---

## Lab 5 — Budget Actions: Auto-Stop EC2 on Overspend

**Goal:** enforcement — when the sandbox budget blows, instances stop.

### Step 5.1 — Execution role for Budgets

Create role `BudgetsActionRole` with trust policy for `budgets.amazonaws.com`:

```bash
cat > trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "budgets.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF
aws iam create-role --role-name BudgetsActionRole --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name BudgetsActionRole \
  --policy-arn arn:aws:iam::aws:policy/AWSBudgetsActionsWithAWSResourceControlAccess
```

### Step 5.2 — Budget with an action (console is easiest)

1. Budgets → Create → Cost budget `sandbox-budget`, $10/month
2. **Configure actions** → Add action:
   - Trigger: **Actual > 100%**
   - Action type: **Stop EC2 instances** → region + instance IDs (your Lab 2 instance)
   - IAM role: `BudgetsActionRole`
   - Approval: **Automatic** (or *Require approval* to see the workflow)
3. Add a second action (optional): **Apply IAM policy** → attach a deny policy (`ec2:RunInstances`) to your dev group

### Step 5.3 — Verify

`aws budgets describe-budget-actions-for-account --account-id $ACCOUNT_ID` — status should be `STANDBY`. When triggered it moves to `EXECUTION_SUCCESS` and the instance stops. You can force a run for testing:

```bash
aws budgets execute-budget-action --account-id $ACCOUNT_ID \
  --budget-name sandbox-budget --action-id <action-id> \
  --execution-type APPROVE_BUDGET_ACTION
```

✅ **Checkpoint:** executing the action stops your tagged EC2 instance.

---

## Lab 6 — Cost Anomaly Detection

**Goal:** catch spend spikes you didn't predict.

1. Cost Management → **Cost Anomaly Detection** → **Cost monitors** → Create:
   - Type: **AWS services** (one monitor watches every service individually)
   - Name: `all-services-monitor`
2. **Alert subscriptions** → Create:
   - Name: `finops-immediate`
   - Frequency: **Individual alerts** → SNS topic `budget-alerts` (reuse; add `costalerts.amazonaws.com` publish permission to the topic policy)
   - Threshold: impact ≥ **$5** (low for the lab; $50–100 real world)
3. Add a second subscription: **Daily summary** → your email
4. CLI equivalents in cheatsheet §4.

> ⏳ The model needs ~10 days of history before it can flag anomalies, and evaluates ~3x/day. Revisit the **Detection history** tab after a week — each anomaly shows a **root cause** (account/service/region/usage-type) and asks for feedback (submit it; it tunes the model).

✅ **Checkpoint:** monitor Active, subscriptions listed by `aws ce get-anomaly-subscriptions`.

---

## Lab 7 — Consolidated Billing with AWS Organizations

**Goal:** one payer, many accounts, shared discounts, org-level guardrails.

> Needs a second email address. Member accounts created via Organizations don't require a credit card.

### Step 7.1 — Create the organization

```bash
aws organizations create-organization --feature-set ALL
aws organizations create-account --email you+dev@example.com --account-name dev-account
watch aws organizations describe-create-account-status --create-account-request-id <id>
```

### Step 7.2 — Observe consolidated billing

- Billing console (management account) → **Bills** → **Charges by account** tab
- Cost Explorer → Group by **Linked account**
- Billing preferences → note the **RI and Savings Plans discount sharing** toggles per account

### Step 7.3 — Cost guardrail SCP

Create + attach the `deny-big-instances` SCP from cheatsheet §9 to a `Sandbox` OU, move `dev-account` into it, and (assuming the `OrganizationAccountAccessRole` role into the member account) verify a `p3.2xlarge` launch is denied while `t3.micro` succeeds.

### Step 7.4 — Org-level cost tooling

- Activate cost allocation tags **in the management account** (tag activation is payer-level!)
- Budgets can filter by **Linked account** → per-account budgets from the payer
- Anomaly monitor type **Linked account** → watch specific accounts

✅ **Checkpoint:** single bill shows per-account charges; SCP blocks the expensive launch.

---

## Lab 8 — CUR 2.0 → S3 → Athena → Dashboard

**Goal:** build the standard FinOps data pipeline.

### Step 8.1 — S3 bucket

```bash
aws s3 mb s3://cur2-$ACCOUNT_ID --region us-east-1
```

### Step 8.2 — Create the Data Export

1. Billing console → **Data Exports** → **Create**
2. Type: **Standard data export (CUR 2.0)**
3. Name `cur2-daily`; include **resource IDs**; granularity **Daily**; format **Parquet**; overwrite versioning
4. Select your bucket — the console **applies the bucket policy for you** (billingreports/bcm-data-exports principals)
5. Create. First delivery lands within ~24h under `s3://cur2-<acct>/…/data/`

(CLI version: cheatsheet §5.2.)

### Step 8.3 — Athena table

Easiest path: use the Glue crawler on the export's `data/` prefix, or create the table manually. Minimal manual DDL for a Parquet CUR 2.0 export:

```sql
CREATE DATABASE IF NOT EXISTS finops;

CREATE EXTERNAL TABLE finops.cur2_daily (
  bill_billing_period_start_date timestamp,
  line_item_usage_account_id string,
  line_item_product_code string,
  line_item_usage_type string,
  line_item_usage_start_date timestamp,
  line_item_resource_id string,
  line_item_unblended_cost double,
  product_region_code string
)
STORED AS PARQUET
LOCATION 's3://cur2-<ACCOUNT_ID>/cur2/cur2-daily/data/';
```

> If you selected different/more columns in the export, match them here — or let a **Glue crawler** infer the schema (recommended for the full column set).

### Step 8.4 — Run the analysis queries

Run every query in cheatsheet §14 (top services, daily trend, top resources, untagged spend, data-transfer costs). Set an Athena query result location first (Athena → Settings).

### Step 8.5 — (Optional) Visualize

- **QuickSight**: connect Athena dataset → build a cost-by-service stacked bar + daily trend line
- Or deploy the open-source **CUDOS / Cloud Intelligence Dashboards** (AWS's own QuickSight dashboard suite on CUR) — the enterprise-standard approach

✅ **Checkpoint:** Athena returns your top-N services matching Cost Explorer's numbers (small deltas are normal until the month finalizes).

---

## Lab 9 — Waste Hunt: Trusted Advisor, Compute Optimizer, Cost Optimization Hub

**Goal:** produce a prioritized savings backlog.

1. **Compute Optimizer:** `aws compute-optimizer update-enrollment-status --status Active` → after ~24–30h of metrics (14 days for full confidence) review `get-ec2-instance-recommendations` — note the *finding* (OVER_PROVISIONED etc.) and per-option savings
2. **Cost Optimization Hub:** `aws cost-optimization-hub update-enrollment-status --status Active` → `list-recommendations` → sort by `estimatedMonthlySavings`
3. **Trusted Advisor:** console → Cost Optimization pillar (full checks need Business support; free tier shows a subset)
4. **Manual sweep:** run every one-liner from cheatsheet §15; record findings in a table:

| Finding | Resource | Monthly $ | Action | Owner |
|---------|----------|-----------|--------|-------|
| gp2 volume | vol-0abc | ~$1.6 saved | migrate gp3 | you |
| Unattached EIP | 3.91.x.x | ~$3.6 | release | you |

5. Execute the safe ones: `aws ec2 modify-volume --volume-id vol-xxx --volume-type gp3`, release idle EIPs, delete old snapshots.

✅ **Checkpoint:** a written savings backlog + at least one applied optimization.

---

## Lab 10 — Savings Plans Analysis & (Simulated) Purchase

**Goal:** understand the full commitment workflow **without spending money**.

> ⚠️ **Do NOT click Purchase / run `create-savings-plan` in a personal account** — it's a binding 1–3 year commitment (7-day return window exists for eligible plans, but don't rely on it).

1. **Recommendations:** Cost Management → Savings Plans → **Recommendations** → Compute SP, 1-year, No Upfront, 30-day lookback. Study: hourly commitment suggested, estimated savings %, estimated On-Demand spend covered
2. CLI: `aws ce get-savings-plans-purchase-recommendation ...` (cheatsheet §2.4) — read `EstimatedSavingsPercentage`, `HourlyCommitmentToPurchase`
3. **Cart mechanics (stop before purchase):** add to cart → note **start date, payment option, total commitment** = hourly × 8,760 (1yr)
4. **Decision drill — answer in writing:**
   - Why commit to *less* than the recommendation? (headroom for usage drops; you can stack more SPs later)
   - When choose EC2 Instance SP over Compute SP? (stable family+region, want max discount)
   - When RIs instead? (RDS/ElastiCache/Redshift/OpenSearch — SPs don't cover them; or need zonal capacity reservation)
5. **Guardrail budgets:** create an **SP utilization budget** (alert < 90%) and an **SP coverage budget** (alert < 70%) — Budgets → *Savings Plans budget* type

✅ **Checkpoint:** you can explain commitment math, and utilization/coverage budgets exist.

---

## Lab 11 — Automation: Scheduled Shutdown of Dev Resources

**Goal:** the highest-ROI automation in FinOps — stop dev at night (~65–70% saved on those instances).

### Option A (simplest): EventBridge Scheduler direct EC2 call

```bash
# Role EventBridge Scheduler assumes
cat > sched-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"scheduler.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name SchedulerEc2StopStart \
  --assume-role-policy-document file://sched-trust.json
aws iam put-role-policy --role-name SchedulerEc2StopStart --policy-name ec2 \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["ec2:StopInstances","ec2:StartInstances"],"Resource":"*"}]}'

# Stop at 20:00 IST every weekday (14:30 UTC)
aws scheduler create-schedule --name stop-dev-nightly \
  --schedule-expression "cron(30 14 ? * MON-FRI *)" \
  --flexible-time-window '{"Mode":"OFF"}' \
  --target '{
    "Arn":"arn:aws:scheduler:::aws-sdk:ec2:stopInstances",
    "RoleArn":"arn:aws:iam::'"$ACCOUNT_ID"':role/SchedulerEc2StopStart",
    "Input":"{\"InstanceIds\":[\"i-0123456789abcdef0\"]}"
  }'

# Start at 08:00 IST (02:30 UTC) — clone with startInstances
```

### Option B (tag-driven): Lambda that stops everything tagged `AutoStop=true`

Lambda (Python) triggered by an EventBridge rule:

```python
import boto3
ec2 = boto3.client("ec2")

def handler(event, context):
    ids = [i["InstanceId"]
           for r in ec2.describe_instances(
               Filters=[{"Name": "tag:AutoStop", "Values": ["true"]},
                        {"Name": "instance-state-name", "Values": ["running"]}]
           )["Reservations"]
           for i in r["Instances"]]
    if ids:
        ec2.stop_instances(InstanceIds=ids)
    return {"stopped": ids}
```

### Option C (enterprise): deploy AWS's **Instance Scheduler on AWS** solution (CloudFormation, cross-account, RDS support).

✅ **Checkpoint:** at the scheduled time your `AutoStop=true` instance stops; next-day Cost Explorer daily view shows the dip.

---

## Lab 12 — Cleanup

```bash
# Terminate lab EC2
aws ec2 terminate-instances --instance-ids <id>

# Delete schedules, Lambda, roles
aws scheduler delete-schedule --name stop-dev-nightly
aws iam delete-role-policy --role-name SchedulerEc2StopStart --policy-name ec2
aws iam delete-role --role-name SchedulerEc2StopStart

# Delete budgets (action-enabled budgets cost after the first 2!)
aws budgets describe-budgets --account-id $ACCOUNT_ID --query "Budgets[].BudgetName"
aws budgets delete-budget --account-id $ACCOUNT_ID --budget-name sandbox-budget
# ... repeat per budget (deleting a budget deletes its actions & alerts)

# Anomaly detection
aws ce get-anomaly-subscriptions --query "AnomalySubscriptions[].SubscriptionArn"
aws ce delete-anomaly-subscription --subscription-arn <arn>
aws ce get-anomaly-monitors --query "AnomalyMonitors[].MonitorArn"
aws ce delete-anomaly-monitor --monitor-arn <arn>

# Data export + bucket (empty first)
aws bcm-data-exports list-exports
aws bcm-data-exports delete-export --export-arn <arn>
aws s3 rb s3://cur2-$ACCOUNT_ID --force

# SNS
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:$ACCOUNT_ID:budget-alerts --region us-east-1

# Athena/Glue
# Athena console → drop table/database; delete query-results bucket contents

# Organizations (only if created for the lab)
# Remove/close member accounts first, then:
# aws organizations delete-organization
```

**Keep** (they're free and useful): cost allocation tag activation, Cost Explorer, one small monthly budget, the anomaly monitor.

---

## 🎓 What You've Built

By the end you have implemented the full FinOps loop on AWS:

```
INFORM   → Tags + Categories + Cost Explorer + CUR/Athena (Labs 2,3,8)
OPTIMIZE → Waste backlog + rightsizing + commitment analysis (Labs 9,10)
OPERATE  → Budgets + Actions + Anomaly Detection + Automation + Org guardrails (Labs 4,5,6,7,11)
```
