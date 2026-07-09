# AWS Global Infrastructure — Regions, AZs, Shared Responsibility & Delivery Models
---
## Table of Contents

1. [High-Level Architecture & Service Flow](#1-high-level-architecture--service-flow)
2. [Core Features & Deep-Dive](#2-core-features--deep-dive)
3. [Step-by-Step Configuration & Implementation Guide](#3-step-by-step-configuration--implementation-guide)
4. [How to Use & Where to Use (Target Use Cases)](#4-how-to-use--where-to-use-target-use-cases)
5. [Frequently Missed Details / FAQ](#5-frequently-missed-details--faq)
6. [Commands Cheatsheet](#6-commands-cheatsheet)
7. [Hands-On Practical Labs](#7-hands-on-practical-labs)
8. [Troubleshooting Log](#8-troubleshooting-log)

---

## 1. High-Level Architecture & Service Flow

AWS's physical and logical footprint is organized as a **hierarchy**, from the largest geographic construct down to the smallest cache node. Understanding this hierarchy is the single most important prerequisite before you touch any AWS service, because it determines **latency, cost, compliance, and availability** for everything you build.

```
                              AWS GLOBAL INFRASTRUCTURE
                                        │
        ┌───────────────────────────────┼───────────────────────────────┐
        │                                │                                │
   AWS REGION                      AWS REGION                      AWS REGION
  (e.g. us-east-1)                (e.g. ap-south-1)               (e.g. eu-west-1)
        │
        ├── Availability Zone A (us-east-1a)  ── Data Center(s) ── Power/Cooling/Network
        ├── Availability Zone B (us-east-1b)  ── Data Center(s) ── Power/Cooling/Network
        ├── Availability Zone C (us-east-1c)  ── Data Center(s) ── Power/Cooling/Network
        │        (all interconnected via private, high-bandwidth, low-latency fiber)
        │
        ├── AWS Local Zones (optional extension closer to metro end-users)
        ├── AWS Wavelength Zones (embedded inside telco 5G networks)
        └── AWS Outposts (AWS hardware physically inside YOUR data center)

                                        │
                        ┌───────────────┴───────────────┐
                        │                                │
                 EDGE LOCATIONS                  REGIONAL EDGE CACHES
          (400+ global cache/PoP nodes)         (mid-tier cache layer)
                        │
                 Amazon CloudFront (CDN) ── serves static/dynamic content
                 to end-users from the nearest point of presence (PoP)
```

### Request Flow Example (a typical global web application)

1. **User in Mumbai requests `example.com`** → DNS (Route 53) resolves to the nearest **Edge Location**.
2. If content is cached (image, JS bundle, video) → served instantly from the **Edge Location** (CloudFront) — no trip to the origin Region.
3. If content is dynamic/uncached → CloudFront forwards the request over AWS's private backbone to the **origin Region** (e.g., `ap-south-1`, Mumbai Region).
4. Inside the Region, a **Load Balancer (ALB/NLB)** distributes the request across **multiple Availability Zones** for fault tolerance.
5. Compute (EC2/ECS/Lambda) in the chosen AZ processes the request, reads/writes to a **Multi-AZ database** (e.g., RDS Multi-AZ), and returns the response.
6. Response is cached at the Edge Location (if cacheable) for the next nearby user.

**Key architectural takeaway:** Every well-designed AWS architecture is built as concentric circles of redundancy — *Edge → Region → AZ → Data Center* — so that the failure of any single circle does not take down the whole system.

---

## 2. Core Features & Deep-Dive

### 2.1 AWS Regions

- A **Region** is a fully isolated geographic area (e.g., `us-east-1` = N. Virginia, `ap-south-1` = Mumbai, `eu-west-1` = Ireland).
- Regions **never share data automatically** — replication across Regions is always something *you* explicitly configure (e.g., S3 Cross-Region Replication, DynamoDB Global Tables, RDS Read Replicas across Regions).
- **Total footprint (as of mid-2026):** AWS operates in the range of **37+ geographic Regions** and **117+ Availability Zones** worldwide, and this number keeps growing every year as AWS opens new Regions (recent additions include Asia Pacific (Taipei), Asia Pacific (New Zealand), and the AWS European Sovereign Cloud). Because this is a constantly moving number, always check the **[official AWS Global Infrastructure page](https://aws.amazon.com/about-aws/global-infrastructure/)** for the current live count rather than relying on any static figure — including this document.
- **Special/restricted Region types:**
  - **AWS GovCloud (US):** Isolated Regions for US government workloads with strict compliance (ITAR, FedRAMP).
  - **AWS China Regions (Beijing / Ningxia):** Operated by licensed local partners (Sinnet, NWCD), require a *separate* China-specific AWS account — not accessible from a standard global account.
  - **AWS European Sovereign Cloud:** A newer, fully independent partition for EU data-sovereignty requirements.
- **Opt-in Regions:** Regions launched after March 20, 2017 (e.g., `ap-east-1` Hong Kong, `me-south-1` Bahrain) are **disabled by default** and must be manually enabled per account before you can deploy into them. Older Regions (pre-2017) are enabled by default.
- **Region selection criteria:** latency to end-users, data residency/sovereignty laws (GDPR, HIPAA, local regulations), service/feature availability (not every service launches in every Region simultaneously), and pricing (cost varies significantly by Region).

### 2.2 Availability Zones (AZs)

- Each Region contains a **minimum of 3 Availability Zones** (some Regions have more — up to 6 in the largest Regions).
- An AZ = **one or more discrete, physically separate data centers**, each with independent power substations, cooling, networking, and physical security.
- AZs within the same Region are **physically separated by a meaningful distance** (many kilometers) to avoid correlated failure from fire, flood, or regional power loss — but are kept **within ~100 km (60 miles)** of each other so that **synchronous replication** remains possible with single-digit-millisecond latency.
- All AZs in a Region are linked by **fully redundant, dedicated, encrypted metro fiber** — this is what allows services like RDS Multi-AZ or EBS to replicate data synchronously without noticeable application latency.
- **AZ naming vs. AZ ID (important distinction):**
  - The **AZ Name** (e.g., `us-east-1a`, `us-east-1b`) is just a *human-friendly, per-account label* — Region code + a letter suffix (`a`, `b`, `c`...).
  - The **AZ ID** (e.g., `use1-az1`, `use1-az2`) is the *true, fixed physical identifier* that refers to the same physical location for every AWS account.
  - **Why this matters:** For accounts created *before November 2025*, AWS **deliberately randomizes** the mapping between AZ Name and physical AZ **per account**. This means *"us-east-1a" in your account may not be the same physical data center as "us-east-1a" in someone else's account*. AWS does this intentionally to **spread customer workloads evenly across the real physical AZs** — if every account's "1a" mapped to the same physical location, that one data center would receive a disproportionate share of all AWS customer traffic.
  - Accounts created **from November 2025 onward** get a **consistent** Name-to-physical mapping across accounts.
  - **Practical implication:** When you need to guarantee two AWS accounts are (or are not) using the same physical AZ — e.g., for cross-account disaster recovery planning — always compare **AZ IDs**, never AZ Names. You get the AZ ID via `aws ec2 describe-availability-zones`.
- **Scaling/design limits to know:**
  - You cannot choose a *specific* physical AZ by request — AWS assigns based on the account-scoped letter mapping.
  - Not all instance types/services are available in every AZ within a Region.
  - Some accounts get access to only 2 AZs in older/smaller Regions (e.g., historically `us-west-1`).

### 2.3 Local Zones, Wavelength, and Outposts (the "edge" of core Regions)

| Construct | What it is | Best for |
|---|---|---|
| **AWS Local Zones** | A small extension of a parent Region placed in a major metro area, running a subset of services (EC2, EBS, VPC, ELB) | Single-digit-millisecond latency apps: media rendering, real-time gaming, EDA, ML inference near users |
| **AWS Wavelength** | AWS compute/storage embedded *inside* telco 5G network infrastructure | Ultra-low-latency mobile apps (AR/VR, connected vehicles, IoT) that must avoid the public internet hop entirely |
| **AWS Outposts** | Physical AWS-managed server rack installed in *your own* data center or colocation facility | Hybrid workloads that must stay on-premises for latency, data residency, or local system dependencies, while still using native AWS APIs |

### 2.4 Edge Locations & Amazon CloudFront (CDN)

- **Edge Locations** are smaller, globally distributed points of presence (PoPs) — far more numerous than Regions (typically 400+ worldwide).
- They do **not** run your full application logic — they exist purely to **cache and serve static/dynamic content** close to end-users via **Amazon CloudFront**.
- **Regional Edge Caches** sit as a mid-tier between Edge Locations and the origin Region — larger cache, longer retention, reduces origin load for less-popular content.
- Typical cached content: images, video, JS/CSS bundles, API responses (with proper cache headers), and even some dynamic content via Lambda@Edge / CloudFront Functions.
- **Benefit:** Reduces perceived latency for global users from hundreds of milliseconds down to tens of milliseconds, and offloads traffic from your origin infrastructure.

### 2.5 The Shared Responsibility Model

| AWS Responsibility — "Security **OF** the Cloud" | Customer Responsibility — "Security **IN** the Cloud" |
|---|---|
| Physical security, power, and maintenance of data centers, servers, and cabling | Encrypting data at rest and in transit |
| Maintaining physical host machines, storage arrays, and the virtualization/hypervisor layer | Patching and updating the guest OS on virtual servers (e.g., EC2 instances) |
| Securing and operating the global network backbone connecting Regions/AZs | Configuring Security Groups, Network ACLs, VPC routing, and firewalls |
| Ensuring availability of managed foundation services' control planes (IAM, S3, DynamoDB, etc.) | Managing IAM users/roles/policies, password policies, and MFA |
| Decommissioning storage media securely | Classifying data correctly and applying appropriate access controls |
| Environmental controls (fire suppression, climate control) | Application-level security (input validation, dependency patching, secrets management) |

- **The dividing line shifts depending on the service model.** For IaaS (EC2), you own more of the stack (OS, middleware, app). For managed/serverless services (Lambda, S3), AWS absorbs more of the operational burden, but you are *still* responsible for your data, access configuration, and application logic.
- This is **not a legal disclaimer you can ignore** — the majority of real-world cloud security breaches are due to **customer-side misconfiguration** (public S3 buckets, over-permissive IAM policies, unpatched OS/application layers), not AWS infrastructure failures.

### 2.6 Cloud Delivery / Service Models

| Model | What AWS manages | What you manage | Example services |
|---|---|---|---|
| **IaaS** (Infrastructure as a Service) | Physical hardware, virtualization | OS, middleware, runtime, application, data, scaling policy | Amazon EC2, Amazon VPC, Amazon EBS |
| **PaaS** (Platform as a Service) | Hardware, OS, and runtime environment | Application code and configuration only | AWS Elastic Beanstalk, AWS App Runner |
| **SaaS / Serverless** | Almost everything, including scaling and execution | Code/config and data only — no server management at all | AWS Lambda, Amazon S3, Amazon DynamoDB, Amazon SES |

- **Trade-off spectrum:** IaaS = maximum control, maximum operational burden. Serverless = minimum control over infrastructure, minimum operational burden, but potential trade-offs in cold-start latency, vendor-specific execution limits, and per-invocation cost model.

---

## 3. Step-by-Step Configuration & Implementation Guide

This section walks through the **foundational account and Region setup** every production AWS environment should start with.

### Step 1 — Secure the Root Account
1. Create the AWS account with a **dedicated, non-personal root email alias** (e.g., `aws-root@yourcompany.com`), not an individual's inbox.
2. Immediately enable **MFA (Multi-Factor Authentication)** on the root user — hardware key (e.g., YubiKey) preferred over virtual MFA for production accounts.
3. **Never use the root account for daily operations.** Create an IAM Administrator user/role immediately and lock the root credentials away (e.g., in a password manager or vault with break-glass access procedures).

### Step 2 — Set Up AWS Organizations (Multi-Account Strategy)
```bash
aws organizations create-organization --feature-set ALL
```
4. Create separate accounts for `Security`, `Log-Archive`, `Shared-Services`, `Production`, and `Non-Production` following the **AWS Landing Zone / Control Tower** best practice.
5. Apply **Service Control Policies (SCPs)** at the Organizational Unit (OU) level to enforce guardrails (e.g., deny disabling CloudTrail, restrict Regions).

### Step 3 — Enable Required Regions
By default, only Regions launched before March 2017 are enabled. To use a newer opt-in Region:
```bash
aws account enable-region --region-name ap-east-1
aws account list-regions --region-opt-status-contains ENABLED
```

### Step 4 — Discover Availability Zones for Your Target Region
```bash
aws ec2 describe-availability-zones \
  --filters Name=zone-type,Values=availability-zone \
  --region ap-south-1 \
  --query "AvailabilityZones[].{Name:ZoneName,ID:ZoneId,State:State}"
```
Example output:
```json
[
  { "Name": "ap-south-1a", "ID": "aps1-az1", "State": "available" },
  { "Name": "ap-south-1b", "ID": "aps1-az2", "State": "available" },
  { "Name": "ap-south-1c", "ID": "aps1-az3", "State": "available" }
]
```
> Always capture the **AZ ID**, not just the Name, in your infrastructure-as-code and documentation — especially in multi-account setups.

### Step 5 — Design the VPC Across Multiple AZs
6. Create **one VPC per Region**, and **subnets spread across at least 2–3 AZs** (public + private subnet pairs per AZ) — never place all resources in a single AZ for anything production-facing.
```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region ap-south-1

aws ec2 create-subnet --vpc-id vpc-xxxx --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a
aws ec2 create-subnet --vpc-id vpc-xxxx --cidr-block 10.0.2.0/24 --availability-zone ap-south-1b
aws ec2 create-subnet --vpc-id vpc-xxxx --cidr-block 10.0.3.0/24 --availability-zone ap-south-1c
```

### Step 6 — Deploy Compute & Database Across AZs
7. Place an **Application Load Balancer** across all 3 public subnets (spans all 3 AZs automatically).
8. Configure an **Auto Scaling Group** with instances distributed across the 3 private subnets.
9. Deploy **RDS in Multi-AZ mode** for automatic synchronous standby failover:
```bash
aws rds create-db-instance \
  --db-instance-identifier prod-db \
  --multi-az \
  --engine postgres \
  --db-instance-class db.r6g.large \
  --allocated-storage 100
```

### Step 7 — Add a CloudFront Distribution for Edge Delivery
```bash
aws cloudfront create-distribution \
  --origin-domain-name my-alb-123456.ap-south-1.elb.amazonaws.com \
  --default-root-object index.html
```
10. Attach an **AWS WAF Web ACL** to the distribution for edge-level security filtering.
11. Configure cache behaviors and TTLs appropriately per content type (static assets = long TTL, API responses = short/no cache as needed).

### Step 8 — Apply the Shared Responsibility Checklist
- [ ] Root account MFA enabled and root keys not in daily use
- [ ] IAM least-privilege policies (no wildcard `*:*` policies in production)
- [ ] Security Groups scoped to minimum required ports/sources
- [ ] EBS volumes and S3 buckets encrypted at rest (KMS)
- [ ] TLS enforced in transit (ALB listeners, CloudFront viewer protocol policy)
- [ ] CloudTrail enabled organization-wide and log files sent to a dedicated, access-restricted Log-Archive account
- [ ] Config rules / Security Hub enabled for continuous compliance monitoring

---

## 4. How to Use & Where to Use (Target Use Cases)

### When to design for **Multi-AZ** (do this for virtually every production workload)
- Any workload with an **SLA/uptime commitment** (e.g., 99.9%+).
- Databases (RDS, ElastiCache) that cannot tolerate a single data-center failure causing data loss or downtime.
- Stateless compute tiers behind a load balancer — trivially cheap insurance against a single AZ outage.

### When to design for **Multi-Region**
- **Regulatory/data-sovereignty requirements** force certain data to stay within a specific country/region (e.g., GDPR in `eu-west-1`/`eu-central-1`, India data localization in `ap-south-1`).
- **True disaster recovery** requirements where an entire Region-level event (rare, but has happened) must not take the business offline — active-active or active-passive Region pairs (e.g., `us-east-1` + `us-west-2`).
- **Global-latency-sensitive** applications serving users on multiple continents.

### When to use **Local Zones**
- Applications needing single-digit-millisecond latency to a specific metro area (e.g., media/broadcast editing suites, real-time multiplayer gaming servers) but not requiring a full Region's service catalog.

### When to use **Wavelength**
- Mobile-carrier-dependent, ultra-low-latency use cases: AR/VR streaming, connected/autonomous vehicle telemetry, smart-factory IoT over 5G — where even the hop to a Local Zone/Region adds unacceptable latency.

### When to use **Outposts**
- Regulatory or technical requirements mandate the workload stay physically on customer premises (e.g., manufacturing floor systems, government facilities with air-gap-adjacent requirements, hospitals with local imaging equipment latency needs) but you still want consistent AWS APIs/tools.

### When NOT to use Multi-Region (the case most people skip)
- If your application has **no regulatory driver** and **no truly global user base**, multi-Region adds substantial complexity (data replication conflicts, higher latency for cross-Region writes, doubled operational surface) for marginal benefit over a well-built Multi-AZ single-Region design. Most startups and mid-size companies over-engineer this — Multi-AZ alone already covers the vast majority of real-world failure scenarios.

### Choosing IaaS vs. PaaS vs. Serverless
| Criteria | Choose IaaS (EC2) | Choose PaaS (Elastic Beanstalk) | Choose Serverless (Lambda) |
|---|---|---|---|
| Team has strong ops/SRE capability | ✅ | Optional | Not required |
| Need full OS-level control (custom kernels, licensing, legacy apps) | ✅ | ❌ | ❌ |
| Want fastest path from code to deployed app | ❌ | ✅ | ✅ |
| Highly variable/spiky/event-driven traffic | Possible w/ ASG | Possible | ✅ Best fit |
| Long-running processes (>15 min), stateful workloads | ✅ | ✅ | ❌ (Lambda hard limit: 15 min) |
| Cost model preference: pay-for-idle-capacity vs pay-per-execution | Pay for provisioned capacity | Pay for provisioned capacity | Pay per invocation/duration |

---

## 5. Frequently Missed Details / FAQ

**Q: How many Availability Zones does AWS have right now?**
A: This is a genuinely moving target, not a fixed number — AWS is continuously opening new Regions and AZs. As reference points: AWS reported **117 Availability Zones across 37 geographic Regions** at the time of the Asia Pacific (Taipei) Region launch in June 2025, with the Asia Pacific (New Zealand) Region and the AWS European Sovereign Cloud added shortly after. By mid-2026 the true count is somewhat higher still. **Do not hardcode this number anywhere in production documentation or code** — always query it live via `aws ec2 describe-regions` / `describe-availability-zones`, or check the official [AWS Global Infrastructure page](https://aws.amazon.com/about-aws/global-infrastructure/), since this document itself will go stale.

**Q: Why is an AZ named like `us-east-1a` — what does the naming actually mean?**
A: Breaking down the name:
- `us-east-1` = the **Region code** (US, East geography, Region #1 = N. Virginia).
- `a` = a **letter suffix** identifying one specific AZ within that Region for *your account*.

The subtle and frequently-missed part: **this letter mapping is account-specific**, not global — at least for accounts created before November 2025. AWS intentionally randomizes which physical AZ gets labeled `a` vs `b` vs `c` on a per-account basis, specifically **to prevent every customer from concentrating their "first" resources in the same physical data center** (which would create uneven load across AWS's physical facilities). The **AZ ID** (e.g., `use1-az1`) is the fixed, true physical identifier that stays the same across every account — use AZ IDs whenever physical-location accuracy matters (e.g., coordinating shared infrastructure across multiple AWS accounts, or verifying two teams aren't unknowingly both single-homed in the same physical AZ).

**Q: What other things are commonly missed by beginners on this topic?**
- **AZs are not "one data center each."** An AZ can itself contain **more than one physical data center** if their combined power/network profile is isolated enough to be treated as one fault domain.
- **Not every Region has the same number of AZs** — most modern Regions launch with 3, but some Regions have grown to 4, 5, or 6 over time as demand increased.
- **Region ≠ Service parity.** New/smaller Regions often lack some of the newest AWS services at launch — always verify service availability per Region before committing to a design (check the "Region Table" on AWS's site).
- **Edge Locations are not compute environments.** You cannot run EC2 or containers in a CloudFront Edge Location — only cache/compute-at-edge via Lambda@Edge/CloudFront Functions, which have their own restricted execution model.
- **China Regions and GovCloud require entirely separate AWS accounts** and are not visible or reachable from a standard global AWS account, even if you have billing permissions elsewhere.
- **The Shared Responsibility Model shifts per service** — it is not a fixed 50/50 split. The more "managed" the service (Lambda vs. EC2), the more of the stack AWS absorbs, but customer responsibility for **data classification, IAM, and application-layer security never disappears**, no matter how serverless the architecture becomes.
- **Multi-AZ ≠ Multi-Region.** Multi-AZ protects against a data-center-level failure within one geography; it does **not** protect against a full-Region event or satisfy data-residency requirements that mandate a specific *country*.

---

## 6. Commands Cheatsheet

Quick-reference AWS CLI commands for working with global infrastructure. Keep this section updated as you discover more.

| Task | Command | Notes |
|---|---|---|
| List all enabled Regions for your account | `aws account list-regions --region-opt-status-contains ENABLED --query "Regions[].RegionName"` | Only shows Regions enabled for *your* account |
| List every Region AWS offers (enabled or not) | `aws account list-regions --query "Regions[].RegionName"` | Includes opt-in Regions you haven't enabled yet |
| Enable an opt-in Region | `aws account enable-region --region-name ap-east-1` | Takes a few minutes to propagate |
| Disable a Region you don't use | `aws account disable-region --region-name ap-east-1` | Reduces attack surface / accidental resource sprawl |
| List AZs (Name + AZ ID) for a Region | `aws ec2 describe-availability-zones --region ap-south-1 --query "AvailabilityZones[].{Name:ZoneName,ID:ZoneId}"` | Always log the **AZ ID**, not just the Name |
| Check current default Region for CLI | `aws configure get region` | Confirms which Region your commands will target by default |
| Set default Region for CLI | `aws configure set region ap-south-1` | Avoids needing `--region` on every command |
| Check which account/identity is active | `aws sts get-caller-identity` | Always run this before executing destructive commands, to confirm you're in the right account |
| List Local Zones opted in for your account | `aws ec2 describe-availability-zones --all-availability-zones --filters Name=zone-type,Values=local-zone --query "AvailabilityZones[].ZoneName"` | Local Zones must be explicitly opted in per Region |
| Check service availability in a Region | `aws ssm get-parameters-by-path --path /aws/service/global-infrastructure/regions/ap-south-1/services --region us-east-1` | Confirms whether a specific service is launched in a given Region |

---

## 7. Hands-On Practical Labs

Three progressively harder labs. Do them in order — each builds on the previous one's output. Record your actual results in [Section 8](#8-troubleshooting-log) as you go.

### Lab 1 (Beginner): Explore Your Account's Regions and AZs

**Goal:** Get comfortable reading your own account's global infrastructure footprint.

**Prerequisites:** AWS CLI installed and configured with an IAM user (not root) that has at least `ec2:Describe*` and `account:ListRegions` permissions.

**Steps:**
1. Run `aws sts get-caller-identity` and confirm you're in the correct account.
2. Run `aws account list-regions --region-opt-status-contains ENABLED --query "Regions[].RegionName"` and count how many Regions are enabled by default.
3. Pick one Region you've never used (e.g., `ap-south-1` if you're US-based) and run the `describe-availability-zones` command from Section 6.
4. Note down both the **AZ Name** and **AZ ID** for each AZ returned.

**Expected Outcome:** A list of enabled Regions, and a Name→ID mapping table for at least one Region's AZs.

**What this teaches you:** How to verify infrastructure facts yourself via CLI instead of trusting static documentation (including this file) — the single most important habit for working with a constantly-changing cloud platform.

---

### Lab 2 (Intermediate): Build a Multi-AZ VPC and Verify Fault Isolation

**Goal:** Actually construct the Multi-AZ network foundation described in Section 3, and prove to yourself that AZs are independent.

**Prerequisites:** Lab 1 complete; IAM permissions for VPC/EC2 create actions.

**Steps:**
1. Create a VPC and 3 subnets across 3 different AZs using the commands in Step 5 of Section 3.
2. Launch one small EC2 instance (e.g., `t3.micro`) in each of the 3 subnets.
3. Run `aws ec2 describe-instances --query "Reservations[].Instances[].{ID:InstanceId,AZ:Placement.AvailabilityZone}"` and confirm each instance landed in a different AZ.
4. Stop (don't terminate) the instance in AZ #1 only, and confirm via the console/CLI that the instances in AZ #2 and AZ #3 are completely unaffected.
5. Clean up: terminate all 3 instances and delete the VPC/subnets to avoid ongoing charges.

**Expected Outcome:** Three EC2 instances confirmed running in three distinct AZs, and direct proof that stopping one has zero effect on the others.

**What this teaches you:** This is the practical, hands-on proof of the "fault isolation" claim in Section 1 — you're not just reading that AZs are independent, you're demonstrating it yourself.

---

### Lab 3 (Production-Scenario): Multi-AZ RDS Failover Test

**Goal:** Simulate a real production failover event and measure actual downtime.

**Prerequisites:** Labs 1–2 complete; budget awareness (RDS Multi-AZ costs more than single-AZ — clean up immediately after).

**Steps:**
1. Deploy an RDS Multi-AZ instance using the command in Step 6 of Section 3.
2. Once available, note the **primary AZ** via `aws rds describe-db-instances --db-instance-identifier prod-db --query "DBInstances[].AvailabilityZone"`.
3. Connect a simple script or client that writes a timestamp to the database once per second.
4. Trigger a manual failover: `aws rds reboot-db-instance --db-instance-identifier prod-db --force-failover`.
5. Watch your write script's logs — measure exactly how many seconds of write failures occur during the failover.
6. Confirm the new primary AZ differs from the original (`describe-db-instances` again).
7. **Clean up immediately:** `aws rds delete-db-instance --db-instance-identifier prod-db --skip-final-snapshot` to stop billing.

**Expected Outcome:** A measured failover time (typically 60–120 seconds for RDS Multi-AZ), and confirmation the standby in a different AZ was promoted to primary.

**What this teaches you:** This is the exact skill interviewers probe for in DevOps/SRE roles — "have you actually seen a failover happen, and do you know what it costs you in downtime?" Now you have a real, measured answer instead of a textbook one.

---

## 8. Troubleshooting Log

Real issues you're likely to hit while doing the labs above, plus fixes. Add your own entries here as you encounter new ones.

### Issue: `An error occurred (OptInRequired)` when using a newer Region
**Context:** Trying to launch resources in a Region like `ap-east-1` (Hong Kong) or `me-south-1` (Bahrain) without enabling it first.
**Error:**
```
An error occurred (OptInRequired) when calling the DescribeInstances operation:
You are not subscribed to this service in this region
```
**Root cause:** Regions launched after March 2017 are opt-in only and disabled by default per account.
**Fix:** Run `aws account enable-region --region-name <region>` and wait a few minutes for propagation before retrying.
**Prevention:** Check `aws account list-regions --region-opt-status-contains ENABLED` before scripting any deployment into an unfamiliar Region.

---

### Issue: EC2 instance launch fails with `Insufficient capacity` in one specific AZ
**Context:** Launching an instance with a specific `--availability-zone` flag pinned to one AZ.
**Error:**
```
An error occurred (InsufficientInstanceCapacity) when calling the RunInstances operation
```
**Root cause:** A specific AZ can temporarily run out of capacity for a specific instance type/size, especially for less-common instance families.
**Fix:** Either omit the AZ constraint and let EC2 choose, switch instance type/size, or retry in a different AZ within the same Region.
**Prevention:** For Auto Scaling Groups, span multiple AZs so capacity constraints in one AZ don't block scaling entirely.

---

### Issue: RDS Multi-AZ failover takes longer than expected
**Context:** Ran Lab 3, expected near-instant failover, saw 60–120+ seconds of write failures.
**Error:** No hard error — just elevated connection timeouts/refused connections during the window.
**Root cause:** Multi-AZ failover for RDS is **not instantaneous** — it involves DNS re-pointing to the new primary endpoint, and depending on engine (MySQL/PostgreSQL/etc.), can take 60 seconds to a few minutes.
**Fix:** Design application-layer retry logic with exponential backoff for database connections; don't assume zero-downtime from Multi-AZ alone.
**Prevention:** For workloads that truly cannot tolerate any failover gap, look at Aurora (faster failover, ~30s) or read replicas with application-level routing, not just standard RDS Multi-AZ.

---

### Issue: Confusion when comparing AZs across two AWS accounts
**Context:** Two team accounts both show "us-east-1a" and assumptions were made that they're the same physical location.
**Error:** No error — a silent design mistake (e.g., accidentally co-locating "redundant" resources in the same physical AZ).
**Root cause:** AZ Names are mapped independently per account (for accounts created before November 2025); "us-east-1a" is not guaranteed to be the same physical AZ across accounts.
**Fix:** Always compare **AZ IDs** (`describe-availability-zones` → `ZoneId` field), never AZ Names, when coordinating placement across multiple AWS accounts.
**Prevention:** Bake AZ ID checks into any cross-account architecture review or disaster-recovery runbook.

---

### Issue: CloudFront distribution serves stale content after an update
**Context:** Updated an S3/ALB origin's content but users still see the old version.
**Error:** No error — just incorrect/old content being served.
**Root cause:** CloudFront caches at Edge Locations based on TTL settings; content isn't automatically refreshed until the TTL expires.
**Fix:** Create a cache invalidation: `aws cloudfront create-invalidation --distribution-id <ID> --paths "/*"`.
**Prevention:** Use versioned file names (e.g., `app.v2.js`) for static assets instead of relying on invalidations, and set shorter TTLs for content that changes frequently.

---

### Next Module Suggestion
Now that global infrastructure, shared responsibility, and delivery models are covered — including hands-on proof via the labs above — the natural next module in the structured learning path is:
**`01-foundations/02-iam-deep-dive.md`** — Identity and Access Management (users, groups, roles, policies, permission boundaries, and the principle of least privilege) — since IAM is the customer's single largest responsibility inside the Shared Responsibility Model, and every lab above depended on IAM permissions you'll now understand in depth.
