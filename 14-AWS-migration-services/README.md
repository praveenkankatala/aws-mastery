# AWS Migration — The 6 R's, End to End

> A practical, no-hand-waving guide to moving real workloads into AWS: how to decide, how to plan, how to actually do it, and how to prove it worked.

Most migration content falls into one of two traps. Either it's a slide with six words on it (`Rehost, Replatform, Refactor…`) and no idea what to do on Monday morning, or it's a 400-page enterprise framework nobody reads. This repo tries to sit in the middle: enough theory that you understand *why*, and enough command-line detail that you can run it.

Everything here has been written so a smart person who has never migrated anything can follow along, and so someone who has done it three times still finds the checklists useful.

---

## 📚 Repository Contents

| File | What's inside | Read it when |
|---|---|---|
| **README.md** (this file) | Concepts, the 6 R's (really 7), architecture, service deep-dives, full implementation guide, use cases | You're starting out or planning |
| **[commands-cheatsheet.md](commands-cheatsheet.md)** | Every CLI command you'll need, grouped by service and by task | You're at the terminal |
| **[hands-on-labs.md](hands-on-labs.md)** | 12 build-it-yourself labs, from empty account to a cut-over application | You want to actually practice |
| **[troubleshooting.md](troubleshooting.md)** | Real error messages, root causes, fixes | Something is broken at 2am |

---

## 📖 Table of Contents

1. [What "migration" actually means](#1-what-migration-actually-means)
2. [Prerequisites](#2-prerequisites)
3. [The AWS migration lifecycle: Assess → Mobilize → Migrate](#3-the-aws-migration-lifecycle)
4. [The 6 R's (and the 7th)](#4-the-6-rs-and-the-7th) — the heart of this guide
5. [How to choose an R: decision tree and scoring matrix](#5-how-to-choose-an-r)
6. [High-level architecture and service flow](#6-high-level-architecture-and-service-flow)
7. [Core AWS migration services — deep dive](#7-core-aws-migration-services--deep-dive)
8. [Step-by-step implementation guide](#8-step-by-step-implementation-guide)
9. [Landing zone and network foundation](#9-landing-zone-and-network-foundation)
10. [Data migration mechanics](#10-data-migration-mechanics)
11. [Wave planning and the migration factory](#11-wave-planning-and-the-migration-factory)
12. [Cutover strategies and rollback](#12-cutover-strategies-and-rollback)
13. [Testing and validation](#13-testing-and-validation)
14. [Security, compliance and governance](#14-security-compliance-and-governance)
15. [Cost, licensing and the business case](#15-cost-licensing-and-the-business-case)
16. [Workload-specific playbooks](#16-workload-specific-playbooks)
17. [Post-migration: operate, optimize, modernize](#17-post-migration-operate-optimize-modernize)
18. [KPIs — how you prove success](#18-kpis--how-you-prove-success)
19. [Where and when to use each approach](#19-where-and-when-to-use-each-approach)
20. [Anti-patterns](#20-anti-patterns)
21. [Glossary](#21-glossary)
22. [Further reading](#22-further-reading)

---

## 1. What "migration" actually means

A cloud migration is the process of moving applications, data, and the operational practices around them from one environment (usually a data centre, colo, or another cloud) into AWS — *without breaking the business while you do it*.

Three things make it harder than it sounds:

1. **Applications are not islands.** A "simple" app has a database, a file share, an LDAP dependency, a batch job, a hardcoded IP in a config file someone wrote in 2013, and a printer. Move one piece and you discover the other seven.
2. **Data has gravity.** Compute is easy to move. Terabytes are not. Data volume and change rate dictate your timeline more than anything else.
3. **The clock is the enemy.** Every migration has a cutover window, and most cutover windows are Saturday nights.

The 6 R's exist to answer a single question for every application in your estate: **what are we going to *do* with this thing?** Answering it consistently is what turns a chaotic project into a repeatable factory.

### Why bother migrating at all

Be honest about the driver, because the driver dictates the R you pick:

| Driver | Typical reality | Tends to push you toward |
|---|---|---|
| Data centre lease expiry / hardware EOL | Hard deadline, no flexibility | **Rehost** (speed wins) |
| Cost reduction | CFO-led, needs a business case | Rehost then **Replatform** |
| Agility / release velocity | Engineering-led | **Refactor** |
| Escaping licensing costs | Oracle, SQL Server, Windows | **Replatform** (to Aurora/PostgreSQL/Linux) |
| Resilience / DR | Audit finding or an outage that hurt | Rehost + Elastic Disaster Recovery |
| M&A divestiture | Aggressive timeline, clean separation | **Rehost / Relocate** |
| Compliance or data residency | Region-specific | Any R, with guardrails |
| Talent and toil | Ops team drowning | **Repurchase** (SaaS) |

---

## 2. Prerequisites

### Accounts and access

- An **AWS account** (a sandbox one for the labs — don't practice in prod).
- **AWS Organizations** access if you're building a multi-account landing zone.
- An IAM principal you can use with admin-ish rights in the sandbox, and least-privilege roles in real environments.
- For source environments: administrator/root on the servers you'll migrate, and vCenter credentials if you're using agentless discovery.

### Tooling on your workstation

```bash
# AWS CLI v2
aws --version                     # aws-cli/2.x.x
aws configure sso                 # or: aws configure

# Handy extras used throughout the labs
jq --version                      # JSON wrangling
session-manager-plugin --version  # shell into EC2 with no SSH keys
terraform -version                # optional, for the landing-zone lab
git --version
```

### Networking that must exist before you migrate anything

- Outbound HTTPS (443) from source servers to AWS endpoints, either over the internet, a **Site-to-Site VPN**, or **Direct Connect**.
- DNS resolution both ways (on-prem ↔ VPC) if the apps talk to each other during the transition.
- Enough **bandwidth** to replicate your data in the window you have (see [§10](#10-data-migration-mechanics) for the arithmetic).

### Knowledge you'll want

Comfortable with: EC2, VPC/subnets/route tables/security groups, IAM roles, EBS, S3, and basic Linux + Windows administration. If you can build a VPC with a public and private subnet by hand, you're ready.

### Pre-flight checklist

```
[ ] Sandbox AWS account created, billing alerts set
[ ] AWS CLI configured and `aws sts get-caller-identity` works
[ ] Region chosen (and it's the same region everywhere — mismatched regions cause 30% of migration errors)
[ ] Service quotas checked: vCPU limits, EIPs, VPCs, EBS volume count
[ ] Source inventory started (even a spreadsheet counts on day one)
[ ] Application owners identified — you will need them, more than you think
[ ] Change/approval process understood (who signs off a cutover?)
```

---

## 3. The AWS migration lifecycle

AWS organises migration into three phases. This is the **Migration Acceleration Program (MAP)** structure, and it's genuinely useful as a mental model regardless of whether you formally enrol.

```
┌───────────────────┐   ┌───────────────────┐   ┌──────────────────────────┐
│   1. ASSESS       │──▶│   2. MOBILIZE     │──▶│  3. MIGRATE & MODERNIZE  │
├───────────────────┤   ├───────────────────┤   ├──────────────────────────┤
│ • Discovery       │   │ • Landing zone    │   │ • Wave execution         │
│ • TCO / business  │   │ • Security base   │   │ • Rehost / Replatform /  │
│   case            │   │ • Ops model       │   │   Refactor per app       │
│ • Migration       │   │ • Skills & CCoE   │   │ • Test → cutover → run   │
│   readiness (MRA) │   │ • Pilot migration │   │ • Decommission source    │
│ • 6 R disposition │   │ • Migration       │   │ • Optimize & modernize   │
│                   │   │   factory build   │   │                          │
└───────────────────┘   └───────────────────┘   └──────────────────────────┘
   Weeks 0–6              Weeks 4–12              Months 3–24
```

**Assess** — Do you know what you have and what it costs? Output: an inventory, a TCO comparison, a business case, and a first-pass R disposition for every app.

**Mobilize** — Fix the gaps the assessment found. Output: a working landing zone, security guardrails, a trained team, and one or two pilot apps *actually migrated* to prove the machine works.

**Migrate & Modernize** — Run waves. Boring is the goal. Output: an empty data centre and a modernization backlog.

### Migration Readiness Assessment (MRA)

Before you migrate at scale, honestly score yourself 1–5 across the six **Cloud Adoption Framework (CAF)** perspectives. Gaps here are what kill migrations mid-flight:

| CAF perspective | Question you're really answering |
|---|---|
| **Business** | Is there a funded business case with an executive sponsor? |
| **People** | Does the team have cloud skills, and is there a target org model? |
| **Governance** | Do we have tagging, budgets, change management, portfolio management? |
| **Platform** | Is there a landing zone, network design, and reference architectures? |
| **Security** | IAM model, encryption standards, logging, incident response? |
| **Operations** | Monitoring, patching, backup, runbooks, on-call for cloud? |

---

## 4. The 6 R's (and the 7th)

The original "5 R's" came from Gartner in 2010; AWS extended them to 6, and then quietly added **Relocate** for VMware, giving us 7. People still say "6 R's" — that's the phrase everyone searches for, so that's what this repo is called.

Here they are ordered by *how much you change the application*, from "nothing" to "everything":

```
 LEAST CHANGE ◀──────────────────────────────────────────────────▶ MOST CHANGE
 LOWEST COST/RISK                                        HIGHEST COST/BENEFIT

 Retire     Retain      Relocate    Rehost     Replatform   Repurchase  Refactor
   │          │            │           │            │            │          │
 turn it   leave it     move the    lift &      lift, tweak,   buy SaaS   rebuild
   off      alone       hypervisor   shift          shift       instead    cloud-native
```

### 4.1 Retire — turn it off

**Definition:** Decommission the application. Don't migrate it at all.

**Why it's first:** In almost every real portfolio assessment, **10–20% of the estate is dead or duplicated**. Servers running a decommissioned reporting tool. Three apps that all do expense claims. A dev box from a project cancelled in 2019. Every one you retire is a server you don't pay to migrate, test, or run.

**How you identify candidates:**
- Zero or near-zero network connections in discovery data over 30–90 days.
- CPU flatlined at 1–3% (idle OS noise) with no disk I/O.
- No identifiable business owner (the "orphan server" test).
- Functionality duplicated by another system.

**How you do it:**
1. Confirm with a business owner in writing. Always.
2. Take a final backup / snapshot and archive it to **S3 Glacier Deep Archive** with a retention date.
3. Export any data the business may need for compliance (often to S3 + Athena for occasional queries).
4. Power off but don't delete for 30–60 days (the "dark period" — someone always shouts on day 22).
5. Delete, update the CMDB, reclaim licences, cancel support contracts.

**Effort:** Very low · **Cost saving:** Immediate and permanent · **Risk:** Low, if you did step 1.

**Example:** A legacy Crystal Reports server with 0 inbound connections in 90 days of flow data. Data exported to S3, archived, server retired. Saved 4 vCPU, 16 GB RAM and a $12k/yr support contract.

---

### 4.2 Retain — leave it where it is

**Definition:** Do nothing for now. Revisit later. Sometimes called "revisit".

**When it's the right answer:**
- Mainframe or AS/400 workloads that need their own dedicated program.
- Hardware-coupled systems: a manufacturing line controller wired to a PLC, medical imaging tied to a physical modality, an HSM.
- Regulatory or contractual blockers ("data must remain in this building").
- Applications being replaced in 6 months anyway — migrating them is pure waste.
- Latency-sensitive systems tied to on-prem equipment (a trading gateway metres from an exchange cross-connect).
- Vendors who won't support their software on AWS (this is getting rarer, but it happens).

**Important nuance:** *Retain is a decision, not an absence of one.* It must have an owner, a documented reason, and a review date. Otherwise "retain" becomes the bucket where hard applications go to be forgotten, and your data centre never closes.

**Hybrid options that make Retain less painful:**
- **AWS Outposts** — genuine AWS hardware in your rack, same APIs, for latency or residency constraints.
- **AWS Local Zones / Wavelength** — for single-digit-millisecond needs near metros.
- **Storage Gateway** — keep the app local but push backups/archives to S3.
- **Direct Connect + Transit Gateway** — retain the app but let migrated apps talk to it cleanly.

**Effort:** None now (but ongoing data-centre cost) · **Risk:** Low technically, high strategically if abused.

---

### 4.3 Relocate — move the hypervisor, not the servers

**Definition:** Move whole VMware environments to **VMware Cloud on AWS** (or similar) without changing the VMs, the hypervisor, or your operational tooling. This is the 7th R.

**Why it exists:** For a 2,000-VM VMware estate with a lease expiring in 90 days, individually rehosting every VM is impossible. Relocate moves vSphere clusters using **vMotion / HCX**, often with *zero downtime* per VM.

**Characteristics:**
- No OS changes, no IP changes (with stretched L2 via HCX), no re-testing of the app stack.
- Fastest possible exit from a data centre.
- You keep using vCenter, NSX, vSAN — which is both the benefit and the catch.
- Costs more per workload than native EC2, and you don't get native cloud benefits (auto scaling, managed services) until you migrate *again* later.

**Typical pattern:** Relocate now to hit the lease deadline → then migrate VM-by-VM from VMware Cloud on AWS into native EC2/RDS at your own pace. Sometimes called the "two-step" or "beachhead" strategy.

**Effort:** Low per workload, high setup · **Cost:** Higher run cost, lower project cost · **Risk:** Low.

---

### 4.4 Rehost — "lift and shift"

**Definition:** Move the server as-is to EC2. Same OS, same application binaries, same configuration. Only the infrastructure underneath changes.

**This is the workhorse.** In large migrations, 50–70% of workloads get rehosted. It's not a failure of ambition — it's a deliberate strategy: *get out of the data centre first, modernize from a position of safety second.* Modernizing while also fighting a lease deadline is how projects fail.

**Primary tool: AWS Application Migration Service (MGN).** Block-level, continuous replication from any x86 source (physical, VMware, Hyper-V, Azure, GCP, other AWS regions) into a staging area in your AWS account, then a non-disruptive test launch, then a cutover with minutes of downtime.

**How MGN works, mechanically:**

```
┌────────────────────────┐                    ┌─────────────────────────────────────┐
│   SOURCE SERVER        │                    │            AWS ACCOUNT              │
│ ┌────────────────────┐ │   continuous       │ ┌─────────────────────────────────┐ │
│ │ AWS Replication    │ │   block-level      │ │  STAGING AREA SUBNET            │ │
│ │ Agent (kernel-     │─┼───replication──────┼▶│  ┌───────────────────────────┐  │ │
│ │ level block reads) │ │   TCP 1500, AES-256│ │  │ Replication Server (t3.sm)│  │ │
│ └────────────────────┘ │                    │ │  │  + staging EBS volumes    │  │ │
│  OS + app + data       │                    │ │  └───────────────────────────┘  │ │
└────────────────────────┘                    │ └─────────────────┬───────────────┘ │
        stays running                         │                   │ launch          │
        the whole time                        │                   ▼                 │
                                              │ ┌─────────────────────────────────┐ │
                                              │ │ TEST or CUTOVER instance        │ │
                                              │ │ (right-sized, converted drivers,│ │
                                              │ │  ENA/NVMe, agents installed)    │ │
                                              │ └─────────────────────────────────┘ │
                                              └─────────────────────────────────────┘
```

1. **Install the agent** on the source. It reads blocks directly from the disk, below the filesystem, so it doesn't care what the app is doing.
2. **Initial sync** copies every used block to EBS volumes attached to a lightweight replication server in your staging subnet.
3. **Continuous replication** then ships only changed blocks, asynchronously, with sub-second RPO typically.
4. **Launch a Test instance** any time. MGN takes a point-in-time snapshot, runs **conversion** (injects ENA/NVMe drivers, fixes bootloaders, handles licence activation for Windows), and boots a real EC2 instance — while replication keeps running and the source keeps serving users.
5. **Cutover** when tests pass: stop the app, wait for final sync, launch, repoint DNS. Downtime is minutes, not hours.
6. **Finalize** the cutover, which terminates the staging resources.

**What you change during a rehost (allowed and encouraged):**
- Instance type / right-sizing (this is where most of the savings come from).
- Volume type: gp2 → **gp3** (cheaper and faster by default), io1 → io2.
- Adding CloudWatch agent, SSM agent, backup tags via **post-launch actions**.
- Security groups instead of firewall rules; IAM roles instead of stored credentials.

**What you deliberately do *not* change:** OS version, app version, architecture, database engine. Change one variable at a time.

**Effort:** Low–medium · **Speed:** Fast (hundreds of servers/month with a factory) · **Cloud benefit:** Modest · **Risk:** Low.

**Alternatives to MGN for rehost:** `import-image` / VM Import-Export for one-offs and golden AMIs; **CloudEndure** (legacy, now MGN); Elastic Disaster Recovery (DRS) if you want ongoing DR rather than a one-way move; native Hyper-V/VMware export + import for small counts.

---

### 4.5 Replatform — "lift, tinker and shift"

**Definition:** Move to AWS while swapping some components for managed services. The application code stays broadly the same; what runs underneath it changes.

This is the sweet spot for value-per-unit-of-effort, and it's the most under-used R.

**The classic replatform moves:**

| From | To | What you gain | Watch out for |
|---|---|---|---|
| Self-managed MySQL/PostgreSQL on VM | **Amazon RDS / Aurora** | Backups, patching, HA, read replicas, no OS | No OS access, no unsupported plugins, maintenance windows |
| SQL Server on Windows | **RDS for SQL Server** or **Babelfish/Aurora PostgreSQL** | Licence savings, managed HA | Feature gaps (SSRS, linked servers, SQL Agent specifics) |
| Oracle | **RDS for Oracle** (lift licence) or **Aurora PostgreSQL** (convert) | Huge licence savings when converting | PL/SQL conversion effort — use SCT + DMS |
| Windows file server | **Amazon FSx for Windows File Server** | Managed DFS, AD-integrated, no server | Quota/ACL nuances, throughput sizing |
| NFS server | **Amazon EFS** or **FSx for ONTAP/Lustre** | Elastic, multi-AZ | Latency profile differs; per-op cost model |
| Tomcat/IIS on VM | **Elastic Beanstalk** or **ECS/Fargate** | Deploys, scaling, health checks | Session state must go external |
| Self-managed Kafka | **Amazon MSK** | Managed brokers, patching | Version pinning |
| Self-managed Redis/Memcached | **ElastiCache** | Managed, cluster mode | Command/config compatibility |
| Cron server | **EventBridge Scheduler + Lambda/ECS tasks** | No server, retries, observability | Rework of scripts |
| Self-hosted Jenkins | **CodeBuild/CodePipeline** or managed runners | Less toil | Plugin ecosystem loss |
| Windows IIS + .NET Framework | **Windows containers on ECS**, or port to .NET on Linux | Density, cost | Test matrix |
| Hadoop cluster | **EMR** / **EMR Serverless** | Elastic, S3-decoupled storage | Job tuning |
| Self-managed AD | **AWS Managed Microsoft AD** | Patching, HA, trust to on-prem | Schema extension limits |

**Primary tools:** **AWS DMS** (Database Migration Service) for the data movement, **AWS SCT** (Schema Conversion Tool) or **DMS Schema Conversion** for the schema/code when the engine changes, **DataSync** for file data, plus normal deployment tooling for the app tier.

**Homogeneous vs heterogeneous database migration — the single most important distinction:**

- **Homogeneous** (MySQL → RDS MySQL, Oracle → RDS Oracle): the schema and SQL are compatible. Use native tools (`mysqldump`, `pg_dump`, RMAN, backup/restore) for the bulk load, or DMS if you need near-zero downtime. Low risk.
- **Heterogeneous** (Oracle → Aurora PostgreSQL, SQL Server → MySQL): data types, procedural code, functions, sequences and triggers all differ. You need **SCT** to convert the schema and flag what can't be converted automatically, application code changes, and a much longer test cycle. High effort, high reward (licence elimination).

**The near-zero-downtime pattern with DMS (this is the pattern to memorise):**

```
1. SCT / DMS Schema Conversion  ──▶ create target schema (indexes & FKs disabled)
2. DMS Full Load                ──▶ bulk copy existing rows, source stays live
3. DMS CDC (change data capture) ─▶ stream ongoing changes from source txn logs
4. Re-enable indexes, FKs, triggers; validate row counts & checksums
5. Lag ≈ 0 → freeze writes on source (the only downtime, often <10 min)
6. Repoint application connection string / DNS to target
7. Keep DMS task running in reverse as a rollback path for a few days, then stop
```

**Effort:** Medium · **Cloud benefit:** High · **Risk:** Medium (test the data, always).

---

### 4.6 Repurchase — "drop and shop"

**Definition:** Stop running the application yourself. Buy a SaaS equivalent and migrate the data and users into it.

**Classic examples:**

| Self-hosted thing | Repurchase target |
|---|---|
| On-prem Exchange | Microsoft 365 |
| Home-grown CRM | Salesforce / Dynamics 365 |
| Self-hosted HR system | Workday / SuccessFactors |
| On-prem file shares + intranet | SharePoint Online / Google Workspace |
| Self-managed ticketing | ServiceNow / Jira Service Management / Zendesk |
| Legacy BI stack | Amazon QuickSight / Tableau Cloud / Power BI |
| Custom e-commerce | Shopify / commercetools |
| Self-hosted VDI | Amazon WorkSpaces / AppStream 2.0 |

**Why it's attractive:** You delete an entire operational burden. No servers, no patching, no upgrades, no capacity planning. Often the fastest route to a *better* experience for users.

**Why it's harder than it looks:**
- **Data migration and mapping** is the whole job. Your 47 custom fields don't exist in the SaaS data model.
- **Process change.** You now do it the SaaS product's way. This is organizational change management, not IT work.
- **Integrations.** Every system that talked to the old app needs a new API integration (often via **AWS AppFlow**, EventBridge, Lambda, or an iPaaS).
- **Licensing model flips** from capex to per-user opex — sometimes cheaper, sometimes dramatically not, at scale.
- **Data egress and lock-in.** Ask, in writing, how you get your data out.
- **Identity.** Federate via IAM Identity Center / Entra ID / Okta from day one; don't create local SaaS accounts.

**Effort:** Medium–high (mostly non-technical) · **Cloud benefit:** Very high · **Risk:** Medium (business process risk > technical risk).

---

### 4.7 Refactor / Re-architect — rebuild it cloud-native

**Definition:** Substantially change the architecture and often the code to exploit cloud-native capabilities: microservices, serverless, event-driven, managed data stores, containers.

**Do this when:**
- The business need can't be met by the current architecture (can't scale, can't release fast enough, can't survive an AZ failure).
- The monolith is the bottleneck on the roadmap, not just the ops bill.
- There's a strong business case: a *new* revenue capability, or a 10× cost/agility improvement — not just "microservices are modern".

**Target patterns:**
- **Monolith → microservices** behind an API Gateway / ALB, with **strangler fig** incremental extraction.
- **VMs → containers** on ECS Fargate or EKS.
- **Servers → serverless**: Lambda + API Gateway + DynamoDB + EventBridge + Step Functions.
- **Batch → event-driven**: S3 events, SQS, Kinesis, EventBridge.
- **Relational-for-everything → purpose-built data stores**: DynamoDB (key-value), OpenSearch (search), Neptune (graph), Timestream (time series), ElastiCache (cache).
- **Static web tier → S3 + CloudFront**, with the API decoupled behind it.

**The strangler fig pattern (the safest way to refactor a live system):**

```
Phase 1                    Phase 2                       Phase 3
                                                     
 ┌──────────┐          ┌────────────┐              ┌────────────┐
 │ Monolith │          │  Router /  │              │  Router /  │
 │ (all     │◀── all ──│  Refactor  │── 30% ──▶┌───│  Refactor  │──▶ new services
 │  traffic)│  traffic │  Spaces /  │           │   │  Spaces    │    (100%)
 └──────────┘          │  ALB rules │── 70% ──▶ │   └────────────┘
                       └────────────┘  monolith │       
                                                └── monolith retired
```

Route one URL path or one bounded context at a time to a new service. The monolith shrinks until it's gone. **AWS Migration Hub Refactor Spaces** provisions exactly this routing infrastructure (an API Gateway + VPC Link + Transit Gateway mesh) for you.

**Supporting tools:**
- **AWS App2Container (A2C)** — analyses a running Java or .NET app and generates a container image, ECS/EKS task definitions and a CI/CD pipeline. Genuinely useful for "containerize this .NET app I don't have the source for".
- **AWS Transform** / **Amazon Q Developer transformation** — assisted code upgrades (e.g. .NET Framework → cross-platform .NET, Java 8 → 17, VMware/mainframe modernization assistance).
- **AWS Mainframe Modernization** — refactor COBOL to Java, or replatform to a managed runtime.

**Effort:** Highest · **Time:** Months–years · **Cloud benefit:** Highest · **Risk:** Highest.

**Blunt advice:** Don't refactor during the migration itself unless the app is small or the business case is undeniable. Rehost or replatform to get out, then refactor with a stable baseline, real cloud telemetry, and no data-centre deadline hanging over you.

---

### 4.8 The 7 R's at a glance

| R | Also called | Change to app | Effort | Downtime | Cloud benefit | Cost of project | Run cost | Primary AWS tools |
|---|---|---|---|---|---|---|---|---|
| **Retire** | Decommission | N/A | ⬤○○○○ | N/A | Immediate saving | Very low | Zero | ADS, Migration Hub, S3 Glacier |
| **Retain** | Revisit | None | ○○○○○ | None | None | None | Unchanged | Outposts, DX, Storage Gateway |
| **Relocate** | Hypervisor move | None | ⬤⬤○○○ | Near-zero | Low | Low | Higher | VMware Cloud on AWS, HCX |
| **Rehost** | Lift & shift | None | ⬤⬤○○○ | Minutes | Moderate | Low | Lower than DC | **MGN**, VM Import, DRS |
| **Replatform** | Lift, tinker, shift | Config/components | ⬤⬤⬤○○ | Minutes–hours | High | Medium | Lower | **DMS**, **SCT**, DataSync, FSx, RDS |
| **Repurchase** | Drop & shop | Replaced | ⬤⬤⬤⬤○ | Varies | Very high | Medium | Subscription | AppFlow, IAM Identity Center, DMS |
| **Refactor** | Re-architect | Rewritten | ⬤⬤⬤⬤⬤ | Phased | Highest | High | Lowest at scale | Refactor Spaces, A2C, Lambda, ECS/EKS |

---

## 5. How to choose an R

### 5.1 Decision tree

Walk every application through this. It takes about five minutes per app once you have discovery data.

```
                      ┌──────────────────────────────────┐
                      │ Is the app still used by anyone? │
                      └───────────────┬──────────────────┘
                            NO ◀──────┴──────▶ YES
                            │                   │
                      ┌─────▼─────┐             │
                      │  RETIRE   │             │
                      └───────────┘             │
                                                ▼
                      ┌─────────────────────────────────────────────┐
                      │ Is there a hard blocker? (hardware coupling,│
                      │ regulation, vendor support, mainframe,      │
                      │ being replaced in <6 months)                │
                      └───────────────┬─────────────────────────────┘
                            YES ◀─────┴─────▶ NO
                            │                  │
                      ┌─────▼─────┐            │
                      │  RETAIN   │            │
                      │ (+ review │            ▼
                      │   date)   │  ┌────────────────────────────────────┐
                      └───────────┘  │ Is a mature SaaS product available │
                                     │ that meets the business need?      │
                                     └───────────────┬────────────────────┘
                                          YES ◀──────┴──────▶ NO
                                          │                    │
                                   ┌──────▼──────┐             │
                                   │ REPURCHASE  │             ▼
                                   └─────────────┘   ┌──────────────────────────────┐
                                                     │ Does the business case demand │
                                                     │ a new architecture? (scale,   │
                                                     │ velocity, new capability)     │
                                                     └───────────┬───────────────────┘
                                                       YES ◀─────┴─────▶ NO
                                                       │                  │
                                                ┌──────▼──────┐           │
                                                │  REFACTOR   │           ▼
                                                └─────────────┘  ┌───────────────────────────┐
                                                                 │ Can we swap a component    │
                                                                 │ for a managed service with  │
                                                                 │ acceptable effort? (DB,     │
                                                                 │ file share, cache, queue)   │
                                                                 └────────────┬───────────────┘
                                                                   YES ◀──────┴──────▶ NO
                                                                   │                    │
                                                            ┌──────▼──────┐      ┌──────▼──────┐
                                                            │ REPLATFORM  │      │   REHOST    │
                                                            └─────────────┘      └─────────────┘
                                        (Large VMware estate + hard deadline? ──▶ RELOCATE first)
```

### 5.2 Scoring matrix

For portfolios where the tree gives ambiguous answers, score each app 1–5 and let the weights decide. Keep this in a spreadsheet — it becomes your portfolio artefact.

| Criterion | Weight | 1 = | 5 = |
|---|---|---|---|
| Business criticality | 15% | Sandbox | Revenue-critical, 24×7 |
| Technical complexity | 20% | Single stateless VM | 20-tier, clustered, undocumented |
| Interdependencies | 15% | Standalone | Talks to 15 systems |
| Data volume / change rate | 10% | < 100 GB, static | > 50 TB, high churn |
| Compliance sensitivity | 10% | None | PCI / HIPAA / regulated |
| Available downtime window | 10% | Days | Zero |
| Team skills for target state | 10% | Strong | None |
| Remaining app lifespan | 10% | < 1 year | > 5 years |

Rules of thumb from the score:
- Low complexity + short lifespan → **Rehost** (don't invest in something dying).
- Low complexity + long lifespan → **Replatform** (cheap win, long payback).
- High complexity + long lifespan + strategic → **Refactor**.
- High complexity + commodity function → **Repurchase**.
- High compliance + hardware coupling → **Retain** (with Outposts as an option).

### 5.3 A sample portfolio disposition

What a realistic 200-server estate looks like after assessment:

| Disposition | Servers | % | Note |
|---|---|---|---|
| Retire | 28 | 14% | Idle, orphaned, or duplicated |
| Retain | 12 | 6% | 1 mainframe, 2 lab systems, 9 pending replacement |
| Rehost | 108 | 54% | The bulk — get out fast |
| Replatform | 36 | 18% | 22 databases → RDS, 8 file servers → FSx, 6 app servers → Beanstalk |
| Repurchase | 10 | 5% | Exchange → M365, ticketing → SaaS |
| Refactor | 6 | 3% | The two strategic customer-facing apps |

If your plan says 60% refactor, your plan is a wish, not a plan.

---

## 6. High-level architecture and service flow

### 6.1 The end-to-end picture

```
╔══════════════════════════════ SOURCE ENVIRONMENT ══════════════════════════════╗
║                                                                                ║
║  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌────────────────┐   ║
║  │ Physical │  │  VMware  │  │  Hyper-V  │  │ Databases│  │ File servers / │   ║
║  │ servers  │  │  vSphere │  │           │  │ Ora/SQL/ │  │ NAS / SAN      │   ║
║  │          │  │          │  │           │  │ MySQL/PG │  │                │   ║
║  └────┬─────┘  └────┬─────┘  └─────┬─────┘  └────┬─────┘  └───────┬────────┘   ║
║       │             │              │             │                │            ║
║  ┌────▼─────────────▼──────────────▼──┐     ┌────▼────┐     ┌─────▼──────┐     ║
║  │ Discovery Agent / Agentless        │     │ DMS     │     │ DataSync   │     ║
║  │ Collector (vCenter appliance)      │     │ replic. │     │ Agent      │     ║
║  │ + MGN Replication Agent            │     │ instance│     │            │     ║
║  └────────────────┬───────────────────┘     └────┬────┘     └─────┬──────┘     ║
╚═══════════════════╪══════════════════════════════╪════════════════╪════════════╝
                    │                              │                │
        ┌───────────▼──────────────────────────────▼────────────────▼───────────┐
        │   CONNECTIVITY:  Direct Connect  |  Site-to-Site VPN  |  Internet     │
        │                  (+ Transit Gateway, PrivateLink endpoints)           │
        └───────────┬──────────────────────────────┬────────────────┬───────────┘
                    │                              │                │
╔═══════════════════▼══════════════════════════════▼════════════════▼════════════╗
║                              AWS  LANDING  ZONE                                ║
║  ┌──────────────────────────────────────────────────────────────────────────┐  ║
║  │ PLAN & TRACK LAYER                                                       │  ║
║  │  Application Discovery Service ─▶ Migration Hub ─▶ Strategy Recommend.    │  ║
║  │  Migration Hub Orchestrator  ·  Migration Evaluator (TCO)                │  ║
║  └──────────────────────────────────────────────────────────────────────────┘  ║
║  ┌──────────────────────────────────────────────────────────────────────────┐  ║
║  │ MIGRATE LAYER                                                            │  ║
║  │  MGN (staging subnet ─▶ EC2)   DMS (─▶ RDS/Aurora/S3)                    │  ║
║  │  DataSync (─▶ S3/EFS/FSx)      Snow Family (offline bulk)                │  ║
║  │  Transfer Family (SFTP)        App2Container (─▶ ECS/EKS)                │  ║
║  │  Refactor Spaces (strangler)   Elastic Disaster Recovery (DR)            │  ║
║  └──────────────────────────────────────────────────────────────────────────┘  ║
║  ┌──────────────────────────────────────────────────────────────────────────┐  ║
║  │ TARGET WORKLOAD ACCOUNTS (per env: prod / non-prod)                      │  ║
║  │   VPC ┌──────────────┬──────────────┬──────────────────┐                 │  ║
║  │       │ Public/ALB   │ App (private)│ Data (private)   │                 │  ║
║  │       │ ALB, NAT     │ EC2 / ECS /  │ RDS/Aurora multi-│                 │  ║
║  │       │ CloudFront   │ EKS / Lambda │ AZ, FSx, EFS     │                 │  ║
║  │       └──────────────┴──────────────┴──────────────────┘                 │  ║
║  └──────────────────────────────────────────────────────────────────────────┘  ║
║  ┌──────────────────────────────────────────────────────────────────────────┐  ║
║  │ OPERATE & GOVERN LAYER                                                   │  ║
║  │  Control Tower · Organizations · IAM Identity Center · CloudTrail        │  ║
║  │  CloudWatch · Config · Systems Manager · AWS Backup · Security Hub      │  ║
║  │  Cost Explorer · Budgets · Trusted Advisor · Compute Optimizer          │  ║
║  └──────────────────────────────────────────────────────────────────────────┘  ║
╚════════════════════════════════════════════════════════════════════════════════╝
```

### 6.2 Service flow by phase

```
ASSESS
  Application Discovery Service (agent or agentless)
        │  hardware specs, processes, network dependencies, utilization
        ▼
  AWS Migration Hub  ──▶  group servers into APPLICATIONS
        │
        ├──▶ Migration Hub Strategy Recommendations  ──▶ suggested R per app
        └──▶ Migration Evaluator / Pricing Calculator ──▶ TCO & business case

MOBILIZE
  Control Tower / Landing Zone Accelerator ──▶ accounts, OUs, guardrails
  Direct Connect / Site-to-Site VPN + Transit Gateway ──▶ connectivity
  Pilot migration (2–5 low-risk apps) ──▶ prove the factory

MIGRATE  (per wave)
  ┌── Rehost ──────▶ MGN: agent ▸ replicate ▸ test ▸ cutover ▸ finalize
  ├── Replatform ──▶ SCT/DMS SC: convert schema ▸ DMS full load + CDC ▸ validate ▸ cutover
  │                  DataSync: file data ▸ FSx/EFS/S3
  ├── Refactor ────▶ A2C / rebuild ▸ Refactor Spaces routing ▸ incremental traffic shift
  ├── Repurchase ──▶ SaaS tenant ▸ data load ▸ SSO ▸ integrations ▸ user cutover
  ├── Relocate ────▶ VMware Cloud on AWS ▸ HCX vMotion
  ├── Retire ──────▶ archive to Glacier ▸ dark period ▸ delete ▸ CMDB
  └── Retain ──────▶ document reason + review date ▸ hybrid connectivity

OPERATE & OPTIMIZE
  CloudWatch/Config/Backup baselines ▸ Compute Optimizer right-sizing
  Savings Plans / RIs ▸ Well-Architected review ▸ decommission source ▸ modernize
```

---

## 7. Core AWS migration services — deep dive

### 7.1 AWS Application Discovery Service (ADS)

**Job:** Build an accurate inventory of what you have, including the dependencies nobody documented.

Two collection modes — you often use both:

| | **Agentless Collector** | **Discovery Agent** |
|---|---|---|
| Deployment | One OVA appliance into vCenter | Small agent on each server |
| Data collected | VM specs, utilization (from vCenter), plus DB discovery via connectors | Specs, running processes, **TCP/UDP connections**, performance |
| Dependency mapping | Limited (network-level) | **Yes — process-to-process** |
| Guest OS access needed | No | Yes |
| Best for | Fast, broad VMware sweep | Physical servers, non-VMware, deep dependency mapping |

**Key point:** run collection for **at least 2–4 weeks**, ideally including a month-end. Discovery over three days will miss the batch job that only runs on the last Friday, and you'll find out about it during cutover.

**Outputs you actually use:** server inventory CSV, network dependency graph, utilization percentiles (use p95, not average, for sizing), and process lists that reveal what an app really is. Data can be exported to S3 and queried with **Athena** — this is where serious portfolio analysis happens.

**Import option:** if you already have a CMDB or an RVTools export, you can bulk-import inventory into Migration Hub with a CSV instead of deploying anything.

---

### 7.2 AWS Migration Hub

**Job:** Single pane of glass. Group servers into applications, track migration status across MGN/DMS/partner tools, and hold your portfolio data.

Features worth knowing:
- **Home Region** — you pick one region where Migration Hub stores discovery and tracking data. Everything reports there even if you migrate into other regions. Choose it once; changing later is painful.
- **Applications** — logical groupings of servers. This is the unit you plan waves around, not individual servers.
- **Strategy Recommendations** — analyses your inventory (and optionally source code and databases via a collector) and recommends an R per application, with anti-pattern reports for things like unsupported runtime versions.
- **Migration Hub Orchestrator** — templated, automatable workflows (e.g. "migrate SAP NetWeaver", "rehost a server") with step-level tracking. Useful for standardising a factory.
- **Refactor Spaces** — see [§4.7](#47-refactor--re-architect--rebuild-it-cloud-native).
- **Journeys / plans** — track waves, owners and dates.

---

### 7.3 AWS Application Migration Service (MGN)

The rehost engine. Details of the mechanism are in [§4.4](#44-rehost--lift-and-shift); here's what you configure.

**Replication settings template** (set once per account/region, override per server):

| Setting | What to choose and why |
|---|---|
| Staging subnet | A dedicated private subnet, ideally its own; needs outbound 443 to S3 + MGN endpoints |
| Replication server instance type | `t3.small` default; bump for many-disk or high-churn servers |
| EBS volume type for staging | gp3 for most; `st1` for very large, low-IOPS data disks to save cost |
| EBS encryption | Enable, with a KMS CMK — do this from the start |
| Data routing | Private IP (via VPN/DX), or public with optional **VPC endpoint** for PrivateLink |
| Throttling | Cap bandwidth (Mbps) so replication doesn't starve production traffic |
| Use dedicated replicator | Off by default (shared replication servers save money) |
| Point-in-time (PIT) snapshot retention | Days of recovery points to keep — costs EBS snapshots |

**Launch settings template** (what the target EC2 looks like):

| Setting | Notes |
|---|---|
| Instance type right-sizing | `BASIC` auto-sizing, or `NONE` + explicit type. **Prefer explicit**, informed by p95 utilization, not source specs |
| Target subnet / SG / IAM role | Set to your real app subnets and instance profile |
| Licensing | BYOL vs licence-included for Windows (`osByol`) |
| Boot mode | BIOS / UEFI — must match the source or it won't boot |
| Copy private IP / tags | Handy for like-for-like network moves |
| **Post-launch actions** | Install SSM/CloudWatch agents, join domain, run validation scripts, enable Inspector — automated per launch |

**Lifecycle states you'll see:** `NOT_STARTED` → `INITIATING` → `INITIAL_SYNC` → `READY_FOR_TEST` → `TESTING` → `READY_FOR_CUTOVER` → `CUTTING_OVER` → `CUTOVER_COMPLETE`. `Finalize cutover` then deletes staging resources and stops replication billing.

**Limits and gotchas:** 
- Source disks are replicated up to supported sizes; very large single volumes (>~16 TiB) need a different approach (DataSync for the data + rehost the OS).
- Agent needs root/Administrator and a supported kernel; ancient kernels may need `--no-prompt` legacy handling or a different R.
- MGN charges are free for the migration period per server (90 days) but you pay for the EBS, snapshots, and replication instances.

---

### 7.4 AWS Elastic Disaster Recovery (DRS)

Same replication engine as MGN, different purpose: keep an ongoing, cheap standby copy of servers in AWS for DR, with sub-second RPO and minutes RTO.

Use it in a migration context when:
- You want DR *first* as a lower-risk stepping stone, then flip to using AWS as primary.
- You need a proven rollback: after cutover, use DRS to replicate **back** to on-prem (failback) for a safety period.

---

### 7.5 AWS Database Migration Service (DMS)

**Job:** Move data between databases, with minimal downtime, homogeneous or heterogeneous.

**Building blocks:**
1. **Replication instance** — an EC2-based (or **serverless**) instance that does the work. Size it for throughput; put it in the target VPC with routes to both source and target. Multi-AZ for prod.
2. **Endpoints** — source and target connection definitions. Test them before doing anything else. Use **Secrets Manager** for credentials.
3. **Task** — what to migrate, with a **migration type**:
   - `full-load` — one-time bulk copy.
   - `cdc` — changes only, from a chosen start point (LSN/SCN/binlog position).
   - `full-load-and-cdc` — bulk copy then continuous changes. **This is the near-zero-downtime option.**
4. **Table mappings** — JSON selection rules (which schemas/tables) and transformation rules (rename, change case, add prefix, remove columns).
5. **Task settings** — LOB handling, error handling, `TargetTablePrepMode`, logging, batch apply, parallel load.
6. **Data validation** — DMS can compare source and target row-by-row and report mismatches. **Turn it on.**

**Prerequisites people forget:**
- Source must have **logging enabled for CDC**: MySQL `binlog_format=ROW` + `binlog_row_image=FULL`; PostgreSQL `wal_level=logical` + `pglogical`/native slots; Oracle ARCHIVELOG + supplemental logging; SQL Server full recovery model + change tracking/CDC and `MS-REPLICATION` support.
- Tables generally need **primary keys** for CDC; ones without will full-load fine but not replicate updates cleanly.
- **LOBs**: full LOB mode is slow and memory-hungry; limited LOB mode truncates. Know your max LOB size and set `LobMaxSize`.
- DMS moves **data**, not: secondary indexes (create after load for speed), sequences (reset manually after cutover), stored procedures, triggers (disable during load), users/grants, or scheduled jobs.

**Other DMS uses in migration:** database → S3 (as Parquet, for a data lake), database → Kinesis/Kafka (event streams), and **DMS Fleet Advisor** (discover and size your on-prem database fleet).

---

### 7.6 AWS Schema Conversion Tool (SCT) / DMS Schema Conversion

**Job:** Convert schema and procedural code between different engines, and tell you honestly what it can't convert.

Workflow:
1. Connect to source and target (or just source, with a virtual target).
2. Run the **assessment report** — a percentage-based view of automatic conversion, with **action items** graded by effort (simple / medium / significant).
3. Convert schema, review the generated DDL, fix action items by hand.
4. Apply to target.
5. Use **SCT data extraction agents** for very large warehouse migrations (Teradata/Netezza/Greenplum → Redshift) where DMS isn't the right mover.

SCT is also the tool that reports "your app has 412 embedded SQL statements that need changing" — run it early, because that number changes your timeline.

---

### 7.7 AWS DataSync

**Job:** Move file and object data efficiently, online, with verification.

- Sources: NFS, SMB, HDFS, self-managed object storage, S3, EFS, FSx, and other clouds.
- Targets: S3 (any storage class), EFS, FSx for Windows / Lustre / ONTAP / OpenZFS.
- Features: parallel multi-threaded transfer, incremental sync, **data integrity verification**, bandwidth throttling, filters/includes-excludes, scheduling, metadata and ACL preservation (SMB → FSx keeps NTFS ACLs).
- Deployment: an **agent** VM on-prem (or an EC2 agent for in-cloud tasks); tasks defined in AWS.

Rule of thumb: DataSync for anything from ~100 GB to tens of TB over a decent link; Snow Family beyond that or when bandwidth is poor.

---

### 7.8 AWS Snow Family

**Job:** Offline, physical data transfer when the network can't do it in time.

| Device | Capacity | Use for |
|---|---|---|
| **Snowcone** | ~8–14 TB | Edge, small/remote sites, ruggedised |
| **Snowball Edge Storage Optimized** | ~80 TB usable | Bulk data transfer, the common choice |
| **Snowball Edge Compute Optimized** | ~42 TB + GPU/compute | Edge processing then ship |

Flow: order in console → device arrives → unlock with manifest + unlock code → copy data (`snowballEdge` CLI / S3 adapter / DataSync on device / NFS mount) → ship back → AWS imports to S3 → validate → device erased to NIST standards.

**The classic calculation:** if `data_size / usable_bandwidth > your window`, go offline. See [§10.1](#101-the-bandwidth-arithmetic).

---

### 7.9 Other services in the toolbox

| Service | Use it for |
|---|---|
| **AWS Transfer Family** | Preserve existing SFTP/FTPS/AS2 workflows while the backend becomes S3/EFS |
| **AWS Storage Gateway** | Hybrid file/volume/tape gateway — great for Retain and for backup-first migrations |
| **AWS App2Container** | Containerize existing Java/.NET apps without source-code archaeology |
| **AWS Application Discovery Service Agentless Collector — DB module** | Discover and size on-prem databases |
| **AWS Migration Evaluator** | Automated TCO business case from discovery data |
| **AWS Compute Optimizer** | Post-migration right-sizing recommendations from real CloudWatch data |
| **AWS Backup** | Central, policy-driven backup for the migrated estate from day one |
| **AWS Systems Manager** | Patching, inventory, Session Manager, Run Command, Automation runbooks |
| **AWS Control Tower / LZA** | Multi-account landing zone with guardrails |
| **AWS Resilience Hub** | Assess whether the migrated app actually meets its RTO/RPO |
| **AWS Mainframe Modernization** | Replatform or refactor mainframe workloads |
| **AWS Managed Microsoft AD / AD Connector** | Identity for migrated Windows workloads |
| **Route 53 Resolver + endpoints** | Hybrid DNS during the transition — often the unsung hero |
| **VM Import/Export** | Import an OVA/VHD/VMDK into an AMI for golden images and one-offs |

---

## 8. Step-by-step implementation guide

This is the sequence I'd follow on a real programme. Commands for each step live in [commands-cheatsheet.md](commands-cheatsheet.md); guided practice is in [hands-on-labs.md](hands-on-labs.md).

### Stage 0 — Set the foundations (Week 0)

1. **Pick your Migration Hub home region.** Everything reports here.
2. **Create the account structure** (or at least the sandbox + one target account).
3. **Enable billing alerts and cost allocation tags** before you spend anything.
4. **Define your tagging standard now.** Non-negotiable minimum: `Application`, `Environment`, `Owner`, `CostCenter`, `MigrationWave`, `DataClassification`. Retrofitting tags across 400 instances is miserable.
5. **Check service quotas** for the target region: vCPU per family, EBS volume count, EIPs, VPCs, security groups per ENI, RDS instances. Raise them early — quota increases take days.

### Stage 1 — Discover (Weeks 1–4)

1. Deploy the **Agentless Collector** into vCenter and/or roll out **Discovery Agents** to physical and non-VMware servers.
2. Let it run **at least two weeks**, preferably four, covering a month-end close.
3. Import any existing CMDB/RVTools data to fill gaps.
4. Export data to S3 and analyse in Athena: which servers are idle? which talk to each other? what's the p95 CPU and memory?
5. Interview application owners. Discovery tells you the *what*; only humans tell you the *why* and the "oh, and it also…".
6. **Group servers into applications** in Migration Hub. An application = the set of servers you would cut over together.

**Deliverables:** server inventory, application inventory, dependency map, utilization data.

### Stage 2 — Assess and decide (Weeks 3–6)

1. Run **Strategy Recommendations** for a machine opinion on each app's R.
2. Apply the [decision tree](#51-decision-tree) and [scoring matrix](#52-scoring-matrix) with the app owners in the room. Record the decision *and the reason*.
3. Build the **TCO business case**: current-state cost (hardware refresh, DC, power, licences, support, staff) vs target-state AWS cost (right-sized compute with Savings Plans, storage, data transfer, managed services, migration project cost).
4. Right-size using **p95 utilization**, not the source spec. A 32 vCPU box averaging 4% CPU is not a 32 vCPU instance.
5. Get executive sign-off on scope, budget and the data-centre exit date.

**Deliverables:** disposition register (every app has an R and an owner), TCO model, funded business case, high-level wave plan.

### Stage 3 — Build the landing zone (Weeks 4–10, parallel)

See [§9](#9-landing-zone-and-network-foundation). Output: accounts, network, identity, logging, backup, guardrails and reference architectures — tested.

### Stage 4 — Pilot (Weeks 8–12)

Pick 2–5 applications that are: low business risk, representative of common patterns, and owned by people who will actually engage. Migrate them end-to-end including cutover and a week of operations.

The pilot's real output isn't the migrated app — it's **the runbook**, the discovered gaps, and a team that has done it once.

### Stage 5 — Build the migration factory (Weeks 10–14)

Turn the pilot into a repeatable process:
- Standard **runbooks** per pattern (rehost-Linux, rehost-Windows, DB-replatform, file-migration).
- **Automation**: MGN launch templates, post-launch actions, IaC modules for target infra, validation scripts.
- **Roles**: wave lead, migration engineers, app SMEs, network, security, testing, comms.
- **Cadence**: e.g. Mon plan → Tue–Thu prep/replicate → Fri test → Sat cutover → Mon hypercare.
- **Tracking**: Migration Hub + a simple dashboard everyone can see.

### Stage 6 — Execute waves (Months 3–24)

For each wave, the loop is always the same:

```
 1. Wave kickoff       — scope confirmed, owners engaged, downtime window agreed
 2. Pre-migration      — target infra built via IaC; firewall/route changes raised;
                         backups verified; rollback plan written and reviewed
 3. Replicate          — MGN agents installed; DMS full load started; DataSync running
 4. Test launch        — isolated test VPC/subnet; functional + integration + perf tests
 5. Fix & re-test      — iterate until the app owner signs off (in writing)
 6. Cutover            — freeze, final sync, launch, repoint DNS, smoke test, release
 7. Validate           — functional, performance, integration, backup, monitoring
 8. Hypercare          — 3–14 days of heightened support, daily check-ins
 9. Finalize           — MGN finalize cutover, stop DMS tasks, remove agents
10. Decommission       — archive & power off source (after the dark period)
11. Retrospective      — what went wrong, fix the runbook before the next wave
```

### Stage 7 — Optimize and modernize (ongoing)

1. **Right-size** using Compute Optimizer after 2–4 weeks of real load.
2. **Commit** to Savings Plans / RIs once usage is stable (not before — you'll over-commit).
3. **Storage lifecycle**: gp2 → gp3, S3 Intelligent-Tiering, delete orphaned snapshots and unattached volumes.
4. **Well-Architected review** per critical workload; fix the high-risk items.
5. **Modernize** from the backlog you built during assessment: the replatform and refactor items you deliberately deferred.

---

## 9. Landing zone and network foundation

### 9.1 Multi-account structure

```
Root
├── Security OU
│   ├── Log Archive account        (CloudTrail, Config, VPC Flow Logs, ALB logs)
│   └── Audit / Security Tooling   (Security Hub, GuardDuty, Detective, Inspector)
├── Infrastructure OU
│   ├── Network account            (Transit Gateway, DX, VPN, Route 53 Resolver, NFW)
│   └── Shared Services account    (AD, patching, tooling, CI/CD, golden AMIs)
├── Migration OU
│   └── Migration Tooling account  (MGN staging, DMS instances, DataSync agents)
└── Workloads OU
    ├── Prod OU        → prod-app-a, prod-app-b …
    └── Non-Prod OU    → dev/test/staging accounts
```

Build it with **AWS Control Tower** (fastest, opinionated) or the **Landing Zone Accelerator** / Terraform modules (most flexible). Apply **SCPs** for guardrails: deny region use outside approved regions, deny root user actions, deny disabling CloudTrail, require encryption.

### 9.2 Network design

```
        ON-PREMISES                              AWS
   ┌──────────────────┐              ┌─────────────────────────────────┐
   │  DC Router        │              │      NETWORK ACCOUNT            │
   │                   │═ DX (primary)│  ┌───────────────────────────┐  │
   │  10.0.0.0/8       │══════════════│▶ │   Transit Gateway         │  │
   │                   │═ VPN (backup)│  │   + route tables per      │  │
   └──────────────────┘              │  │     segment (prod/nonprod)│  │
                                     │  └────┬──────────┬───────────┘  │
                                     └───────┼──────────┼──────────────┘
                                             │          │
                              ┌──────────────▼──┐   ┌───▼───────────────┐
                              │ PROD VPC        │   │ MIGRATION VPC     │
                              │ 10.20.0.0/16    │   │ 10.99.0.0/16      │
                              │ ┌─────────────┐ │   │ ┌───────────────┐ │
                              │ │ AZ-a: pub/  │ │   │ │ MGN staging   │ │
                              │ │ app/data    │ │   │ │ DMS instances │ │
                              │ ├─────────────┤ │   │ │ DataSync      │ │
                              │ │ AZ-b: pub/  │ │   │ └───────────────┘ │
                              │ │ app/data    │ │   └───────────────────┘
                              │ └─────────────┘ │
                              └─────────────────┘
```

**Rules that save you pain:**
- **Plan CIDRs to avoid overlap with on-prem.** Overlapping RFC1918 ranges are the #1 cause of "we need to re-IP everything" surprises. Reserve a clean supernet for AWS.
- Use **Transit Gateway** from the start, not VPC peering, unless you have exactly two VPCs forever.
- **Three-tier subnets per AZ** (public / private-app / private-data) in at least two AZs. Migrated single-server apps still belong in a multi-AZ-capable design.
- **Hybrid DNS**: Route 53 Resolver **inbound** endpoints so on-prem can resolve AWS names, **outbound** endpoints + rules so AWS resolves on-prem zones. Do this *before* migrating anything that uses hostnames (all of it).
- **VPC endpoints** (Gateway for S3/DynamoDB, Interface/PrivateLink for SSM, MGN, DMS, Secrets Manager, KMS…) to keep migration traffic off the internet and cut NAT costs.
- **Security groups reference other security groups**, not IP ranges, wherever possible. Translate the old firewall rules; don't transliterate them.
- Watch the **MTU / jumbo frames** story across DX and TGW; MSS clamping issues show up as "big file transfers hang".

### 9.3 Identity

- **IAM Identity Center** federated to your existing IdP (Entra ID, Okta, ADFS) for humans. No IAM users.
- **IAM roles** for workloads. Kill every hardcoded access key you find during migration — this is a golden opportunity.
- **AWS Managed Microsoft AD** with a trust to on-prem AD, or AD Connector, for Windows workloads and FSx.

---

## 10. Data migration mechanics

### 10.1 The bandwidth arithmetic

Do this calculation *before* promising a date.

```
Effective throughput ≈ link speed × 0.7   (protocol overhead, contention, latency)

Time (hours) = Data (GB) × 8 / (Effective Mbps × 3600 / 1000)
```

| Data | 100 Mbps | 500 Mbps | 1 Gbps | 10 Gbps |
|---|---|---|---|---|
| 1 TB | ~32 h | ~6.5 h | ~3.2 h | ~20 min |
| 10 TB | ~13 days | ~2.7 days | ~32 h | ~3.2 h |
| 50 TB | ~66 days | ~13 days | ~6.6 days | ~16 h |
| 100 TB | ~132 days | ~26 days | ~13 days | ~32 h |
| 500 TB | not viable | ~132 days | ~66 days | ~6.6 days |

And remember the **change rate**: if the data changes faster than you can replicate it, you never converge. Measure daily churn, not just total size.

### 10.2 Choosing a data movement method

```
                      ┌─────────────────────────────┐
                      │  How much data, how fast?   │
                      └──────────────┬──────────────┘
        ┌────────────────────────────┼────────────────────────────┐
        ▼                            ▼                            ▼
  < 10 TB, good link          10–100 TB, decent link        > 100 TB or poor link
        │                            │                            │
   DataSync / DMS /            DataSync (parallel tasks)    Snowball Edge (multiple)
   native dump+restore         or DX + DMS                  or Snowmobile-class engagement
        │                            │                            │
        └──────────── plus CDC / incremental sync to close the gap ─┘
```

| Method | Online/Offline | Best for | Downtime |
|---|---|---|---|
| MGN block replication | Online | Whole servers | Minutes |
| DMS full-load + CDC | Online | Databases, near-zero downtime | Minutes |
| Native dump/restore | Offline-ish | Small DBs, homogeneous | Hours |
| Native backup to S3 + restore | Offline-ish | SQL Server, Oracle RMAN | Hours |
| DataSync | Online | File shares, NAS, object | Minutes (final sync) |
| Storage Gateway | Online/hybrid | Backup-first, gradual | N/A |
| Snow Family | Offline | Huge datasets, bad links | Days (+ final CDC) |
| S3 Transfer Acceleration / multipart | Online | Object data over long distance | N/A |
| Kinesis/Kafka replication | Online | Streaming data | None |
| Application-level replication (AlwaysOn, GoldenGate, log shipping) | Online | Databases with existing tooling | Minutes |

### 10.3 The hybrid pattern (how big datasets are really moved)

```
Day 0    ──▶ Snowball: bulk copy 80 TB, ship to AWS
Day 7    ──▶ AWS imports to S3; restore/seed into target
Day 8-14 ──▶ DMS CDC (or DataSync incremental) catches up the delta since Day 0
Day 15   ──▶ lag ≈ 0 → short freeze → final sync → cutover
```

This is the standard answer to "we have 80 TB and a 100 Mbps link and a 4-hour window."

---

## 11. Wave planning and the migration factory

### 11.1 How to build waves

A **wave** is a batch of applications migrated together over a period (typically 2–4 weeks with one or two cutover events).

Group by **dependency cluster first**, then by everything else:

1. **Never split a tightly-coupled dependency group across waves.** If app A hammers app B's database with chatty low-latency calls, they migrate together — otherwise you create a WAN round-trip for every query and everyone blames the cloud.
2. Order waves by increasing risk: dev/test first, then internal low-criticality, then business apps, then the crown jewels last.
3. Batch similar patterns together — 20 identical Linux web servers in one wave lets you reuse everything.
4. Respect business calendars: no retail cutovers in December, no finance cutovers at quarter-end.
5. Keep wave size to what the team can actually support in hypercare. 10–30 servers per wave is common; 200 is heroism.

### 11.2 Wave plan template

| Wave | Theme | Apps | Servers | R mix | Cutover date | Risk | Owner | Rollback |
|---|---|---|---|---|---|---|---|---|
| 0 | Pilot | 3 | 6 | Rehost ×2, Replatform ×1 | Wk 10 Sat | Low | — | MGN revert to source |
| 1 | Dev/Test | 12 | 34 | Rehost | Wk 14 | Low | — | Rebuild |
| 2 | Internal tools | 9 | 22 | Rehost, Replatform | Wk 18 | Med | — | DNS revert |
| 3 | Reporting + DB | 6 | 18 | Replatform | Wk 22 | Med | — | Reverse DMS |
| 4 | Customer-facing | 4 | 26 | Rehost + Refactor phase 1 | Wk 28 | High | — | Blue/green |

### 11.3 The migration factory concept

```
   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │ INTAKE   │──▶│ PREPARE  │──▶│ MIGRATE  │──▶│ VALIDATE │──▶│ HANDOVER │
   │ discovery│   │ target   │   │ replicate│   │ test &   │   │ to ops + │
   │ + owner  │   │ infra via│   │ + test + │   │ sign-off │   │ hypercare│
   │ + R      │   │ IaC      │   │ cutover  │   │          │   │          │
   └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
        │              │              │              │              │
        └──────── standard runbooks · automation · tracking ─────────┘
```

The point of a factory is that migration #50 costs a fraction of migration #1. Every manual step you automate, every runbook you refine, compounds. Measure it: track hours-per-server and watch it fall.

---

## 12. Cutover strategies and rollback

### 12.1 Strategies compared

| Strategy | How it works | Downtime | Rollback | Use when |
|---|---|---|---|---|
| **Big bang** | Everything moves in one window | Hours | Restore/repoint back | Small estates, hard deadlines, tightly coupled systems |
| **Phased / wave** | App by app, wave by wave | Minutes per app | Per-app | Default for most programmes |
| **Blue/green** | Full parallel environment, switch traffic | Near-zero | Switch back instantly | Critical apps, budget for double-running |
| **Canary / weighted** | Route 5% → 25% → 100% via Route 53 weights or ALB | Zero | Reduce weight to 0 | Stateless web tiers |
| **Strangler fig** | Migrate one function at a time behind a router | Zero | Route back per function | Refactoring a monolith |
| **Parallel run** | Both systems live, compare outputs | Zero | Just keep using old | High-assurance (finance, billing) |

### 12.2 Cutover runbook template

Write this per application. Rehearse it during the test launch.

```
CUTOVER RUNBOOK — <Application> — Wave <n> — <date>

ROLES
  Cutover lead:            Comms lead:
  Migration engineer:      App SME:
  DBA:                     Network:
  Business validator:      Approver (go/no-go):

TIMELINE (T = cutover start)
 T-7d   Test launch signed off by app owner            [ ]
 T-3d   Change request approved                        [ ]
 T-2d   Rollback plan reviewed & rehearsed             [ ]
 T-1d   Final backup of source verified restorable     [ ]
 T-1d   DNS TTL lowered to 60s                         [ ]
 T-4h   Go/no-go call — GO / NO-GO                     [ ]
 T-0    Notify users; enable maintenance page          [ ]
 T+5m   Stop application services on source            [ ]
 T+10m  Confirm final replication / DMS lag = 0         [ ]
 T+15m  Launch cutover instances (MGN) / promote target [ ]
 T+30m  Post-launch: agents, domain join, services up   [ ]
 T+40m  Smoke tests (checklist below)                   [ ]
 T+60m  Repoint DNS / update connection strings         [ ]
 T+75m  Business validation                             [ ]
 T+90m  Remove maintenance page; announce completion    [ ]
 T+2h   Monitoring & alarms confirmed firing correctly  [ ]

SMOKE TESTS
 [ ] App responds on expected URL/port
 [ ] Login works (incl. SSO / AD auth)
 [ ] Database connectivity + a read and a write
 [ ] Scheduled job / cron / SQL Agent visible and enabled
 [ ] Integrations reachable (list each one)
 [ ] File shares mounted, permissions correct
 [ ] Certificates valid, TLS chain complete
 [ ] Logs flowing to CloudWatch
 [ ] Backup job registered in AWS Backup

ROLLBACK TRIGGERS (decide these in advance, not in the moment)
 • Smoke tests fail and can't be fixed within <X> minutes
 • Data validation mismatch on any critical table
 • Performance > <Y>% worse than baseline
 • Any data-integrity doubt whatsoever

ROLLBACK PROCEDURE
 1. Announce rollback decision (comms lead)
 2. Revert DNS / connection strings to source
 3. Restart services on source
 4. Verify smoke tests against source
 5. Stop target instances (do NOT delete — needed for forensics)
 6. Confirm no writes landed on target that need reconciling
 7. Post-incident review within 48h; fix, then reschedule

POINT OF NO RETURN: <explicitly state when rollback stops being possible —
usually once production writes have landed on the target and been accepted>
```

### 12.3 Rollback realities

- **Rehost:** easiest. The source VM still exists and is intact. Repoint DNS, start services. Keep the source powered off but present for the dark period.
- **Replatform (DB):** hardest. Once writes land on the new database, going back means reverse-replicating. Set up a **reverse DMS task** before cutover if the app is critical.
- **Refactor:** rollback = route traffic back to the monolith. This is why strangler fig is worth the effort.
- **Repurchase:** rollback is usually "keep the old system running in read-only mode for 90 days".

**Rule:** if you cannot describe the rollback in one paragraph, you are not ready to cut over.

---

## 13. Testing and validation

### 13.1 Test layers

| Layer | What you check | When |
|---|---|---|
| **Infrastructure** | Instance boots, right size, disks attached and mounted, network reachable, DNS resolves, NTP synced, agents running | Test launch |
| **Functional** | Core user journeys work; the app owner's checklist | Test launch |
| **Integration** | Every upstream and downstream system, including the ones nobody mentioned | Test launch |
| **Data** | Row counts, checksums, spot-check business records, DMS validation report | Before cutover |
| **Performance** | Response times and throughput vs a **pre-migration baseline you actually captured** | Test launch |
| **Security** | Ports, SGs, encryption at rest and in transit, no public exposure, IAM least privilege, patch level | Before cutover |
| **Operational** | Backups run and **restore**, monitoring alarms fire, logs arrive, runbooks accurate, on-call knows | Before handover |
| **DR** | Failover to another AZ/region meets the stated RTO/RPO | Post-migration |

### 13.2 The baseline you must capture before you touch anything

You cannot prove the migration didn't make things slower if you never measured "before". Capture, for a normal business week:

- CPU / memory / disk IOPS / network — p50, p95, p99 (not averages).
- Application response times for the top 10 transactions.
- Batch job durations.
- Peak concurrent users.
- Error rates.

### 13.3 Data validation techniques

```sql
-- 1. Row counts, per table
SELECT 'orders' AS t, COUNT(*) FROM orders
UNION ALL SELECT 'customers', COUNT(*) FROM customers;

-- 2. Aggregate checksums on business-critical numbers
SELECT SUM(amount), MIN(created_at), MAX(created_at), COUNT(DISTINCT customer_id)
FROM orders;

-- 3. Hash a canonical representation of each row (engine-specific)
SELECT MD5(STRING_AGG(id || '|' || amount || '|' || status, ',' ORDER BY id))
FROM orders;

-- 4. Spot-check known records, including edge cases:
--    NULLs, unicode, emojis, max-length strings, negative numbers,
--    dates before 1970 and after 2038, timezone-sensitive timestamps
```

Also let **DMS data validation** run — it does row-by-row comparison and writes mismatches to a control table. Then have a human check ten records they care about. Both matter.

---

## 14. Security, compliance and governance

### 14.1 Security during migration (the transit risk)

- **Encrypt in transit**: MGN and DMS use TLS/AES-256; DataSync encrypts in transit. Prefer **private connectivity** (DX/VPN + VPC endpoints) over the public internet for replication.
- **Encrypt at rest**: enable EBS encryption by default, KMS CMKs for staging volumes, snapshots, RDS, S3 (SSE-KMS), FSx.
- **Credentials**: use **Secrets Manager** for DMS endpoints and app credentials. Rotate every credential that has ever appeared in a migration runbook or chat message.
- **Least privilege**: dedicated IAM roles for MGN (`AWSApplicationMigrationAgentPolicy`), DMS (`dms-vpc-role`, `dms-cloudwatch-logs-role`), DataSync. Don't run migrations as admin because it's easier.
- **Network isolation**: a dedicated staging/migration subnet; test launches into an **isolated test subnet with no route to production** so a test instance can't send real emails or write to real systems. This bites people. Every time.
- **Logging**: CloudTrail on everywhere, VPC Flow Logs on the migration and target VPCs, agent logs retained.

### 14.2 Compliance carry-over

Map each control, not each server:

| Concern | Where it lands in AWS |
|---|---|
| Data residency | Region and AZ choice; SCPs denying other regions |
| Encryption mandates | KMS CMKs, key rotation, Config rules to detect unencrypted resources |
| Audit trail | CloudTrail (org trail, log archive account, immutable S3 with Object Lock) |
| Access review | IAM Identity Center + Access Analyzer + quarterly review |
| Vulnerability management | Inspector, SSM Patch Manager, patch baselines |
| Change management | CloudTrail + Config timeline + your ITSM integration |
| Data classification | Tags + Macie for S3 |
| Segregation of duties | Multi-account boundaries, permission sets |

### 14.3 Governance guardrails to set on day one

- **SCPs**: deny unapproved regions, deny root actions, deny CloudTrail/Config deletion, require IMDSv2, deny public S3 where inappropriate.
- **AWS Config rules**: required tags, encrypted volumes, no unrestricted 0.0.0.0/0 on SSH/RDP, approved AMIs.
- **Budgets and anomaly detection**: per account and per `MigrationWave` tag.
- **Backup policies**: an AWS Backup plan applied by tag, so every migrated instance is protected automatically rather than by memory.

---

## 15. Cost, licensing and the business case

### 15.1 What goes into a real TCO comparison

**Current state (people forget most of these):**
- Server hardware amortisation and the next refresh cycle
- Storage arrays, SAN switches, tape
- Network hardware, load balancers, firewalls
- Data-centre space, power, cooling, cross-connects
- Software licences and support (OS, DB, hypervisor, backup, monitoring)
- Staff time on patching, hardware, capacity planning
- Disaster-recovery site (often 100% duplicate cost)
- Downtime cost and opportunity cost of slow provisioning

**Target state:**
- Right-sized compute, discounted by Savings Plans/RIs (typically 30–60%)
- EBS/S3/EFS/FSx storage, including snapshots (the forgotten line item)
- Data transfer — **egress and cross-AZ**, which surprises everyone
- Managed service premiums (RDS costs more than EC2+MySQL, and is worth it)
- Migration project cost: tools, partner fees, internal time, double-running period
- Ongoing cloud operations and training

### 15.2 Cost optimization specific to migration

| Lever | Typical saving | Note |
|---|---|---|
| Right-sizing from p95 (not source specs) | 20–40% | The single biggest lever |
| Retire dead servers before migrating | 10–20% of estate | Free money |
| Shut down non-prod out of hours | ~65% on those instances | Instance Scheduler / SSM automation |
| gp2 → gp3 | ~20% on EBS | Also usually faster |
| Savings Plans / RIs (after stabilising) | 30–60% | Wait 2–4 weeks post-migration |
| Graviton where supported | 20–40% | Easy for Linux/containers, needs testing |
| Licence optimization (Windows/SQL → Linux/PostgreSQL) | Can dwarf everything else | Drives the replatform case |
| Delete orphaned snapshots/volumes/EIPs post-cutover | Small but constant | Automate the sweep |
| S3 lifecycle + Intelligent-Tiering for archived source data | Large on retired-app archives | Glacier Deep Archive for compliance copies |

### 15.3 Licensing notes that matter

- **Windows Server / SQL Server**: licence-included EC2 vs BYOL on Dedicated Hosts. BYOL usually needs a Dedicated Host and licence mobility rights — check with your reseller *before* planning.
- **Oracle**: licensing on EC2 counts vCPUs differently than on-prem sockets. This is often what makes Aurora PostgreSQL conversion irresistible.
- **Per-socket / per-core products**: recalculate; cloud vCPU counting can inflate cost.
- **MAP funding**: AWS Migration Acceleration Program can provide credits and partner funding — if it applies, engage before you start, because it's tied to assessment artefacts.

---

## 16. Workload-specific playbooks

### 16.1 Linux web/app server → EC2 (Rehost)
MGN agent → replicate → test launch in isolated subnet → verify services, mounts, cron, SELinux context → cutover → post-launch actions install SSM + CloudWatch agent → ALB in front → target group health checks → DNS.
*Watch for:* hardcoded IPs in configs, `/etc/fstab` entries pointing at old NFS, cron jobs that email out, licence dongles, kernel too old for ENA.

### 16.2 Windows app server → EC2 (Rehost)
MGN with correct boot mode → domain join via post-launch action → verify services set to Automatic → check DFS/mapped drives → Windows licensing (BYOL vs included) → EC2Launch v2 settings → RDP via Session Manager, not a public IP.
*Watch for:* SIDs and duplicate machine names, GPO reapplication, static IP in the NIC, scheduled tasks with stored credentials, MSDTC/RPC dynamic port ranges in SGs.

### 16.3 MySQL/PostgreSQL → RDS/Aurora (Replatform, homogeneous)
Pre-check versions and parameters → create target with a **custom parameter group matching source semantics** → `mysqldump`/`pg_dump` for bulk or DMS full-load → DMS CDC → create secondary indexes after load → validate → repoint connection string.
*Watch for:* `sql_mode`, timezone tables, collation differences, superuser-only features, extensions, `max_allowed_packet`, users and grants (not migrated by DMS).

### 16.4 Oracle / SQL Server → Aurora PostgreSQL (Replatform, heterogeneous)
SCT assessment first (this is your effort estimate) → convert schema, fix action items → convert application SQL (or use **Babelfish** for SQL Server T-SQL compatibility) → DMS full-load + CDC → run both in parallel and compare → extensive UAT → cutover.
*Watch for:* PL/SQL packages, hierarchical queries, `DBMS_*` calls, sequences vs identity, empty-string-vs-NULL semantics, date arithmetic, hints, and case sensitivity of identifiers.

### 16.5 Windows file server → FSx for Windows (Replatform)
FSx in the right AD (Managed AD or self-managed) → DataSync SMB→FSx preserving ACLs → repeat incremental syncs → final sync during window → repoint DFS namespace or update mapped drives via GPO → verify ACLs and quotas.
*Watch for:* long paths, open file handles, ACL inheritance, throughput capacity sizing, shadow copies configuration.

### 16.6 NFS/NAS → EFS or FSx (Replatform)
DataSync NFS→EFS → mount targets in each AZ → performance mode and throughput mode choice → update `/etc/fstab` with the EFS DNS name and TLS mount option.
*Watch for:* per-operation latency being higher than a local NAS; chatty workloads may need FSx for Lustre/ONTAP instead.

### 16.7 VMware estate (Relocate then migrate)
VMware Cloud on AWS SDDC → HCX deployment and site pairing → network extension → bulk migration or vMotion → then plan native migration wave-by-wave from the SDDC.

### 16.8 .NET or Java monolith → containers (Refactor-lite)
App2Container `inventory` → `analyze` → `containerize` → `generate` ECS/EKS artefacts → externalise config to Parameter Store/Secrets Manager → externalise session state to ElastiCache/DynamoDB → CI/CD pipeline → deploy behind ALB.
*Watch for:* filesystem writes inside the container, Windows container base-image size, licence terms, logging to stdout.

### 16.9 Batch / cron estate (Replatform)
Inventory every scheduled job (this is always bigger than expected) → map to EventBridge Scheduler + Lambda / ECS Scheduled Tasks / AWS Batch / Step Functions → add retries, DLQs and alarms it never had → decommission the cron server.

### 16.10 SAP, mainframe, HPC
These are programmes in their own right: **AWS Launch Wizard for SAP** / Migration Hub Orchestrator SAP templates; **AWS Mainframe Modernization** (replatform to managed runtime, or refactor COBOL→Java); **ParallelCluster / Batch** for HPC. Treat as Retain until you have a dedicated workstream.

---

## 17. Post-migration: operate, optimize, modernize

### 17.1 Hypercare (first 1–2 weeks per wave)

- Daily check-in call with app owners.
- Heightened monitoring; lower alarm thresholds temporarily.
- A named engineer per app, with the runbook and the rollback plan in hand.
- Log every issue — these become runbook improvements and next-wave preventions.
- Do **not** start decommissioning the source until hypercare closes.

### 17.2 Steady-state operations baseline

Every migrated workload should have, before handover:
- CloudWatch dashboards and alarms (CPU, memory via agent, disk, app-level metrics), alarms routed to the on-call channel.
- **AWS Backup** plan applied by tag, with a **tested restore**.
- SSM Patch Manager baseline and maintenance window.
- Config rules and Security Hub findings triaged.
- Cost allocation tags correct, so someone owns the bill.
- Updated CMDB, architecture diagram and runbook. Documentation debt accrued during migration is real debt.

### 17.3 Optimization loop

```
   measure (CloudWatch, Cost Explorer, Compute Optimizer)
        │
        ▼
   right-size instances & volumes ──▶ commit (Savings Plans/RIs)
        │                                     │
        ▼                                     ▼
   remove waste (orphans, idle,        review architecture
   over-provisioned, unused EIPs)      (Well-Architected, Resilience Hub)
        │                                     │
        └────────────▶ modernize backlog ◀────┘
```

### 17.4 Decommission the source (the step everyone forgets)

The migration isn't finished — and the savings aren't real — until the old kit is off.

```
[ ] Hypercare closed, app owner signed off
[ ] Final backup/snapshot archived with a documented retention date
[ ] Source powered off but retained (dark period, 30–60 days)
[ ] Monitoring and backup jobs pointing at source disabled
[ ] DNS records cleaned up; old A/CNAME entries removed
[ ] Firewall rules for the old server removed
[ ] Licences reclaimed / support contracts cancelled
[ ] CMDB, asset register and diagrams updated
[ ] Hardware wiped (certificate of destruction if required) and disposed
[ ] Data-centre space, power and cross-connects released — tell finance
```

---

## 18. KPIs — how you prove success

| Category | Metric | Why it matters |
|---|---|---|
| **Progress** | Servers/apps migrated vs plan; waves on schedule | The exec dashboard number |
| **Velocity** | Servers per week; hours of effort per server | Proves the factory is working |
| **Quality** | Cutover success rate; rollbacks; P1 incidents in 30 days post-migration | Are we going fast or just going? |
| **Downtime** | Actual vs planned window per cutover | Credibility with the business |
| **Data integrity** | Validation pass rate; mismatched rows | Non-negotiable: target 100% |
| **Cost** | Actual AWS spend vs business case; DC cost eliminated | Did we deliver the case? |
| **Right-sizing** | % instances within Compute Optimizer's recommendation | Are savings real or theoretical? |
| **Retirement** | Servers retired (never migrated) | Free savings captured |
| **Decommission** | Source servers actually powered off | Savings realised, not just promised |
| **Performance** | Post vs pre-migration p95 response time | Users notice this before you do |
| **Adoption** | % workloads with backup, monitoring, tags, patching | Operational readiness |
| **People** | Team members certified; runbooks documented | Sustainability after the partner leaves |

---

## 19. Where and when to use each approach

### 19.1 By scenario

| Scenario | Recommended play |
|---|---|
| Data centre lease ends in 6 months | **Rehost** everything migratable; Retire aggressively; Relocate if it's a big VMware estate. Modernize later. |
| Hardware refresh due, no deadline pressure | Assess properly, then **Replatform** the databases and file servers while rehosting the rest |
| Cutting Oracle/SQL Server licence spend | **Replatform** heterogeneous with SCT + DMS; budget serious test time |
| Startup outgrowing a single VPS | **Refactor** — you're small enough to do it right |
| M&A: divest a business unit in 90 days | **Rehost/Relocate** into a clean new account structure; nothing clever |
| Email, HR, CRM, ticketing | **Repurchase** — stop running commodity software |
| A monolith blocking the product roadmap | **Refactor** with strangler fig, but only after it's stable in AWS |
| Mainframe / plant control / HSM-bound | **Retain** now; separate programme; consider Outposts |
| Dev/test estates | **Rehost** first as a low-risk pilot, then use them to practise automation |
| Global latency problems | Refactor toward CloudFront + multi-region, or use Local Zones |
| DR is on paper only | Rehost + **Elastic Disaster Recovery**, then test a real failover |

### 19.2 Sequencing advice

**Start with:** dev/test, internal tools, stateless web servers, and anything with an eager owner.
**Do in the middle:** the bulk of business applications, databases with good documentation, file servers.
**Save for last:** the crown-jewel revenue system, anything with a regulator watching, anything undocumented and unowned.

**Never:** refactor and migrate a critical system for the first time in the same weekend.

---

## 20. Anti-patterns

| Anti-pattern | Why it hurts | Do instead |
|---|---|---|
| **Skipping discovery** ("we know what we have") | You don't. The dependency you missed causes the outage. | 2–4 weeks of real data collection |
| **Lift-and-shift with no right-sizing** | You move your waste to a place where you pay monthly for it | Size from p95 utilization |
| **Refactoring everything** | Timeline explodes, deadline missed, credibility gone | Rehost/Replatform out, refactor after |
| **Migrating garbage** | Paying to run dead servers in a nicer building | Retire 10–20% first |
| **No tagging standard** | Nobody can attribute cost or find owners; retrofit is brutal | Tags enforced from day one via Config/SCP |
| **Overlapping CIDR with on-prem** | Forces re-IP mid-project | Plan the supernet before the first VPC |
| **Testing in production subnets** | Test instances send real emails and write to real systems | Isolated test subnet, no prod routes |
| **No performance baseline** | Can't defend against "the cloud is slow" | Capture before-metrics for a normal week |
| **Rollback plan written on the night** | Panic decisions, data loss | Written, reviewed and rehearsed at T-2d |
| **Big-bang everything** | One failure takes down the whole business | Waves with blast-radius limits |
| **Ignoring egress and cross-AZ data transfer** | Bill surprise in month two | Model it; use endpoints; keep chatty tiers in one AZ |
| **Forgetting to decommission** | Paying twice; savings never materialise | Decommission checklist with a finance sign-off |
| **Treating it as an infrastructure project** | App owners disengaged, testing inadequate | Business-led, app-owner-signed-off |
| **No hypercare** | Small issues become escalations | 1–2 weeks of heightened support per wave |
| **One hero doing everything** | Bus factor of one, burnout, no scale | Factory, runbooks, cross-training |

---

## 21. Glossary

| Term | Meaning |
|---|---|
| **6 R's / 7 R's** | Migration strategies: Retire, Retain, Rehost, Replatform, Repurchase, Refactor (+ Relocate) |
| **ADS** | AWS Application Discovery Service |
| **A2C** | AWS App2Container |
| **CAF** | AWS Cloud Adoption Framework |
| **CCoE** | Cloud Center of Excellence — the team that owns standards and enablement |
| **CDC** | Change Data Capture — streaming ongoing DB changes from transaction logs |
| **Cutover** | The moment production traffic moves to the target |
| **Dark period** | Time a source system stays powered off but not deleted, in case of rollback |
| **Disposition** | The chosen R for an application |
| **DMS** | AWS Database Migration Service |
| **DRS** | AWS Elastic Disaster Recovery |
| **DX** | AWS Direct Connect |
| **Heterogeneous migration** | Source and target database engines differ |
| **Homogeneous migration** | Same engine on both sides |
| **Hypercare** | Elevated support period immediately after cutover |
| **Landing zone** | The pre-built, secure, multi-account AWS foundation |
| **MAP** | Migration Acceleration Program |
| **MGN** | AWS Application Migration Service |
| **Migration factory** | Repeatable, automated process + team for migrating at scale |
| **MRA** | Migration Readiness Assessment |
| **PIT** | Point-in-time (recovery snapshot) |
| **RPO / RTO** | Recovery Point Objective (data loss tolerance) / Recovery Time Objective (downtime tolerance) |
| **SCT** | AWS Schema Conversion Tool |
| **Staging area** | Subnet + lightweight servers + EBS volumes where MGN holds replicated data |
| **Strangler fig** | Incrementally replacing a monolith by routing functions to new services |
| **TCO** | Total Cost of Ownership |
| **Wave** | A batch of applications migrated together |

---

## 22. Further reading

- AWS Prescriptive Guidance — Migration strategies and playbooks
- AWS Cloud Adoption Framework (CAF) whitepaper
- AWS Well-Architected Framework
- AWS Application Migration Service, DMS, DataSync, and Migration Hub user guides
- AWS Migration Acceleration Program (MAP) overview
- *Ahead in the Cloud*, Stephen Orban — the organisational side of migration
- AWS Database Migration Service Step-by-Step Walkthroughs

---

## Contributing

Found an error, or a gotcha this guide missed? Open an issue or a PR. Real-world failure stories are the most valuable contributions — they're what turns a doc into a runbook.

## License

MIT — use it, fork it, teach from it.

---

*Next: → [commands-cheatsheet.md](commands-cheatsheet.md) · [hands-on-labs.md](hands-on-labs.md) · [troubleshooting.md](troubleshooting.md)*
