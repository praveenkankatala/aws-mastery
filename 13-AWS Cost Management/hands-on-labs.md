# AWS Billing & Cost Management — Hands-On Labs

> Step-by-step labs from a fresh account to a fully automated FinOps setup. Do them in order — later labs depend on earlier ones. Each lab states **What / Why / How** plus verification steps.

**Lab index**
1. [Enable Billing Access & Cost Explorer](#lab-1)
2. [Tagging Strategy + Cost Allocation Tags](#lab-2)
3. [Cost Categories with Split Charges](#lab-3)
4. [AWS Budgets with SNS, Slack & Budget Actions](#lab-4)
5. [Cost Anomaly Detection](#lab-5)
6. [CUR 2.0 Data Export → S3 → Athena](#lab-6)
7. [CUDOS / Cost & Usage Dashboard in QuickSight](#lab-7)
8. [Rightsizing + CloudWatch Agent Memory Metrics](#lab-8)
9. [Compute Optimizer (Org-Wide)](#lab-9)
10. [Trusted Advisor + EventBridge Auto-Remediation](#lab-10)
11. [Cost Optimization Hub](#lab-11)
12. [Service Quotas Alarms, Automatic Management & Templates](#lab-12)

---

<a name="lab-1"></a>
## Lab 1 — Enable Billing Access & Cost Explorer

**What:** Unlock billing visibility for IAM identities and turn on Cost Explorer.
**Why:** By default only the root user sees billing; Cost Explorer needs a one-time enablement and ~24h to populate.

**Steps:**
1. Sign in as **root** → account menu (top-right) → **Account** → scroll to **IAM user and role access to Billing information** → **Edit → Activate → Update**.
2. Create/attach an IAM policy for your admin role including `ce:*`, `budgets:*`, `cur:*`, `aws-portal:View*` (or the newer fine-grained billing actions `billing:*`, `payments:View*` etc. if your account has migrated).
3. Sign in as your IAM identity → open **Billing and Cost Management** console → **Cost Explorer** → **Launch Cost Explorer**.
4. Wait ~24 hours.

**Verify:**
- Cost Explorer shows the current month + up to 13 months of history.
- CLI works: `aws ce get-cost-and-usage --time-period Start=2026-07-01,End=2026-07-28 --granularity MONTHLY --metrics UnblendedCost`

**Explore:** Change *Group by* to **Service**, granularity to **Daily**, toggle metric between **Unblended** and **Amortized**, and open the **Forecast** band (80% confidence interval).

---

<a name="lab-2"></a>
## Lab 2 — Tagging Strategy + Cost Allocation Tags

**What:** Define a tag schema, tag resources, activate tags for billing, and slice costs by them.
**Why:** Tags turn a service-oriented bill into a business-oriented one. They're **not retroactive** — do this early.

**Steps:**
1. **Define the schema** (document it!):
   | Key | Allowed Values | Mandatory |
   |---|---|---|
   | `Environment` | Production, Staging, Development | ✅ |
   | `Project` | Alpha, Beta, ... | ✅ |
   | `Owner` | team email | ✅ |
   | `CostCenter` | 4-digit code | ✅ |
2. **Tag existing resources:**
   ```bash
   aws ec2 create-tags --resources i-0abc123 --tags Key=Environment,Value=Production Key=Project,Value=Alpha
   ```
   For IaC, bake tags into Terraform `default_tags` / CloudFormation stack tags.
3. **Activate in billing:** Billing console → **Cost Allocation Tags** → *User-defined* tab → select `Environment`, `Project`, `Owner`, `CostCenter` → **Activate**. Also activate the AWS-generated tag **`aws:createdBy`** on the *AWS-generated* tab.
4. **Wait up to 24 hours.**
5. **Use it:** Cost Explorer → *Group by* → **Tag: Environment**. Then try *Filter* → Tag `Project = Alpha`, *Group by* → Service.
6. *(Optional)* **Backfill** past periods: Cost Allocation Tags page → **Backfill tags** (one request per quarter allowed).
7. **Enforce going forward:** Organizations → Policies → **Tag Policies** (standardize casing/values) and an SCP that denies `RunInstances` without `aws:RequestTag/Environment`.

**Verify:** `aws ce list-cost-allocation-tags` shows `Status: Active`; a "No Tag" bar exists for pre-activation days (expected).

---

<a name="lab-3"></a>
## Lab 3 — Cost Categories with Split Charges

**What:** Map accounts + tags into business units and distribute shared costs.
**Why:** Categories work at the billing layer (no resource changes) and support first-match rule logic + shared-cost splitting.

**Steps:**
1. Billing console → **Cost Categories** → **Create cost category** → name `BusinessUnit`.
2. Add ordered rules:
   - Rule 1: Dimension *Linked Account* IN (`111122223333`) → value **Platform**
   - Rule 2: Tag `Team = retail` → value **Retail**
   - Default value: **Unallocated**
3. Add a **Split charge rule**: source = **Platform** (shared services), targets = Retail + others, method = **Proportional** (or Fixed %/Even split).
4. Save; values appear in Cost Explorer/CUR within ~24h under dimension **Cost Category: BusinessUnit**.

**Verify:** Cost Explorer → Group by → Cost Category → `BusinessUnit`.

---

<a name="lab-4"></a>
## Lab 4 — AWS Budgets with SNS, Slack & Budget Actions

**What:** A forecast-aware guardrail with automated response.
**Why:** Alerts *before* overspend; actions can literally stop new spend.

**Steps:**
1. **SNS topic:**
   ```bash
   aws sns create-topic --name budget-alerts
   aws sns subscribe --topic-arn <arn> --protocol email --notification-endpoint you@example.com
   ```
   Edit the topic **access policy** to allow `budgets.amazonaws.com` to `SNS:Publish` (see troubleshooting doc for the JSON). Confirm the email subscription.
2. **Create the budget:** Billing → **Budgets** → **Create budget** → *Customize* → **Cost budget**.
   - Period: Monthly; Amount: e.g., $1,000; Scope: Filter by Tag `Environment = Production`.
3. **Alerts:**
   - Alert 1: **Actual** > 80% → email.
   - Alert 2: **Forecasted** > 100% → SNS topic.
4. **Slack/Teams:** set up **AWS Chatbot** → map the SNS topic to a Slack channel.
5. **Budget Action (advanced):**
   - Create IAM policy `DenyExpensiveLaunches` (Deny `ec2:RunInstances` on large types).
   - Create role `BudgetsActionRole` trusted by `budgets.amazonaws.com`.
   - In the budget → **Actions** → *Apply IAM policy* to group/role `Developers` at 100% actual, approval = Automatic. (Alternatives: attach SCP, stop EC2/RDS instances, run SSM Automation.)
6. Repeat: create one **overall** monthly forecast budget for the whole account.

**Verify:** `aws budgets describe-budgets --account-id <id>`; temporarily set the amount to $1 to force a test alert, then restore.

---

<a name="lab-5"></a>
## Lab 5 — Cost Anomaly Detection

**What:** ML monitors + subscriptions for spend-pattern breaks.
**Why:** Catches spikes budgets miss; free.

**Steps:**
1. Billing → **Cost Anomaly Detection** → **Create monitor**.
2. Monitor type: **AWS services** (recommended single monitor). *(Alternatives: Linked account / Cost category / Cost allocation tag monitors for team-level ownership.)*
3. **Create subscription:** name `FinOpsAlerts`; frequency **Individual alerts** (immediate) via SNS, plus a **Daily summary** via email; threshold: impact ≥ $50 (tune to your scale).
4. Wait ~24h for the model to baseline (needs ~10 days of history for best accuracy).
5. When an anomaly fires: open it → review **root-cause analysis** (account/service/region/usage-type) → submit feedback (*expected* / *not an issue*) to tune the model.

**Verify:** `aws ce get-anomaly-monitors` and `aws ce get-anomaly-subscriptions`.

---

<a name="lab-6"></a>
## Lab 6 — CUR 2.0 Data Export → S3 → Athena

**What:** The raw billing ledger, queryable with SQL.
**Why:** Per-resource/per-hour truth, permanent retention, chargeback and custom analytics.

**Steps:**
1. **S3 bucket** (e.g., `org-cur-bucket`, ap-south-1). The export wizard can auto-apply the required bucket policy allowing `billingreports.amazonaws.com` / `bcm-data-exports.amazonaws.com`.
2. Billing → **Data Exports** → **Create** → **Standard data export (CUR 2.0)**.
   - Name: `cur2-daily`; content: include **Resource IDs** and **Split cost allocation data** (EKS/ECS); time granularity: **Hourly**.
   - Delivery: Parquet, Overwrite versioning, your bucket/prefix.
3. Wait up to 24h for the first delivery.
4. **Athena setup:**
   - Easiest: use the AWS-provided Glue crawler/CloudFormation from the legacy Athena-integration flow, or create a Glue Crawler pointed at `s3://org-cur-bucket/cur2/` → database `cur_db`.
   - Set an Athena **query result location** (Settings → `s3://org-cur-bucket/athena-results/`).
5. Run queries (see cheatsheet §12): top resources, cost per tag, data-transfer breakdown, amortized view.
6. *(Optional)* Create a **FOCUS 1.0** export the same way (table `FOCUS_1_0`) for multi-cloud-standard schema.

**Verify:** Parquet files land under the prefix; `SELECT count(*) FROM cur_db.cur_table` returns rows.

---

<a name="lab-7"></a>
## Lab 7 — CUDOS / Cost & Usage Dashboard (QuickSight)

**What:** Deploy the pre-built Cloud Intelligence Dashboard from the Billing console.
**Why:** Executive-ready visuals with zero manual pipeline building — AWS provisions the Data Export, Athena schemas, and QuickSight dashboard for you.

**Prerequisites:** Management/payer account; **QuickSight Enterprise Edition** active in the target region (skip Q add-ons to keep licensing lean).

**Steps:**
1. Billing console → **Analytics and Insights → Dashboards** → *Cost and Usage Dashboard powered by QuickSight* → **Get started / Configure dashboard**.
2. **Data export config:** accept the default name (e.g., `bcm-dashboards-export`), choose/create the S3 bucket, ensure **Parquet + Daily** refresh.
3. **QuickSight permissions:** QuickSight console → profile icon → **Manage QuickSight → Security & Permissions → Manage** → tick **Athena** and **S3** (select the export bucket specifically) → Save.
4. Back in the Billing wizard → **Deploy dashboard** → you're redirected to **CloudFormation** with the CUDOS-v5 template pre-loaded → review parameters (QuickSight principal ARN, S3 export location) → acknowledge IAM capabilities → **Create stack**.
5. Infrastructure (Glue crawlers, Athena schemas, SPICE datasets) is ready in ~15 minutes; **full data can take up to 24 hours**.
6. View it in **QuickSight → Dashboards** (not the Billing console).

**Extensions:** restrict tabs per team with QuickSight **Row-Level Security**; schedule **email report delivery**.

---

<a name="lab-8"></a>
## Lab 8 — Rightsizing + CloudWatch Agent Memory Metrics

**What:** Get terminate/downsize recommendations you can actually trust.
**Why:** Without the CloudWatch Agent, AWS can't see memory — recommendations are CPU/IO-only and risky for memory-bound apps.

**Steps:**
1. **Enable recommendations:** Billing → Cost Explorer → **Preferences** → enable *Rightsizing recommendations* (and cross-family option).
2. **Install CloudWatch Agent fleet-wide via SSM** (instances need the SSM agent + an instance profile with `CloudWatchAgentServerPolicy`):
   ```bash
   aws ssm send-command --document-name "AWS-ConfigureAWSPackage" \
     --targets "Key=tag:Environment,Values=Production" \
     --parameters '{"action":["Install"],"name":["AmazonCloudWatchAgent"]}'
   ```
3. **Store agent config in SSM Parameter Store** (`AmazonCloudWatch-Config`) including:
   ```json
   {"metrics":{"append_dimensions":{"InstanceId":"${aws:InstanceId}"},
     "metrics_collected":{"mem":{"measurement":["mem_used_percent"]},
                          "disk":{"measurement":["used_percent"],"resources":["*"]}}}}
   ```
4. **Start the agent with that config:**
   ```bash
   aws ssm send-command --document-name "AmazonCloudWatch-ManageAgent" \
     --targets "Key=tag:Environment,Values=Production" \
     --parameters '{"action":["configure"],"mode":["ec2"],"optionalConfigurationSource":["ssm"],"optionalConfigurationLocation":["AmazonCloudWatch-Config"],"optionalRestart":["yes"]}'
   ```
5. Also enable **memory utilization ingestion** in Compute Optimizer preferences (it reads the `CWAgent` namespace).
6. Wait the **14-day lookback** window.
7. Review: Billing → **Recommendations/Rightsizing** → filter by account/region/tag → note `Idle` (terminate) vs `Underutilized` (modify), intra- vs cross-family, and Graviton suggestions → **Download CSV** for the implementation ticket.

**Verify:** CloudWatch → Metrics → `CWAgent` namespace shows `mem_used_percent`; `aws ce get-rightsizing-recommendation --service AmazonEC2` returns entries.

---

<a name="lab-9"></a>
## Lab 9 — Compute Optimizer (Org-Wide)

**What:** Turn on the ML engine for the whole organization and tune it.

**Steps:**
1. From the **management account**:
   ```bash
   aws compute-optimizer update-enrollment-status --status Active --include-member-accounts
   ```
2. Console → **Compute Optimizer** → Dashboard: watch findings populate (`Optimized / Over- / Under-provisioned / Idle`) after ~24h (needs ≥30h of CloudWatch data per resource; recommendations refresh daily).
3. **Preferences:**
   - Enable **Enhanced Infrastructure Metrics** (3-month lookback) for cyclical workloads.
   - If pipelines are x86-only, exclude Graviton families; otherwise leave Graviton on for the price-performance wins.
4. Open an EC2 recommendation → check the **Performance Risk** score and the **projected utilization** graph on each option before acting.
5. Explore beyond EC2: **EBS, Lambda, ECS/Fargate, RDS/Aurora, idle resources, license** tabs.
6. *(Optional)* **Automation rules:** create a policy (SSM service-linked role) that auto-deletes idle unattached EBS volumes in non-prod on a schedule.
7. Export org-wide recommendations to S3 for reporting:
   ```bash
   aws compute-optimizer export-ec2-instance-recommendations \
     --s3-destination-config bucket=my-optimizer-exports,keyPrefix=ec2/ --file-format Csv --include-member-accounts
   ```

---

<a name="lab-10"></a>
## Lab 10 — Trusted Advisor + EventBridge Auto-Remediation

**What:** Wire waste-detection checks to automatic cleanup.
**Prereq:** Business/Enterprise Support for cost checks. All TA events emit in **us-east-1**.

**Steps:**
1. Console → **Trusted Advisor** → **Cost Optimization**: review Low-Utilization EC2, Idle RDS, Underutilized EBS, Unassociated EIPs, Idle Load Balancers, S3 incomplete multipart uploads.
2. **Quick win — S3 multipart cleanup:** add a lifecycle rule to every bucket:
   ```bash
   aws s3api put-bucket-lifecycle-configuration --bucket my-bucket \
     --lifecycle-configuration '{"Rules":[{"ID":"abort-mpu","Status":"Enabled","Filter":{},"AbortIncompleteMultipartUpload":{"DaysAfterInitiation":7}}]}'
   ```
3. **Lambda remediator** (us-east-1): a function that parses the TA event, and e.g. releases unassociated EIPs / snapshots+deletes unattached EBS volumes older than N days (guard with a `DoNotDelete` tag check!).
4. **EventBridge rule** (us-east-1) matching `aws.trustedadvisor` status → `WARN|ERROR` for the chosen checks → target = the Lambda (see cheatsheet §13 for the pattern). Add a second target = SNS for a Slack notification via Chatbot.
5. Force a test: allocate an EIP, don't attach it, then `aws support refresh-trusted-advisor-check --check-id <EIP-check-id> --region us-east-1` (cost data otherwise refreshes ~weekly).

---

<a name="lab-11"></a>
## Lab 11 — Cost Optimization Hub

**What:** One consolidated, deduplicated savings backlog.

**Steps:**
1. Billing console → **Cost Optimization Hub** → **Opt in** (from the management account tick *include member accounts*).
2. Preferences: savings estimation **After discounts** (net-true numbers), member-account discount visibility as policy allows.
3. Dashboard: sort recommendations by **estimated monthly savings**; filter by account, region, tag, or action type (Stop / Rightsize / Upgrade / Graviton migrate / Purchase SP-RI / Delete).
4. Export/summarize for sprint planning:
   ```bash
   aws cost-optimization-hub list-recommendation-summaries --group-by ActionType
   ```
5. Feed the top items into your ticketing flow (the API pairs well with Jira automation).

---

<a name="lab-12"></a>
## Lab 12 — Service Quotas Alarms, Automatic Management & Templates

**What:** Never fail a scale-out because of an invisible ceiling.

**Steps:**
1. Console → **Service Quotas** → e.g., *Amazon EC2 → Running On-Demand Standard instances (vCPU)* → view the **utilization graph**.
2. **Create a CloudWatch alarm from the quota page** at **80%** utilization → SNS → DevOps channel.
3. **Automatic Management:**
   ```bash
   aws service-quotas start-quota-utilization-notifications --opt-in-type NotifyOnly
   # or, hands-free increases:
   aws service-quotas start-quota-utilization-notifications --opt-in-type NotifyAndAdjust
   ```
   (`NotifyOnly` alerts at 80%/95% every 24h via AWS Health/User Notifications/EventBridge; `NotifyAndAdjust` also auto-files increase requests.)
4. **Manual increase when needed:**
   ```bash
   aws service-quotas request-service-quota-increase --service-code ec2 --quota-code L-1216C47A --desired-value 256
   ```
5. **Org template** (management account): associate the template, add standard increases (vCPUs, VPCs, EIPs) per region → every new member account (e.g., via Control Tower) gets them automatically.
6. Remember: quotas are **per-region** — repeat for your DR region.

---

## 🎓 Final Capstone

Combine everything into a mini FinOps platform:
1. Tags + Categories feeding Cost Explorer & CUR (Labs 2–3, 6).
2. Guardrails: Budgets + Anomaly Detection (Labs 4–5).
3. Dashboards: CUDOS (Lab 7).
4. Optimization loop: Compute Optimizer → Cost Optimization Hub → tickets (Labs 8–9, 11).
5. Hygiene automation: Trusted Advisor + EventBridge (Lab 10).
6. Scale readiness: Service Quotas (Lab 12).

Document your monthly review ritual: **Inform → Optimize → Operate**. That's FinOps.
