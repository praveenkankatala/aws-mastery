# AWS Billing & Cost Management — End-to-End Practical Guide (FinOps)

> A complete, hands-on learning repository covering the entire AWS Billing & Cost Management ecosystem — from raw billing data ingestion to automated cost optimization and governance. Built in a **What → Why → How** style so anyone (beginner to cloud engineer) can follow along.

---

## 📁 Repository Structure

| File | Purpose |
|------|---------|
| [`README.md`](README.md) | Overview, architecture, core features deep-dive, configuration guide, use cases |
| [`commands-cheatsheet.md`](commands-cheatsheet.md) | All AWS CLI commands, organized by service |
| [`hands-on-labs.md`](hands-on-labs.md) | Step-by-step labs from scratch to full deployment |
| [`troubleshooting.md`](troubleshooting.md) | Common issues, error messages, and solutions |

---

## 1. What Is AWS Billing & Cost Management?

**What:** AWS Billing & Cost Management is a suite of services that lets you **visualize, analyze, allocate, forecast, control, and optimize** your AWS spend across one account or an entire AWS Organization.

**Why:** Cloud costs are elastic — they can grow silently. Without visibility (which team spent what?), guardrails (budgets, alerts), and optimization (rightsizing, commitments), organizations routinely overspend by 20–40%. This discipline is called **FinOps** (Cloud Financial Operations).

**How:** AWS provides a layered toolset — each tool answers a different question:

| Tool | Question It Answers |
|------|---------------------|
| **Cost Explorer** | *What am I spending, on what, and what's the trend?* |
| **Cost & Usage Report (CUR) / Data Exports** | *Why exactly? (raw ledger, per-resource, per-hour)* |
| **Cost Allocation Tags & Cost Categories** | *Who owns this spend? (team / project / environment)* |
| **AWS Budgets** | *Am I about to overspend? Stop it before it happens.* |
| **Cost Anomaly Detection** | *Did anything unusual just happen?* |
| **Rightsizing / Compute Optimizer** | *Is my infrastructure over-provisioned?* |
| **Trusted Advisor** | *Am I following best practices? What's actively wasted?* |
| **Cost Optimization Hub** | *One consolidated view of all savings opportunities* |
| **Savings Plans / Reserved Instances** | *How do I pay less for steady-state workloads?* |
| **Service Quotas** | *Will I hit a limit before I can even scale?* |
| **Billing Conductor** | *How do I show custom internal chargeback rates?* |
| **Consolidated Billing (Organizations)** | *One bill for many accounts, shared discounts* |

---

## 2. High-Level Architecture & Service Flow

```
                          ┌──────────────────────────────────────────┐
                          │            AWS ORGANIZATION              │
                          │  (Management / Payer Account at the top) │
                          └───────────────────┬──────────────────────┘
                                              │ Consolidated Billing
        ┌──────────────┬──────────────────────┼──────────────────────┐
        ▼              ▼                      ▼                      ▼
   [ Dev Acct ]   [ QA Acct ]           [ Prod Acct ]         [ Shared Svcs ]
        │              │                      │                      │
        └──────────────┴──────── Usage Metering (per resource) ──────┘
                                              │
                                              ▼
                        ┌─────────────────────────────────────┐
                        │        AWS BILLING DATASET          │
                        │   (single source of truth for $$)   │
                        └───────┬──────────────────┬──────────┘
                                │                  │
              ┌─────────────────┘                  └──────────────────┐
              ▼                                                       ▼
   ┌──────────────────────┐                          ┌────────────────────────────┐
   │   COST EXPLORER      │                          │  CUR 2.0 / DATA EXPORTS    │
   │  (Interactive UI +   │                          │  Raw .parquet/.csv → S3    │
   │   API + Forecasts)   │                          └──────────┬─────────────────┘
   └─────────┬────────────┘                                     │
             │                                        ┌─────────▼─────────┐
   ┌─────────▼──────────┐                             │  Amazon Athena    │
   │  AWS BUDGETS       │                             │  (SQL over CUR)   │
   │  Actual/Forecast   │                             └─────────┬─────────┘
   │  alerts + Actions  │                                       │
   └─────────┬──────────┘                             ┌─────────▼─────────┐
             │                                        │ Amazon QuickSight │
             ▼                                        │ (CUDOS Dashboards)│
   [ SNS → Email / Slack / Chatbot ]                  └───────────────────┘

   ┌─────────────────────────── OPTIMIZATION LAYER ───────────────────────────┐
   │  CloudWatch Metrics ──► Compute Optimizer (ML) ──► Rightsizing Recs     │
   │  Trusted Advisor Checks ──► EventBridge ──► Lambda / SSM (auto-fix)     │
   │  Cost Anomaly Detection (ML monitors) ──► SNS / Email alerts            │
   │  Cost Optimization Hub  ◄── aggregates all of the above, org-wide       │
   └──────────────────────────────────────────────────────────────────────────┘
```

**Flow summary:**
1. Every resource in every account emits **metered usage**.
2. Consolidated Billing aggregates it into the **payer account's billing dataset**.
3. **Cost Explorer** visualizes it; **CUR/Data Exports** dump the raw ledger to S3.
4. **Tags & Cost Categories** slice the data by business dimensions.
5. **Budgets & Anomaly Detection** act as proactive guardrails.
6. **Compute Optimizer, Trusted Advisor, Cost Optimization Hub** close the loop with optimization actions.

---

## 3. Prerequisites

- An AWS account (ideally the **management/payer account** of an AWS Organization for full features).
- **IAM access to Billing activated** — by default, IAM users/roles *cannot* see billing even with `AdministratorAccess`. The root user must enable *"IAM user and role access to Billing information"* in Account Settings.
- IAM permissions: `ce:*`, `budgets:*`, `cur:*`, `bcm-data-exports:*`, `compute-optimizer:*`, `support:*` (Trusted Advisor API needs Business+ support), `servicequotas:*` as needed.
- AWS CLI v2 installed and configured (`aws configure`).
- For dashboards: **Amazon QuickSight Enterprise Edition** subscription.
- For Trusted Advisor's full checks: **Business or Enterprise Support plan**.

---

## 4. Core Features & Deep-Dive

### 4.1 AWS Cost Explorer

**What:** The primary interactive console tool to visualize and analyze costs and usage over time, with filtering, grouping, forecasting, and an API.

**Why:** It's the fastest way to answer "where is my money going?" without building any data pipeline.

**How it works:**
- **Enablement & data:** Enable it once in the payer account; it backfills the **current month + up to 13 months of history**. First data takes ~**24 hours**; refreshes **at least once every 24 hours**.
- **Granularity:** Monthly and daily views; **hourly granularity** available (for up to the last 14 days) if you opt in (extra cost).
- **Filtering (micro view):** Isolate a slice — e.g., only `Project: Alpha`.
- **Group By (macro view):** Break the chart into segments by a dimension.

**Key dimensions:**
| Dimension | Example Use |
|-----------|-------------|
| Service | EC2 vs S3 vs RDS spend |
| Linked Account | Which sub-account (Dev/QA/Prod) drives cost |
| Region | `us-east-1` vs `ap-south-1` distribution |
| Usage Type / Usage Type Group | Data Transfer (Inter-AZ, Internet-Out), instance-hours |
| Instance Type | `m5.xlarge` vs `t3.medium` |
| Purchase Option | On-Demand vs Spot vs Reserved |
| Tag (Cost Allocation Tag) | `Environment: Production`, `Owner: DevOpsTeam` |
| Cost Category | Custom business groupings |

**Cost metrics (aggregation types):**
- **Unblended Cost** — the actual charge on the day the resource was consumed (default; cash-basis view).
- **Amortized Cost** — spreads upfront RI/Savings Plan payments evenly over the term (true daily operational cost; accrual view).
- **Blended Cost** — averaged rates across the whole organization (consolidated-billing accounting view).
- **Net costs** — variants after discounts/credits.

**Forecasting:**
- ML-based forecast for up to **12–18 months** ahead, shown with an **80% confidence interval** band.
- Great for predicting whether current growth will breach budgets.

**Built-in commitment analysis:**
- **RI/Savings Plans Utilization reports** — how much of your purchased commitment is actually being used (low utilization = wasted prepaid capacity).
- **Coverage reports** — what % of eligible usage is covered by discounts (low coverage = you're paying On-Demand unnecessarily).
- **Recommendations engine** — analyzes 7/30/60-day history and suggests specific RI or Savings Plan purchases with estimated savings.

**Analyze with Amazon Q:** Built-in generative AI can analyze your current report view, explain spikes, spot anomalies, and suggest optimizations.

**Cost Explorer API:** Programmatic access (`aws ce ...`) for custom dashboards/tools. **Each paginated API request costs $0.01.**

---

### 4.2 AWS Cost and Usage Report (CUR) / Data Exports

**What:** The most granular, comprehensive raw billing dataset AWS produces — every line item, resource, API-metered charge, and discount, delivered as `.csv` or `.parquet` files into **your own S3 bucket**.

**Why:** Cost Explorer is the *dashboard*; CUR is the *ledger*. When you need per-resource, per-hour truth (chargeback, audits, custom analytics, container cost splitting), you need CUR.

**CUR vs Cost Explorer:**

| Feature | Cost Explorer | CUR |
|---|---|---|
| Format | Interactive UI/charts | Raw files in S3 (`.csv`/`.parquet`) |
| Granularity | Daily/Monthly (hourly ≤14 days) | Hourly/daily/monthly, per-resource |
| Resource IDs | Aggregated | Individual ARNs/IDs (`i-0abc...`) |
| Retention | 13 months | Permanent (your S3 bucket) |
| Container detail | No | **Split Cost Allocation** for ECS tasks / EKS pods |

**Anatomy — the "5 Ws" of a CUR row:**
- **What** was spent → `line_item_unblended_cost` / `line_item_net_amortized_cost`
- **Who** spent it → `line_item_usage_account_id`
- **When** → `line_item_usage_start_date` / `line_item_usage_end_date`
- **Where** → `product_region`
- **Why** (which resource) → `line_item_resource_id`

Other important columns: `line_item_line_item_type` (Usage, Tax, Credit, RIFee, SavingsPlanCoveredUsage…), `pricing_term`, `reservation_*`, `savings_plan_*`, and `resource_tags_user_*` for every activated tag.

**Advanced capabilities:**
- **Split Cost Allocation Data** — ingests container telemetry to split the cost of shared EC2 hosts across individual **EKS pods / ECS tasks** → true microservice-level chargeback.
- **Commitment lifecycle detail** — which instance consumed which RI hour, the equivalent On-Demand price, exact amortization of upfront payments.
- **CUR 2.0 + AWS Data Exports** — the modern engine: define **SQL-like column selection/filters at export time**, reduce data sparsity, choose only the columns you need.
- **FOCUS 1.0 export** — export billing data conforming to the **FinOps Open Cost and Usage Specification**, a vendor-neutral schema usable identically across AWS/Azure/GCP.

**Standard analytics pipeline:**
```
[ AWS Billing ] → [ S3 (Parquet) ] → [ AWS Glue Crawler ] → [ Amazon Athena (SQL) ] → [ QuickSight / Grafana ]
```

---

### 4.3 Cost Allocation Tags

**What:** Key–value labels on resources (`Environment=Prod`, `CostCenter=1234`) that — once **activated** in the Billing console — become filterable/groupable dimensions in Cost Explorer, CUR, and Budgets.

**Why:** AWS bills by *service*, but businesses think in *teams, projects, environments*. Tags bridge that gap and enable showback/chargeback.

**How (operational flow):**
1. **Apply tags** to resources — via console, IaC (Terraform/CloudFormation/Ansible), or the Resource Groups Tagging API.
2. **Activate** the tag keys in **Billing Console → Cost Allocation Tags** (they start as inactive user-defined tags).
3. **Wait up to 24 hours** for ingestion.
4. **Filter/Group** by the tag in Cost Explorer.

**Two tag types:**
- **AWS-generated tags** (e.g., `aws:createdBy`) — automatically applied; identifies the IAM principal that created the resource. Excellent for tracing surprise costs to a person/pipeline.
- **User-defined tags** — your custom schema (`Owner`, `Application`, `CostCenter`).

**Critical nuances:**
- **Not retroactive.** Activate on the 15th → days 1–14 appear as **"No Tag"**.
- **Case-sensitive.** `Environment`, `environment`, and `Env` are three different dimensions — enforce a schema.
- **Some charges can't be tagged** — Support fees, some data-transfer line items, subscriptions → always some "No Tag" residue.
- **Enforce with guardrails:** AWS Organizations **Tag Policies** (standardize keys/values) and **SCPs** (block resource creation without mandatory tags).

---

### 4.4 Cost Categories

**What:** Rule-based virtual groupings that map raw billing data (accounts, services, tags, charge types, other categories) into your **own business hierarchy** — e.g., a "BusinessUnit" category with values `Retail`, `Payments`, `Platform`.

**Why:** Tags require touching every resource; Cost Categories are defined once at the billing layer and can combine multiple rules (e.g., "these 4 accounts + anything tagged Team=Search = *Search BU*"). They also support **split charge rules** to distribute shared costs (support fees, shared services) proportionally or by fixed percentage across categories.

**How:** Billing Console → Cost Categories → define rules (ordered, first-match-wins) → values appear as a dimension across Cost Explorer, CUR, and Budgets.

---

### 4.5 AWS Budgets

**What:** Proactive guardrails — define a Cost, Usage, RI/SP Utilization, or RI/SP Coverage budget and get alerts (or automated actions) when thresholds are crossed.

**Why:** Cost Explorer is *reactive* (what happened). Budgets checks your data (roughly daily) and warns you **before** the bill lands.

**Alert types:**
- **Actual** — "Alert me when spend crosses 80% of ₹/$1,000."
- **Forecasted** — ML projects your end-of-period spend; "alert me if we're *on track* to hit 110%," even if the money isn't spent yet.

**Notification channels:** Email (up to 10 recipients), **Amazon SNS** topic, **AWS Chatbot** → Slack / Microsoft Teams / Amazon Chime.

**Budget Actions (advanced):** When a threshold breaches, automatically:
- Apply a restrictive **IAM policy** or **SCP** (e.g., deny `ec2:RunInstances`) to stop new spend,
- Stop specific **EC2/RDS instances**,
- Trigger **SSM Automation** or **Lambda** workflows.
Actions can run automatically or require manual approval.

**Scoping:** Filter budgets by service, linked account, region, tag, or Cost Category — e.g., a budget that watches only `Environment: Production`.

**Free tier:** First **2 budgets free**; action-enabled budgets and additional budgets have small daily charges.

---

### 4.6 AWS Cost Anomaly Detection

**What:** ML-driven monitors that learn your historical spend patterns per segment (service, account, tag, or cost category) and alert you on **unusual spikes** with root-cause hints (which service/account/region/usage type drove it).

**Why:** Budgets catch *threshold* breaches; anomaly detection catches *pattern* breaks — e.g., a misconfigured Lambda loop tripling NAT Gateway charges on day 3 of the month, long before any monthly budget trips.

**How:**
1. Create a **monitor** (AWS Services monitor is the recommended default; or Linked Account / Cost Category / Tag monitors).
2. Create an **alert subscription** — individual alerts (immediate), daily, or weekly summaries; delivered via email or SNS.
3. Set an alert threshold (absolute $ impact or anomaly-score based).
4. Provide feedback ("expected / not an issue") to tune the model. The service itself is **free** (you pay only for SNS etc.).

---

### 4.7 Rightsizing Recommendations (Cost Explorer)

**What:** Console feature that flags **idle** and **underutilized** EC2 instances and recommends terminate/downsize actions, with estimated savings.

**Why:** Over-provisioning is the #1 source of cloud waste. Rightsizing = matching resource size/type to the real workload at the lowest cost.

**How the engine evaluates (14-day lookback):**
- **Idle** → max CPU ≤ **1%** across 14 days → recommendation: **Terminate**.
- **Underutilized** → running with large headroom → recommendation: **Modify (downsize)**.

**Metrics analyzed:** CPU utilization, network I/O, disk I/O from CloudWatch. **Memory is NOT visible by default** — install the **CloudWatch Agent** to publish `mem_used_percent`, otherwise recommendations are CPU/IO-driven only.

**Recommendation paths:**
- **Intra-family:** `m6g.2xlarge → m6g.xlarge` (lowest risk; same architecture).
- **Cross-family:** `t3.xlarge → c6i.large` (workload-shape driven).
- **Graviton advantage:** engine often suggests x86 → **Graviton (ARM)** moves — up to **~40% better price-performance**.

**Commitment-aware math:** The engine accounts for size-flexible RIs (discount scales down cleanly) vs fixed commitments (downsizing could strand a prepaid RI). It only shows recommendations where **net estimated savings ≥ $0**.

---

### 4.8 AWS Compute Optimizer

**What:** The deep ML engine behind rightsizing — analyzes performance telemetry and recommends optimal configurations across many resource types, not just EC2.

**Why:** Cost Explorer looks at *dollars*; Compute Optimizer looks at *performance shape* (bursty vs steady, CPU-bound vs memory-bound) using models trained on millions of workloads — and shows you **projected utilization** on the recommended type *before* you change anything.

**Coverage:**
- **EC2 instances & Auto Scaling groups** (family/generation/size)
- **EBS volumes** (IOPS, throughput, size; checks like `VolumeIOPSExceededCheck`)
- **Lambda functions** (memory-size vs duration/cost balance)
- **ECS services on Fargate** (task CPU/memory)
- **Databases** — Aurora, RDS, DynamoDB, ElastiCache (Redis/Valkey), MemoryDB, DocumentDB
- **Others** — WorkSpaces, SageMaker endpoints, commercial **license optimization** (e.g., SQL Server on EC2)

**Findings classification:** `Optimized` / `Over-provisioned` / `Under-provisioned` / `Idle`.

**Key features:**
- **Lookback:** default **14 days**; enable **Enhanced Infrastructure Metrics** (paid) for up to **3 months** — essential for monthly-cyclical workloads.
- **Performance Risk score** (Very Low → High) per recommendation — probability the new size struggles at peak.
- **Recommendation preferences:** include/exclude architectures (e.g., exclude Graviton if pipelines are x86-only), prefer specific families.
- **Automation rules:** policies via SSM service-linked roles that **auto-apply** matching recommendations (e.g., auto-delete idle non-prod EBS volumes) on a schedule.
- **Org-wide view:** opt in at the management account to see all member accounts.

---

### 4.9 AWS Trusted Advisor

**What:** An automated "cloud consultant" running rule-based checks against best practices across **five pillars**: Cost Optimization, Security, Fault Tolerance, Performance, Service Limits (+ Operational Excellence in newer console).

**Why:** It catches *actively wasted* and *risky* resources with simple, binary thresholds — no ML, immediate signal.

**Signature cost checks (with thresholds):**
| Check | Trigger |
|---|---|
| Low-utilization EC2 | Daily avg CPU ≤ **10%** AND network I/O ≤ **5 MB** on ≥ **4 of last 14 days** |
| Idle RDS instances | **0 connections for 7 consecutive days** |
| Underutilized EBS volumes | Unattached, or < **1 IOPS/day for 7 days** |
| Unassociated Elastic IPs | Allocated but not attached (AWS now charges hourly for these) |
| Incomplete S3 multipart uploads | Abandoned upload fragments accruing storage cost (fix: lifecycle abort rule) |
| Idle Load Balancers, unused RI expirations, etc. | Various |

**Cost Optimization Hub integration:** 16 advanced checks are inherited from the Hub — making Trusted Advisor **commitment-aware** (cross-references your RIs/SPs before recommending, e.g., t3/m5 → Graviton moves with true net savings).

**Support-tier gating:**
- **Basic/Developer (free):** Service Quotas checks (80% limit alerts) + core Security checks only. **No cost checks.**
- **Business/Enterprise:** Full suite of **100+ checks**, Trusted Advisor API access, and priority refresh. Cost data refreshes on a **weekly** cycle (not real-time).

**Automation:** Trusted Advisor → **EventBridge** (status change OK→WARN/ERROR) → **Lambda / SSM** → auto-remediate (delete unattached EBS, release stray EIP) or alert in Slack.

---

### 4.10 AWS Cost Optimization Hub

**What:** A single, org-wide console that **aggregates and deduplicates** all optimization recommendations — rightsizing, idle resources, Graviton migration, RI/SP purchases, upgrades — from Compute Optimizer, Cost Explorer, and Trusted Advisor, quantified in one currency: **estimated monthly savings**.

**Why:** Instead of checking 3+ consoles across dozens of accounts, FinOps teams get one prioritized, filterable savings backlog (by account/region/tag), with dedupe logic so the same instance isn't counted twice.

**How:** Opt in (free) from Billing Console → Cost Optimization Hub; enable org-wide from the management account.

---

### 4.11 Savings Plans & Reserved Instances (Commitment Models)

**What:** Discount mechanisms in exchange for a 1- or 3-year commitment.

| Model | Commit To | Flexibility | Typical Discount |
|---|---|---|---|
| **Compute Savings Plan** | $/hour of compute | Any instance family, region, OS, EC2/Fargate/Lambda | up to ~66% |
| **EC2 Instance Savings Plan** | $/hour in a family+region | Size/OS/AZ flexible within family | up to ~72% |
| **Standard RI** | Specific instance attributes | Size-flexible (regional, Linux); sellable on RI Marketplace | up to ~72% |
| **Convertible RI** | Exchangeable attributes | Can exchange family/OS | up to ~66% |

**Why:** Steady-state workloads shouldn't pay On-Demand. Utilization and coverage reports in Cost Explorer tell you when and how much to commit.

**How:** Use Cost Explorer's recommendations (7/30/60-day lookback) → purchase → monitor **Utilization** (are commitments used?) and **Coverage** (is enough usage discounted?) monthly. Payment options: All Upfront / Partial Upfront / No Upfront (affects amortized view).

---

### 4.12 AWS Service Quotas

**What:** Central console + API to view/manage the maximum limits (quotas) of resources and API rates per service, per region.

**Why:** Hitting a quota mid-scale-out = production provisioning failures and throttling errors. Quotas are a *capacity-planning* concern, and quota alerting is part of cost/ops governance.

**Key mechanics:**
- **Default quota** (AWS hardcoded) vs **Applied quota** (your granted increase).
- **Regional scope** — most quotas are per-region; DR/multi-region plans need increases in *every* region.
- **Adjustable vs non-adjustable** — some are hard design limits (e.g., IAM policy size, 5 IGWs/VPC... actually 1 IGW per VPC — many structural limits are fixed).
- **Utilization graphs** + **CloudWatch alarms** directly from the console (common pattern: alarm at 80%).

**Automatic Management modes:**
- `NotifyOnly` — AWS scans usage; at **80%/95%** utilization, fires recurring alerts (every 24h) to AWS Health, User Notifications, EventBridge/chat.
- `NotifyAndAdjust` — additionally **auto-submits quota increase requests** on your behalf.

**Org scale:** **Quota Request Templates** in the management account auto-apply a bundle of increases to every new member account (pairs well with Control Tower).

---

### 4.13 Consolidated Billing & AWS Organizations

**What:** One management (payer) account receives a single bill for all member accounts.

**Why & benefits:**
- **One invoice**, per-account cost visibility (Linked Account dimension).
- **Volume discount sharing** — usage tiers (e.g., S3 pricing tiers) aggregate across accounts.
- **RI & Savings Plan discount sharing** — unused commitment discounts float to other accounts (can be disabled per account).
- Central place for CUR, Cost Explorer, Budgets, tag/SCP policies.

---

### 4.14 AWS Billing Conductor

**What:** A layer that lets you create **custom pro-forma billing views** — group accounts into *billing groups*, apply custom pricing rules (markups/discounts), and produce alternate CURs per group.

**Why:** Resellers, MSPs, and enterprises doing internal chargeback often need to show teams a *rate card* different from AWS list prices — without changing the real AWS bill.

---

### 4.15 Free Tier Tracking, Credits & Billing Console Essentials

- **Free Tier usage alerts** — automatic emails at 85% of a free-tier limit; visible in the Billing console Free Tier page.
- **Credits** — applied automatically; visible in Billing → Credits; consumption order can matter for shared org credits.
- **Payment methods, invoices, tax settings, purchase orders** — all managed in the Billing console.
- **CUDOS / Cloud Intelligence Dashboards** — pre-built QuickSight dashboards (deployable from the Billing console) giving executive-ready cost visuals (see Lab 6).

---

## 5. Step-by-Step Configuration & Implementation Guide (Summary)

> Full click-by-click detail lives in [`hands-on-labs.md`](hands-on-labs.md). This is the recommended rollout order:

| Phase | Action | Outcome |
|---|---|---|
| 1 | Enable IAM billing access + Cost Explorer | Visibility unlocked (24h data lag) |
| 2 | Define tagging schema → tag resources → **activate** cost allocation tags | Business-dimension slicing |
| 3 | Create Cost Categories (+ split charges) | Org hierarchy in billing |
| 4 | Create Budgets (actual + forecast) → SNS/Chatbot alerts → optional Budget Actions | Proactive guardrails |
| 5 | Enable Cost Anomaly Detection monitors | Spike detection |
| 6 | Configure CUR 2.0 Data Export → S3 → Athena → QuickSight/CUDOS | Deep analytics pipeline |
| 7 | Opt in Compute Optimizer (org-wide) + CloudWatch Agent for memory | Rightsizing intelligence |
| 8 | Review Trusted Advisor + wire EventBridge auto-remediation | Waste cleanup automation |
| 9 | Opt in Cost Optimization Hub | Single savings backlog |
| 10 | Purchase Savings Plans/RIs from recommendations; track utilization/coverage | Rate optimization |
| 11 | Set Service Quotas alarms / Automatic Management + org templates | Scale readiness |

---

## 6. How to Use & Where to Use (Target Use Cases)

| Use Case | Tools Involved |
|---|---|
| **Monthly spend review for leadership** | Cost Explorer (monthly, group by Service/Account), CUDOS dashboard |
| **Team/project chargeback & showback** | Cost Allocation Tags + Cost Categories + CUR + Athena |
| **Kubernetes/microservice cost per pod** | CUR Split Cost Allocation (EKS/ECS) |
| **"Why did the bill spike yesterday?"** | Cost Anomaly Detection, Cost Explorer daily + Usage Type, Amazon Q analysis |
| **Prevent a dev sandbox from overspending** | Scoped Budget (tag filter) + Budget Action (deny RunInstances) |
| **Quarterly optimization sprint** | Cost Optimization Hub → Compute Optimizer → change tickets (API → Jira) |
| **Commitment strategy (RI/SP)** | Cost Explorer recommendations + Utilization/Coverage reports |
| **MSP/reseller custom rate cards** | Billing Conductor |
| **Multi-cloud FinOps standardization** | FOCUS 1.0 Data Exports |
| **Pre-launch capacity planning** | Service Quotas utilization + increase requests/templates |
| **Continuous hygiene (idle EBS/EIP cleanup)** | Trusted Advisor → EventBridge → Lambda/SSM |

---

## 7. Pricing Notes (Tools Themselves)

| Tool | Cost |
|---|---|
| Cost Explorer UI | Free |
| Cost Explorer **API** | **$0.01 per paginated request** |
| Hourly granularity / resource-level in CE | Paid add-on |
| CUR / Data Exports | Free (pay S3 storage + Athena queries) |
| Budgets | First 2 free; then ~$0.02/day per budget; actions-enabled budgets billed |
| Cost Anomaly Detection | Free (SNS charges apply) |
| Compute Optimizer | Free (Enhanced Infrastructure Metrics is paid) |
| Trusted Advisor full checks | Requires Business/Enterprise Support |
| Cost Optimization Hub / Service Quotas / Cost Categories | Free |

---

## 8. Best Practices Checklist

- [ ] Activate IAM billing access; never operate as root.
- [ ] Enforce a **case-consistent tagging schema** (Tag Policies + SCPs / IaC defaults).
- [ ] Activate cost allocation tags **early** (they're not retroactive).
- [ ] Always keep at least: 1 overall forecast budget + per-environment scoped budgets.
- [ ] Create an anomaly monitor on **AWS Services** dimension on day one (it's free).
- [ ] Ship CUR 2.0 (Parquet) to S3 from the start — history you don't export is history you lose after 13 months.
- [ ] Install CloudWatch Agent for memory metrics before trusting rightsizing.
- [ ] Review Utilization ≥ 90% before buying more commitments; review Coverage to decide *when* to buy.
- [ ] Automate hygiene: Trusted Advisor + EventBridge for EIP/EBS/idle cleanup.
- [ ] Re-run the optimization loop monthly — FinOps is a cycle (Inform → Optimize → Operate), not a one-time project.

---

## 📚 Next Steps

- ➡️ Run the labs: [`hands-on-labs.md`](hands-on-labs.md)
- ➡️ Keep the CLI reference handy: [`commands-cheatsheet.md`](commands-cheatsheet.md)
- ➡️ Something broken? [`troubleshooting.md`](troubleshooting.md)

---

*Maintained as a practical FinOps learning project. PRs and suggestions welcome.*
