# AWS Billing & Cost Management — Complete FinOps Learning Guide

> **A practical, end-to-end guide to AWS Billing, Cost Management, and FinOps** — covering every native cost tool AWS offers, how they fit together, and how to implement cost governance from a single account to a multi-account enterprise organization.

---

## 📚 Repository Structure

| File | Purpose |
|------|---------|
| [`README.md`](./README.md) | Concepts, architecture, service deep-dives, configuration guide, use cases |
| [`commands-cheatsheet.md`](./commands-cheatsheet.md) | All AWS CLI commands for billing, budgets, Cost Explorer, CUR, Savings Plans & more |
| [`hands-on-labs.md`](./hands-on-labs.md) | 12 step-by-step labs from zero to a full FinOps setup |
| [`troubleshooting.md`](./troubleshooting.md) | Common errors, root causes, and fixes |

---

## Table of Contents

1. [What is AWS Billing & Cost Management? (What / Why / How)](#1-what-is-aws-billing--cost-management-what--why--how)
2. [What is FinOps?](#2-what-is-finops)
3. [Prerequisites](#3-prerequisites)
4. [High-Level Architecture & Service Flow](#4-high-level-architecture--service-flow)
5. [Core Features & Deep-Dive](#5-core-features--deep-dive)
6. [Step-by-Step Configuration & Implementation Guide](#6-step-by-step-configuration--implementation-guide)
7. [How to Use & Where to Use (Target Use Cases)](#7-how-to-use--where-to-use-target-use-cases)
8. [Cost Optimization Strategies](#8-cost-optimization-strategies)
9. [FinOps Best Practices & KPIs](#9-finops-best-practices--kpis)
10. [Pricing of the Cost Tools Themselves](#10-pricing-of-the-cost-tools-themselves)
11. [Quick Reference — Which Tool for Which Job?](#11-quick-reference--which-tool-for-which-job)

---

## 1. What is AWS Billing & Cost Management? (What / Why / How)

### 🔹 WHAT

**AWS Billing & Cost Management** is the suite of native AWS services and console pages used to:

- **See** what you are spending (visibility) — bills, invoices, Cost Explorer, Cost & Usage Reports
- **Control** what you spend (governance) — budgets, budget actions, service control policies, anomaly detection
- **Reduce** what you spend (optimization) — Savings Plans, Reserved Instances, rightsizing, Compute Optimizer, Trusted Advisor
- **Allocate** spend to the right owners (accountability) — cost allocation tags, cost categories, consolidated billing, Billing Conductor

### 🔹 WHY

Cloud spend is **variable, decentralized, and easy to lose control of**:

- Any engineer can launch resources → spend is no longer gated by a procurement team
- Pay-as-you-go means a forgotten `p4d.24xlarge` or an unbounded S3 bucket silently burns money
- Finance teams need to map cloud spend to products, teams, and customers for chargeback/showback
- Organizations typically waste **25–35% of cloud spend** on idle, oversized, or unmanaged resources

Without a billing and cost management practice: surprise bills, no accountability, no forecasting, and no ability to answer *"what does product X cost us per month?"*

### 🔹 HOW

AWS solves this with a layered toolset:

```
VISIBILITY  →  Cost Explorer, CUR/Data Exports, Billing Console, Free Tier usage alerts
CONTROL     →  Budgets, Budget Actions, Cost Anomaly Detection, SCPs, IAM billing policies
OPTIMIZE    →  Savings Plans, Reserved Instances, Spot, Compute Optimizer, Trusted Advisor, S3 lifecycle
ALLOCATE    →  Cost Allocation Tags, Cost Categories, AWS Organizations consolidated billing, Billing Conductor
```

---

## 2. What is FinOps?

**FinOps (Cloud Financial Operations)** is an operational framework and cultural practice that brings **engineering, finance, and business teams together** to maximize the business value of cloud spend.

### The Three FinOps Phases (FinOps Foundation Framework)

| Phase | Goal | AWS Tools Used |
|-------|------|----------------|
| **1. Inform** | Visibility, allocation, benchmarking, budgeting, forecasting | Cost Explorer, CUR, Cost Categories, Tags, Budgets |
| **2. Optimize** | Rightsizing, commitment discounts, waste elimination | Savings Plans, RIs, Compute Optimizer, Trusted Advisor, Spot |
| **3. Operate** | Continuous governance, automation, culture, unit economics | Budget Actions, Anomaly Detection, SCPs, automation (Lambda/EventBridge) |

### FinOps Core Principles

1. **Teams need to collaborate** — finance + engineering + product
2. **Everyone takes ownership of their cloud usage** — decentralized accountability
3. **A centralized team drives FinOps** — rates/commitments managed centrally, usage managed by teams
4. **Reports should be accessible and timely** — near-real-time visibility beats month-end surprises
5. **Decisions are driven by business value** — unit economics (cost per customer/transaction), not just totals
6. **Take advantage of the variable cost model** — treat cloud as an opportunity, not a fixed cost

---

## 3. Prerequisites

| Requirement | Details |
|-------------|---------|
| **AWS Account** | Free tier is fine for most labs; management account of an AWS Organization for consolidated billing labs |
| **IAM User/Role (not root)** | Root only needed once — to activate **IAM access to billing** and for a few account-level settings |
| **AWS CLI v2** | `aws --version` → 2.x; configured via `aws configure` or SSO |
| **Billing IAM permissions** | Policies such as `Billing` (AWS-managed job function), `AWSBudgetsActionsWithAWSResourceControlAccess`, `ce:*`, `cur:*` |
| **Basic AWS knowledge** | EC2, S3, IAM fundamentals |
| **Optional** | Athena + QuickSight (for CUR analysis lab), SNS (for alerts), an AWS Organization (for multi-account labs) |

> ⚠️ **Important IAM note:** By default, IAM users **cannot** see billing data even with `AdministratorAccess`. The root user must first enable **"Activate IAM Access"** on the Account settings page. This is covered in Lab 1.

---

## 4. High-Level Architecture & Service Flow

### 4.1 How billing data flows inside AWS

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RESOURCE USAGE (all AWS services)                     │
│         EC2 · S3 · RDS · Lambda · Data Transfer · EBS · CloudWatch ...      │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │  metering records (usage + rates)
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AWS METERING & BILLING PIPELINE                        │
│   • Applies pricing (On-Demand, Savings Plans, RIs, Spot, tiered rates)     │
│   • Applies credits, refunds, taxes                                         │
│   • Aggregates per account → per payer (consolidated billing)               │
└───────┬───────────────┬────────────────┬───────────────────┬────────────────┘
        │               │                │                   │
        ▼               ▼                ▼                   ▼
┌──────────────┐ ┌──────────────┐ ┌────────────────┐ ┌─────────────────────┐
│ Billing      │ │ Cost         │ │ Data Exports   │ │ Budgets & Cost      │
│ Console      │ │ Explorer     │ │ (CUR 2.0)      │ │ Anomaly Detection   │
│ (Bills,      │ │ (visual      │ │ → S3 bucket    │ │ (thresholds & ML    │
│ Invoices,    │ │ analysis,    │ │ → Athena /     │ │ monitors)           │
│ Payments)    │ │ forecast)    │ │   QuickSight   │ └──────────┬──────────┘
└──────────────┘ └──────────────┘ └────────────────┘            │ alerts/actions
                                                                ▼
                                              ┌────────────────────────────────┐
                                              │ SNS → Email / Slack / Lambda   │
                                              │ Budget Actions → stop EC2/RDS, │
                                              │ apply IAM/SCP deny policies    │
                                              └────────────────────────────────┘
```

### 4.2 Multi-account (AWS Organizations) billing flow

```
                    ┌───────────────────────────────┐
                    │   MANAGEMENT (PAYER) ACCOUNT  │
                    │  • Receives ONE consolidated  │
                    │    invoice                    │
                    │  • Owns CUR, Cost Categories, │
                    │    tag activation, SPs/RIs    │
                    └───────────────┬───────────────┘
              consolidated billing  │  discount & RI/SP sharing
        ┌───────────────┬───────────┴────────┬────────────────┐
        ▼               ▼                    ▼                ▼
  ┌───────────┐   ┌───────────┐       ┌───────────┐    ┌───────────┐
  │ Member    │   │ Member    │       │ Member    │    │ Member    │
  │ Acct: Dev │   │ Acct: QA  │       │ Acct: Prod│    │ Acct: Sec │
  └───────────┘   └───────────┘       └───────────┘    └───────────┘

  Benefits: one invoice · volume-tier aggregation (e.g., S3 tiers) ·
  RI/Savings Plan discount sharing across accounts · per-account visibility
```

### 4.3 A reference FinOps reporting pipeline (what you build in Lab 8)

```
Data Exports (CUR 2.0, Parquet) ──► S3 bucket ──► AWS Glue table ──► Athena SQL
                                                                        │
                                                                        ▼
                                                     QuickSight dashboards / CUDOS
```

---

## 5. Core Features & Deep-Dive

Every tool below follows **What → Why → How**.

### 5.1 Billing Console (Bills, Invoices, Payments)

- **What:** The home page of spend truth: monthly **Bills** page (charges by service/account/region), **Invoices**, **Payments**, **Credits**, **Purchase Orders**, **Tax settings**, **Payment methods**, **Payment preferences** (currency, billing address).
- **Why:** Legal/financial record of what AWS charged you; where finance reconciles invoices.
- **How:** Console → **Billing and Cost Management** → *Bills*. Data is final after the month closes (invoice issued in the first days of the following month). Mid-month figures are estimates.
- **Key details:**
  - Charges are shown **by service → region → usage type**.
  - **Credits** are applied automatically before payment.
  - You can pay in local currency for supported currencies (set in Payment preferences).
  - **Purchase orders** can be attached so invoices reference your PO numbers.

### 5.2 AWS Free Tier & Free Tier Alerts

- **What:** Three free-tier types: **Always free** (e.g., 1M Lambda requests/month), **12-months free** (e.g., 750 hrs t2/t3.micro), **Trials** (short-term). Free Tier usage alerts email you when you reach **85%** of a free-tier limit.
- **Why:** #1 cause of "surprise first bill" is silently exceeding free tier.
- **How:** Billing console → *Billing preferences* → enable **Free Tier usage alerts**; track usage at Billing → *Free Tier* page.

### 5.3 Billing Preferences

- **What:** Account-level toggles: **PDF invoice delivery by email**, **AWS Free Tier alerts**, **CloudWatch billing alerts** (legacy metric `EstimatedCharges` in **us-east-1**).
- **Why:** Enabling *Receive Billing Alerts* is required before the legacy CloudWatch billing alarm works.
- **How:** Billing console → *Billing preferences* → enable and save. (Modern practice: prefer **AWS Budgets** over the legacy CloudWatch billing alarm, but know both for interviews.)

### 5.4 IAM Access to Billing (critical gotcha)

- **What:** A root-only switch — **"Activate IAM Access"** — that allows IAM users/roles to view billing pages *if* they also have billing IAM permissions.
- **Why:** Without it, even `AdministratorAccess` users get *"You need permissions"* on billing pages.
- **How:** Sign in as **root** → Account page → *IAM user and role access to Billing information* → Edit → Activate → Update. Then attach IAM policies (`Billing` job-function policy, or fine-grained actions like `aws-portal:ViewBilling` / the newer `billing:*`, `ce:*`, `budgets:*` actions).
- **Note:** AWS migrated old `aws-portal:*` actions to fine-grained actions (`billing:*`, `payments:*`, `invoicing:*`, `tax:*`, `account:*`, `freetier:*`). Know both families.

### 5.5 AWS Cost Explorer

- **What:** Interactive visualization tool for cost & usage: filter/group by **service, account, region, tag, cost category, usage type, purchase option**, etc. Includes **forecasting** (up to 12 months), **hourly & resource-level granularity** (opt-in, extra cost), and **RI/Savings Plans recommendations & utilization/coverage reports**.
- **Why:** Fastest way to answer *"why did my bill go up?"* without writing SQL.
- **How:** Billing console → **Cost Explorer** → enable (first enablement takes ~24 hours to populate; then refreshes at least once every 24 hours). Data available for the **last 13 months** (up to 38 months opt-in via multi-year data at monthly granularity).
- **Key concepts:**
  - **Unblended cost** – what you're billed per account line item (default)
  - **Amortized cost** – spreads RI/SP upfront fees across the term (best for trend analysis)
  - **Blended cost** – averaged rates across an organization (legacy accounting view)
  - **Net costs** – after discounts/credits
  - **API:** `aws ce get-cost-and-usage` — **$0.01 per paginated API request** (console usage is free).

### 5.6 AWS Budgets

- **What:** Threshold-based guardrails. Four budget types:
  1. **Cost budget** — alert when spend (actual or forecasted) exceeds $X
  2. **Usage budget** — alert on usage quantities (e.g., EC2 hours)
  3. **RI / Savings Plans utilization budget** — alert when utilization drops below N%
  4. **RI / Savings Plans coverage budget** — alert when coverage drops below N%
- **Why:** Proactive control — you find out *before* the invoice, and forecasted alerts warn you *before* you even cross the line.
- **How:** Billing console → *Budgets* → Create. Choose period (daily/monthly/quarterly/annual), scope (filters: service, account, tag…), thresholds (up to 5 alerts per budget), notify via **email (up to 10 addresses)** and/or **SNS topic** (→ Slack/Lambda/ChatOps).
- **Limits & pricing:** First **2 action-enabled budgets free**; ~**$0.10/day** per action-enabled budget beyond that; plain alert budgets are free (soft limit ~20,000 budgets/account). Budget data refreshes ~3x/day.

### 5.7 AWS Budget Actions

- **What:** Budgets that **do something** when a threshold is hit, not just alert:
  - Apply an **IAM policy** (e.g., deny `ec2:RunInstances`) to users/roles
  - Attach an **SCP** to an OU/account (Organizations)
  - **Stop EC2 or RDS instances** in a region
- **Why:** Converts monitoring into enforcement — essential for sandbox/dev accounts.
- **How:** Create budget → *Configure actions* → choose action type + IAM execution role → run **automatically** or **require workflow approval**.

### 5.8 AWS Cost Anomaly Detection

- **What:** **Machine-learning** based monitor that learns your spend patterns and flags unusual spikes with a **root-cause analysis** (which service/account/region/usage type caused it).
- **Why:** Budgets catch *known* thresholds; anomaly detection catches the *unknown unknowns* (a leaked key mining crypto, a runaway Lambda loop) — even when total spend is still under budget.
- **How:** Cost Management → *Cost Anomaly Detection* → create a **cost monitor** (AWS services / linked account / cost category / tag) → create an **alert subscription** (individual alerts via SNS in near-real-time, or daily/weekly email summaries; set an alert threshold in $ or %). It needs ~10 days of history to calibrate. **Free** (you pay only for SNS).

### 5.9 Cost Allocation Tags

- **What:** Tags (key/value pairs) that, once **activated** for cost allocation, appear as columns in CUR and as filters in Cost Explorer. Two kinds:
  - **AWS-generated tags** — e.g., `aws:createdBy` (who created the resource)
  - **User-defined tags** — e.g., `Project=Phoenix`, `Environment=prod`, `CostCenter=1234`
- **Why:** The foundation of allocation/chargeback — you cannot split a bill by team without tags.
- **How:** Tag resources (console/CLI/IaC) → Billing console → *Cost allocation tags* → select keys → **Activate**. Takes up to **24 hours** to appear; **not retroactive** (only usage after activation is tagged in reports — backfill of up to 12 months can be requested via the backfill feature). Enforce tagging with **Tag Policies** (Organizations) and IaC standards.

### 5.10 Cost Categories

- **What:** Rule-based virtual grouping of costs into your own business dimensions — e.g., map accounts+tags+services into categories like `Team: Payments`, `Team: Platform`, `Shared`. Supports **split charge rules** to distribute shared costs (proportional, fixed %, or even split).
- **Why:** Tags are messy and incomplete; cost categories let a central team impose a clean business-aligned hierarchy on top, including allocating shared platform costs back to teams.
- **How:** Cost Management → *Cost categories* → create category → define ordered rules (first match wins) → optionally add split charges. Categories appear in Cost Explorer, Budgets, CUR.

### 5.11 Cost & Usage Report (CUR) / Data Exports (CUR 2.0)

- **What:** The **most granular billing dataset AWS produces** — every line item (hourly/daily/monthly granularity, optional **resource IDs**), delivered to **your S3 bucket** in Parquet/CSV. The modern interface is **Data Exports (CUR 2.0)** with a SQL-selectable schema; **legacy CUR** is the classic report. Also supports **Cost Optimization Hub exports** and **carbon emissions exports**.
- **Why:** Source of truth for FinOps tooling, custom dashboards (Athena/QuickSight/CUDOS), chargeback engines, and third-party platforms (CloudHealth, Cloudability, etc.).
- **How:** Billing console → **Data Exports** → Create export → choose CUR 2.0 → select columns, granularity, refresh (report is updated multiple times a day and restated as the month finalizes) → target S3 bucket (bucket policy auto-applied) → query with Athena. **Report itself is free; you pay S3 storage + Athena queries.**

### 5.12 Consolidated Billing (AWS Organizations)

- **What:** One **management (payer) account** pays for all **member accounts**. Features: **single invoice**, **combined volume tiers** (usage aggregates across accounts to hit cheaper tiers faster), **RI & Savings Plans discount sharing** (unused discounts float to other accounts), per-account cost breakdown.
- **Why:** Standard enterprise pattern — account-per-team/environment isolation *without* invoice sprawl, plus discount efficiency.
- **How:** Create an Organization (management account) → invite/create member accounts. Toggle **RI/SP discount sharing** per account under Billing preferences if you need to disable sharing. Management account should be kept resource-free (security best practice).

### 5.13 AWS Billing Conductor

- **What:** A **custom "pro forma" billing layer** for organizations that resell or charge back AWS: create **billing groups**, apply custom **pricing rules** (markups/discounts), custom credits/fees, and generate a second, customized version of the CUR per billing group.
- **Why:** MSPs and enterprises with internal chargeback often need to show teams/customers *different* rates than what AWS actually charges the payer.
- **How:** Billing Conductor console → create billing groups → pricing plans/rules → pro forma CUR per group. (Priced per pro-forma-billed account.)

### 5.14 Savings Plans

- **What:** Commitment-based discount (up to **~72%** off On-Demand): commit to **$X/hour for 1 or 3 years**. Types:
  - **Compute Savings Plans** — most flexible: any EC2 family/region/OS/tenancy + **Fargate + Lambda** (up to ~66%)
  - **EC2 Instance Savings Plans** — locked to an instance *family in a region* (e.g., m5 in us-east-1), deepest EC2 discount (up to ~72%)
  - **SageMaker Savings Plans** — for SageMaker usage
- **Payment options:** All Upfront > Partial Upfront > No Upfront (discount depth in that order).
- **Why:** For steady-state baseline compute, paying On-Demand is pure waste.
- **How:** Cost Management → *Savings Plans* → **Recommendations** (based on last 7/30/60 days) → purchase; then monitor **Utilization** (are you using what you committed?) and **Coverage** (what % of eligible spend is covered?) and alert on both via Budgets.

### 5.15 Reserved Instances (RIs)

- **What:** The older commitment model — reserve a specific instance configuration for 1/3 years.
  - **Standard RI** — up to ~72% off; can modify (AZ, size within family via size flexibility for regional Linux/shared tenancy) but **not exchange** family; sellable on the **RI Marketplace**
  - **Convertible RI** — up to ~66% off; can **exchange** for different family/OS/tenancy of equal or greater value; not sellable
  - **Zonal RI** — scoped to an AZ; provides a **capacity reservation**; Regional RIs provide discount + size flexibility but no capacity guarantee
- **Also RIs for:** RDS, ElastiCache, Redshift, OpenSearch, DynamoDB (reserved capacity).
- **Why vs Savings Plans:** SPs are simpler and generally preferred for compute; RIs still matter for **RDS and other services SPs don't cover**, and for capacity reservations.
- **How:** EC2/RDS console → Reserved Instances → purchase; or via Cost Explorer RI recommendations.

### 5.16 Spot Instances (cost lever, honorable mention)

- **What:** Spare EC2 capacity at up to **90% discount**; can be interrupted with a 2-minute warning.
- **Why:** Cheapest compute for fault-tolerant, stateless, batch, CI/CD, big-data workloads.
- **How:** ASG mixed-instances policy, EMR/EKS/Batch integrations, Spot Fleet. Not a Billing-console feature but a core FinOps lever.

### 5.17 AWS Compute Optimizer

- **What:** ML-based **rightsizing recommendations** for EC2, ASGs, EBS volumes, Lambda memory, ECS on Fargate, RDS — classifies resources as under/over-provisioned with projected savings. Enhanced metrics with CloudWatch memory agent.
- **Why:** Data-backed downsizing beats guessing; typically the biggest "quick win" after commitments.
- **How:** Opt in (account or organization level) → review recommendations → apply during change windows. Free tier of recommendations; enhanced infrastructure metrics is a paid opt-in.

### 5.18 AWS Trusted Advisor (Cost Optimization checks)

- **What:** Best-practice checker; the **Cost Optimization** pillar flags idle load balancers, low-utilization EC2, unassociated Elastic IPs, underutilized EBS, idle RDS, etc.
- **Why:** Fast waste inventory across the account.
- **How:** Trusted Advisor console. Core checks are free; **full cost checks require Business/Enterprise Support**.

### 5.19 Cost Optimization Hub

- **What:** A newer consolidated dashboard that aggregates **all optimization recommendations** (rightsizing, idle resources, Savings Plans/RI purchases, upgrades to Graviton, etc.) across accounts, deduplicated, with quantified savings.
- **Why:** One prioritized list instead of hopping between Compute Optimizer, Trusted Advisor, and Cost Explorer.
- **How:** Cost Management console → Cost Optimization Hub → opt in (free).

### 5.20 AWS Pricing Calculator & Pricing APIs

- **What:** [calculator.aws](https://calculator.aws) — estimate costs of architectures *before* building; shareable estimate links. Programmatic pricing via the **Price List API** (`aws pricing`).
- **Why:** Forecasting new workloads, comparing configurations, producing client estimates.
- **How:** Add services → configure → export/share estimate.

### 5.21 CloudWatch Billing Alarm (legacy but exam-relevant)

- **What:** CloudWatch alarm on the `AWS/Billing` metric `EstimatedCharges` (only in **us-east-1**, only if *Receive Billing Alerts* preference is enabled).
- **Why:** Predecessor of Budgets; still appears in interviews/exams.
- **How:** Enable billing alerts preference → CloudWatch (us-east-1) → Alarms → Billing → threshold → SNS.

### 5.22 Service Control Policies (SCPs) as cost guardrails

- **What:** Organization-level deny policies — e.g., deny expensive instance types, deny regions, deny leaving the org.
- **Why:** Prevention beats detection: block `ec2:RunInstances` for `*.24xlarge` in dev OUs entirely.
- **How:** Organizations console → Policies → SCP → attach to OU.

### 5.23 Reserved Instance Marketplace, Credits, Refunds

- **RI Marketplace:** sell unused **Standard** RIs (US bank account required) — exit hatch for over-commitment.
- **Credits:** promotional credits auto-apply to eligible usage before payment; shared across an org by default (can be disabled).
- **Refunds:** most usage is non-refundable; some cases (accidental RI purchase within policy, duplicate payments) handled via AWS Support.

---

## 6. Step-by-Step Configuration & Implementation Guide

This is the **recommended implementation order** for a real environment. Each step maps to a lab in [`hands-on-labs.md`](./hands-on-labs.md).

| # | Step | Tool | Lab |
|---|------|------|-----|
| 1 | Enable IAM access to billing; create FinOps IAM role/policies | Account settings, IAM | Lab 1 |
| 2 | Set billing preferences: PDF invoices, Free Tier alerts, billing alerts | Billing preferences | Lab 1 |
| 3 | Enable Cost Explorer (wait ~24h for first data) | Cost Explorer | Lab 3 |
| 4 | Define & enforce tagging standard; activate cost allocation tags | Tags, Tag Policies | Lab 2 |
| 5 | Build Cost Categories mapping accounts/tags → teams; add split charges | Cost Categories | Lab 2 |
| 6 | Create budgets: total monthly cost (actual + forecast), per-team, usage | Budgets + SNS | Lab 4 |
| 7 | Add budget actions for sandbox accounts (deny/stop) | Budget Actions | Lab 5 |
| 8 | Enable Cost Anomaly Detection monitors + SNS alerting | Anomaly Detection | Lab 6 |
| 9 | Create Data Export (CUR 2.0) → S3 → Athena → QuickSight/CUDOS | Data Exports | Lab 8 |
| 10 | Set up Organizations consolidated billing (multi-account) | Organizations | Lab 7 |
| 11 | Opt in to Compute Optimizer & Cost Optimization Hub; review Trusted Advisor | Optimizers | Lab 9 |
| 12 | Analyze SP/RI recommendations; purchase; add utilization/coverage budgets | Savings Plans/RIs | Lab 10 |
| 13 | Automate: EventBridge schedules (stop dev at night), lifecycle policies | Automation | Lab 11 |
| 14 | Establish monthly FinOps review: anomalies, top movers, KPIs, actions | Process | §9 |

---

## 7. How to Use & Where to Use (Target Use Cases)

| Scenario | What to use | Why |
|----------|-------------|-----|
| **Personal/learning account — avoid surprise bills** | Free Tier alerts + a $5–10 monthly cost budget (actual+forecast) + anomaly monitor | Zero-cost safety net |
| **Startup — single account, small team** | Cost Explorer + budgets per environment tag + anomaly detection + Compute SP after 2–3 months of steady usage | Simple, low-overhead |
| **Enterprise — 100s of accounts** | Organizations consolidated billing + Cost Categories + CUR→Athena→QuickSight + centralized SP/RI purchasing + SCP guardrails + Billing Conductor for internal chargeback | Scale + governance |
| **MSP / reseller** | Billing Conductor pro forma billing + per-customer billing groups + custom pricing rules | Customer-specific invoices |
| **"Why did the bill spike yesterday?"** | Cost Anomaly Detection root cause → Cost Explorer daily granularity, group by service → usage type → resource (hourly/resource opt-in) → CUR line items | Drill-down path from alert to line item |
| **Chargeback/showback to teams** | Tags + Cost Categories (+ split charges) + monthly CUR-driven reports | Business-aligned allocation |
| **Sandbox/dev cost enforcement** | Budget Actions (deny RunInstances / stop EC2), SCPs denying big instance types, EventBridge night-time shutdowns | Enforcement, not just alerts |
| **Capacity planning / forecasting** | Cost Explorer forecast, Pricing Calculator for new workloads, historical CUR trend in Athena | Forward-looking view |
| **Interview prep (SA/DevOps)** | Everything above + know SP vs RI trade-offs, unblended vs amortized, consolidated billing benefits | Frequently asked |

---

## 8. Cost Optimization Strategies

The classic optimization order (biggest, safest wins first):

1. **Delete waste** — unattached EBS volumes, old snapshots, idle ELBs, unassociated EIPs, orphaned NAT gateways, stale S3 multipart uploads
2. **Stop what's idle** — dev/test off nights & weekends (≈65–70% saving on those resources)
3. **Rightsize** — Compute Optimizer recommendations; downsize before committing
4. **Storage tiering** — S3 Intelligent-Tiering / lifecycle to IA/Glacier; gp2 → **gp3** (≈20% cheaper); EBS snapshot archive
5. **Commit** — Savings Plans / RIs on the *post-rightsizing* baseline (target 70–80% coverage, keep headroom On-Demand)
6. **Spot** — for interruption-tolerant workloads
7. **Modernize** — Graviton (arm64, ~20–40% better price/perf), serverless where bursty, Aurora Serverless v2 for spiky DBs
8. **Network** — VPC endpoints to cut NAT gateway processing, CloudFront for egress-heavy apps, same-AZ traffic where possible
9. **Observability costs** — CloudWatch log retention policies, metric/log filtering (a frequently ignored line item)

---

## 9. FinOps Best Practices & KPIs

### Best practices

- **Tag from day one**; enforce with Tag Policies + IaC linting; treat untagged spend as a KPI to drive to zero
- **Budgets on every account**, forecast-based alerts at 80/100/120%
- **Anomaly monitors** on every high-spend service and every linked account
- **Centralize commitments** (SP/RI) in the payer account; decentralize usage accountability
- **Amortized view** for trend/chargeback reporting; unblended for invoice reconciliation
- **Monthly FinOps review**: top 10 cost movers, anomalies, SP utilization/coverage, optimization backlog
- **Keep the management account empty** of workloads

### KPIs worth tracking

| KPI | Formula / target |
|-----|------------------|
| **Savings Plan / RI utilization** | ≥ 95% (you're using what you bought) |
| **Coverage** | 70–80% of eligible compute (headroom for variability) |
| **Untagged / unallocated spend** | < 5% of total |
| **Waste** (idle/unattached) | Trend to ~0; Trusted Advisor flagged $ |
| **Unit economics** | Cost per customer / per transaction / per environment |
| **Forecast accuracy** | Actual vs forecast within ±5–10% |
| **Effective Savings Rate** | (On-Demand equivalent − actual) / On-Demand equivalent |

---

## 10. Pricing of the Cost Tools Themselves

| Tool | Cost |
|------|------|
| Billing console, Bills, Invoices | Free |
| Cost Explorer (console) | Free; **API $0.01/request**; hourly+resource granularity is a paid opt-in |
| Budgets | Alert-only budgets free; action-enabled: first 2 free, then ~$0.10/day each |
| Cost Anomaly Detection | Free (SNS charges apply) |
| CUR / Data Exports | Free (pay S3 storage + Athena/QuickSight usage) |
| Cost Categories, Cost Allocation Tags | Free |
| Compute Optimizer | Base free; enhanced metrics paid |
| Trusted Advisor full checks | Requires Business/Enterprise support plan |
| Cost Optimization Hub | Free |
| Billing Conductor | Per pro-forma-billed account per month |

---

## 11. Quick Reference — Which Tool for Which Job?

```
"How much did we spend, and on what?"        → Cost Explorer / Bills page
"Alert me before we overspend"                → Budgets (forecasted alerts)
"Stop resources when we overspend"            → Budget Actions
"Catch weird spikes automatically"            → Cost Anomaly Detection
"Split the bill by team/product"              → Tags + Cost Categories
"Raw line-item data for BI"                   → Data Exports (CUR 2.0) + Athena
"One invoice for many accounts"               → Organizations consolidated billing
"Custom rates for internal/external customers"→ Billing Conductor
"Cut steady-state compute cost"               → Savings Plans (or RIs for RDS etc.)
"Cheapest possible batch compute"             → Spot
"What should we downsize?"                    → Compute Optimizer / Cost Optimization Hub
"Estimate before building"                    → Pricing Calculator
"Block expensive mistakes org-wide"           → SCPs
```

---

## 🧭 Next Steps

1. Work through [`hands-on-labs.md`](./hands-on-labs.md) — Labs 1–12
2. Keep [`commands-cheatsheet.md`](./commands-cheatsheet.md) open while you work
3. When something breaks, check [`troubleshooting.md`](./troubleshooting.md)

---

*Maintained as part of my DevOps/Cloud learning series. Contributions and corrections welcome — open an issue or PR.*
