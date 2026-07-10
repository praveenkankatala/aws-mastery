# AWS Load Balancing & Auto Scaling — End-to-End Practical Guide

![AWS](https://img.shields.io/badge/AWS-ELB%20%26%20ASG-orange?logo=amazon-aws)
![Status](https://img.shields.io/badge/Type-Hands--On%20Notes-blue)
![Level](https://img.shields.io/badge/Level-Beginner%20to%20Advanced-green)

A complete, practical, end-to-end reference covering **Elastic Load Balancing (ELB)** and **EC2 Auto Scaling Groups (ASG)** — concepts, architecture, real demos, commands, and troubleshooting. Built from hands-on notes and organized for quick review or interview prep.

## 📁 Repo Structure

| File | Purpose |
|---|---|
| `README.md` | You are here — concepts, architecture, and theory |
| [`commands-cheatsheet.md`](./commands-cheatsheet.md) | Copy-paste ready CLI, Terraform, and curl commands |
| [`hands-on-labs.md`](./hands-on-labs.md) | Step-by-step labs to build and test everything yourself |
| [`troubleshooting.md`](./troubleshooting.md) | Common failures, gotchas, and how to fix them |

---

## 📚 Table of Contents

1. [Elastic Load Balancing (ELB) Fundamentals](#1-elastic-load-balancing-elb-fundamentals)
2. [Sticky Sessions (Session Affinity)](#2-sticky-sessions-session-affinity)
3. [Auto Scaling Group (ASG) Fundamentals](#3-auto-scaling-group-asg-fundamentals)
4. [Launch Template vs Launch Configuration](#4-launch-template-vs-launch-configuration)
5. [Cooldown Periods & Instance Warmup](#5-cooldown-periods--instance-warmup)
6. [Lifecycle Hooks](#6-lifecycle-hooks)
7. [Scaling Policies](#7-scaling-policies)
8. [Self-Healing & AZ Rebalancing](#8-self-healing--az-rebalancing)
9. [Termination Policy & Instance Refresh](#9-termination-policy--instance-refresh)
10. [Suspending ASG Processes](#10-suspending-asg-processes)
11. [Advanced ELB Topics](#11-advanced-elb-topics)
12. [Gateway Load Balancer (GWLB)](#12-gateway-load-balancer-gwlb)
13. [Kubernetes Integration — AWS Load Balancer Controller](#13-kubernetes-integration--aws-load-balancer-controller)
14. [Quick-Reference Key Points](#14-quick-reference-key-points)

---

## 1. Elastic Load Balancing (ELB) Fundamentals

An ELB routes incoming client traffic across multiple healthy backend targets. Four pillars work together end-to-end:

```
[ Client Request ]
       │
       ▼
 1. LOAD BALANCER (Public/Private Virtual Infrastructure)
       │
       ▼
 2. LISTENERS (Evaluates Protocol & Port, e.g., HTTPS 443)
       │
       ▼
 3. ROUTING RULES (Inspects Headers/Paths, e.g., /api/*)
       │
       ▼
 4. TARGET GROUPS (Routes to Healthy Backends, e.g., EC2/ECS)
```

### A. The Load Balancer (Entry Point)
A managed, highly available cluster of proxy nodes spread across multiple Availability Zones (AZs).

- **Internet-facing**: lives in public subnets, has a public DNS name, routes internet traffic to backends.
- **Internal**: lives in private subnets, uses private IPs to route traffic between internal tiers (e.g., web layer → internal payments API).

### B. Listeners
A process that checks for connection requests, defined by:
- **Protocol**: HTTP, HTTPS, TCP, UDP, TLS
- **Port**: e.g., 80 for web traffic, 443 for SSL

### C. Rules
Attached to listeners. Every listener has a **Default Rule** (catch-all); an ALB also supports prioritized custom rules.
- **Conditions ("If")**: path pattern (`/images/*`), host header (`api.yourdomain.com`), HTTP header, source IP range
- **Actions ("Then")**: forward to a target group, redirect (e.g., force HTTP → HTTPS), or return a fixed response directly from the edge

### D. Target Groups & Health Checks
A logical collection of backend resources that receive forwarded traffic.
- **Target types**: Instance ID (EC2), IP Address (private VPC IPs, containers, on-prem via Direct Connect), or Lambda Functions
- **Health checks**: configured at the target group level; the LB pings a path (e.g. `/health`) periodically. Failing targets are dynamically pulled out of rotation until they recover.

---

## 2. Sticky Sessions (Session Affinity)

By default, an ALB uses **round-robin** — every request/refresh can land on a different backend, which breaks apps that store session state in local server memory (e.g., a shopping cart).

**Enabling stickiness** on the target group (duration-based) makes the ALB inject a managed cookie named **`AWSALB`** into the first response:

```
[ Client ] ──( 1. Request )──> [ ALB ] ──> [ Server A ]
[ Client ] <──( 2. Response + Set-Cookie: AWSALB=xyz )── [ ALB ]
```

Once the client sends that cookie back on subsequent requests, the ALB **bypasses round-robin** and routes strictly back to the same backend that issued the cookie — for as long as the cookie is valid and present.

Full walkthrough (curl cookie jar + browser DevTools verification) is in [`hands-on-labs.md`](./hands-on-labs.md#lab-2--prove-sticky-sessions-work).

---

## 3. Auto Scaling Group (ASG) Fundamentals

An ASG is a logical collection of EC2 instances that automatically launches/terminates instances based on policies, schedules, and health status — keeping your app available while matching capacity to demand.

```
[ Launch Template ] ──> Defines WHAT to launch (AMI, Type, Security Groups)
         │
         ▼
[ Auto Scaling Group ] ──> Defines WHERE and HOW MANY to launch
   ├── Size Limits (Min: 2, Desired: 4, Max: 10)
   └── Subnets (AZ-A, AZ-B, AZ-C)
```

### Group Size Attributes
| Attribute | Meaning |
|---|---|
| **Minimum Size** | Absolute floor — the ASG never drops below this, even with zero traffic |
| **Maximum Size** | Absolute ceiling — protects against runaway cost during traffic spikes/DDoS |
| **Desired Capacity** | The target instance count *right now*; scaling policies move this between Min and Max |

### Instance Lifecycle States

```
[Pending] ──> [Pending:Wait] ──> [Pending:Proceed] ──> [InService]
                                                           │
 [Terminating] <── [Terminating:Wait] <── [Terminating:Proceed] ◄┘
```

- **Pending**: ASG detects a capacity need and requests a new EC2 instance.
- **InService**: instance passes EC2 status checks, runs User Data, attaches to the load balancer's target group, and starts taking traffic.
- **Terminating**: a scale-in policy fires or a health check fails; the ASG removes the instance from the target group and shuts it down.

The `Pending:Wait` and `Terminating:Wait` states are where **Lifecycle Hooks** (Section 6) inject custom automation.

---

## 4. Launch Template vs Launch Configuration

Both are blueprints an ASG uses to launch EC2 instances — but they represent two different eras.

### Launch Template (modern, recommended)
Immutable, **version-controlled** config defining:
- **AMI** — OS/base image
- **Instance Type** — hardware profile
- **Network Settings** — VPC, public IP rules, Security Groups
- **Storage (EBS)** — size, performance (gp3/io2), encryption
- **User Data** — boot-time script (install Docker, agents, pull code, etc.)

Key advantage: **versioning**. Patch your AMI or User Data by creating Version 2, 3, etc. — point the ASG at `$Latest` or `$Default` for automated rolling updates, no need to rebuild from scratch.

### Launch Configuration (legacy)
- **No versioning** — immutable and unmodifiable. Any change requires creating a brand-new configuration and repointing the ASG.
- **Feature gaps** — no support for T2/T3 Unlimited, Dedicated Hosts, Elastic Graphics, multiple instance types in one ASG, or advanced Spot allocation strategies.
- ⚠️ AWS actively discourages new use of Launch Configurations; new EC2 features are built for Launch Templates only.

### Side-by-Side Comparison

| Feature | Launch Template ✅ | Launch Configuration ⚠️ |
|---|---|---|
| Versioning | Supported (`$Latest`, `$Default`) | Not supported — must recreate entirely |
| Upgrades | Seamless via version tag | Requires repointing ASG to a new resource |
| Spot + On-Demand mix | Supported | Difficult/limited |
| Multiple instance types | Supported (fallback list) | Restricted to one type |
| Standalone EC2 launch | Yes, via console/CLI | ASG-only |

### Operational Flow
```
[CloudWatch Alarm] ──> [ASG Signals Scale-Out] ──> [ASG Fetches Launch Template v3] ──> [EC2 Launches Fresh Instances]
```
The ASG queries the assigned template/version, calls the EC2 API with those parameters, the instance boots, runs User Data, passes health checks, and joins traffic.

---

## 5. Cooldown Periods & Instance Warmup

A **Cooldown Period** prevents an ASG from launching/terminating *more* instances before a previous scaling action has had time to take effect — avoiding **thrashing** (rapid capacity fluctuation).

### The problem without cooldown
| Minute | Event |
|---|---|
| 0 | Traffic spikes, CPU hits 85% |
| 1 | CloudWatch alarm fires, ASG launches 1 instance |
| 2 | New instance still booting; existing CPU still 85% |
| 3 | CloudWatch samples again — still 85% |

Without a cooldown, the ASG would keep adding instances every sample interval, drastically over-provisioning by the time the first batch finishes booting.

### Mechanism
```
[Alarm Breached] ──> [Instance Starts Launching] ──> [Instance Fully Boots] ──> [Cooldown Timer Starts (300s)]
                                                                                      │
                                                                       🚫 All New Scaling Blocked
                                                                                      │
                                                                       🔓 [Timer Hits 0: Scaling Resumes]
```
- **Scale-out**: cooldown starts after the instance finishes launching; new alarms are ignored until it expires, giving the new instance time to absorb load.
- **Scale-in**: a separate cooldown blocks further terminations, protecting against shrinking the cluster too fast.

### Types of Cooldowns
| Type | Description |
|---|---|
| **Default Group Cooldown** | Global catch-all for Simple Scaling Policies/manual changes; AWS default is **300 seconds** |
| **Scaling Policy-Specific Cooldown** | Overrides the group default per policy — e.g., short (60s) scale-out cooldown for aggressive spikes, long (600s) scale-in cooldown to be conservative |
| **Instance Warmup** (Target Tracking / Step Scaling) | Replaces traditional cooldown for modern policies. New alarms can fire immediately, but instances still in warmup are excluded from the aggregate metric math |

### Guardrails to Remember
- **Connection draining interacts with scale-in cooldown**: if ALB deregistration delay = 300s and scale-in cooldown = 300s, a node takes a full **600s** to fully cycle out.
- **Health checks are never delayed by cooldown**: a failed EC2/ALB health check triggers instant termination + replacement regardless of the cooldown clock.

---

## 6. Lifecycle Hooks

Lifecycle Hooks pause an instance in a **Wait state** during launch or termination so you can run custom automation before it goes live or disappears.

### A. Scale-Out Hook — `autoscaling:EC2_INSTANCE_LAUNCHING`
Instance transitions **Pending → Pending:Wait**. It is **not yet registered** with the target group and receives no traffic.
Use it to:
- Pull a specific build / warm Docker images to local disk
- Run configuration management (Ansible playbook, Chef run)
- Run local environment dependency tests before marking ready

### B. Scale-In Hook — `autoscaling:EC2_INSTANCE_TERMINATING`
Instance enters **Terminating:Wait**. It's **immediately deregistered** from the LB and starts connection draining, but compute stays alive.
Use it to:
- Upload logs/stack traces/metrics to S3 before EBS storage vanishes
- Gracefully close WebSocket/streaming/DB connections
- Drain pods/tasks in a container-orchestrated cluster

### End-to-End Orchestration Workflow
```
[ ASG Launch Event ] ──> [ Instance Paused in Pending:Wait ]
                                      │
                                (Triggers event)
                                      ▼
                        [ EventBridge / SNS / SQS ] ──> [ AWS Lambda / Script ]
                                                               │
                                                       (Does custom work)
                                                               │
[ Instance Moves to InService ] <── [ Complete-Lifecycle-Action (CONTINUE) ] ◄┘
```
1. **Pause**: ASG boots the server, enters `Pending:Wait`.
2. **Notification**: ASG fires a payload (Instance ID + `LifecycleActionToken`) to EventBridge, SQS, or SNS.
3. **Execution**: A Lambda (often via SSM Run Command) does the actual work.
4. **Completion signal**: script calls `complete-lifecycle-action` with the token and a result:
   - **CONTINUE** — hook succeeded, instance proceeds to InService/Terminated
   - **ABANDON** — hook failed; on scale-out, the ASG tears down the bad instance and provisions a fresh one

### Timers & Limits
| Setting | Default / Limit |
|---|---|
| Heartbeat Timeout | 1 hour (configurable, seconds to hours) |
| Heartbeat reset | `record-lifecycle-action-heartbeat` resets the countdown for long-running tasks |
| Global hard cap | **48 hours OR 100 × Heartbeat Timeout**, whichever is smaller |

---

## 7. Scaling Policies

| Policy | Behavior | Example |
|---|---|---|
| **Target Tracking** | AWS handles the math to hold a metric at a target value | "Keep average CPU at 60%" |
| **Step Scaling** | Scales by defined CloudWatch alarm thresholds | CPU 50–70% → +1 instance; CPU >70% → +3 instances |
| **Scheduled Scaling** | Predictive scaling for known time-based patterns | Scale to 20 instances Monday 8 AM IST; scale to 2 on Friday evening |
| **Predictive Scaling** | ML model analyzes historical traffic to proactively scale ahead of forecasted spikes | — |

---

## 8. Self-Healing & AZ Rebalancing

### Health Check Options
| Type | Behavior |
|---|---|
| **EC2 (default)** | Checks hardware/hypervisor liveness only — replaces on crash/freeze |
| **ELB (recommended for web apps)** | Monitors actual HTTP response codes; catches a locked-up app returning `502` even if the EC2 host is technically "running" |

### AZ Rebalancing
If subnets span multiple AZs (e.g., AZ-A, AZ-B, AZ-C), the ASG keeps instances evenly distributed. If AZ-A suffers an outage, the ASG spins up replacements in AZ-B/AZ-C to hold desired capacity — then automatically rebalances back across all three once AZ-A recovers.

---

## 9. Termination Policy & Instance Refresh

### Termination Policy Flow (who dies first during scale-in)
1. Identify the **AZ with the most instances** (maintain geographic balance)
2. Evaluate **Allocation Strategy** (e.g., remove older Spot configurations first)
3. Find instances on the **oldest Launch Template version**
4. Pick the instance **closest to its next billing hour**

### Instance Refresh (Rolling Updates)
Updating a Launch Template (e.g., new AMI v1 → v2) does **not** touch already-running instances automatically. **Instance Refresh** rolls the fleet forward safely:
- Terminates a configurable percentage of instances at a time (e.g., 20%)
- Waits for replacement (v2) instances to pass ALB health checks
- Proceeds to the next batch until the whole fleet is upgraded

---

## 10. Suspending ASG Processes

You can pause individual ASG operational loops for troubleshooting:

| Process | Effect when suspended |
|---|---|
| **Launch / Terminate** | Stops the group from changing size |
| **HealthCheck** | Stops the ASG from killing "unhealthy" instances — lets you SSH into a degraded node to inspect logs/stack traces |
| **AZRebalance** | Prevents shifting nodes across zones during localized network issues |

---

## 11. Advanced ELB Topics

### Routing Algorithms
- **Round-Robin** — ALB default
- **Least Outstanding Requests (LOR)** — routes to the target with the fewest currently active/processed requests; prevents slow nodes from getting overwhelmed when request processing time varies

### Cross-Zone Load Balancing
| LB Type | Default | Notes |
|---|---|---|
| **ALB** | Always enabled, **cannot be disabled** | Traffic always spread evenly across all targets in all AZs |
| **NLB** | **Disabled by default** | A node in AZ-A only sees targets in AZ-A; enabling it protects against uneven distribution but incurs cross-AZ data transfer costs |

### Server Name Indication (SNI)
Instead of one load balancer per domain, attach multiple ACM SSL certificates to a single listener. The LB reads the client's SNI handshake and presents the correct certificate for `app1.com`, `app2.com`, `api.internal`, etc.

---

## 12. Gateway Load Balancer (GWLB)

The fourth ELB type (alongside CLB/ALB/NLB), operating at **Layer 3 (Network Layer)**.

- **Concept**: acts as a transparent network gateway + load balancer combined
- **Use case**: scaling third-party virtual appliances — firewalls (Palo Alto, Check Point), IDS/IPS, deep packet inspection tools
- **Mechanics**: catches all IP packets across all ports, wraps them in the **GENEVE protocol (port 6081)**, routes to a firewall instance for inspection, then forwards the clean packet to its destination VPC

---

## 13. Kubernetes Integration — AWS Load Balancer Controller

In EKS-based microservice architectures, engineers rarely configure ELBs manually. Instead, they install the **AWS Load Balancer Controller** inside the cluster:

- A Kubernetes **Ingress** resource → controller provisions a real AWS **ALB**, mapping Ingress path rules to pod endpoints
- A Kubernetes **Service of `Type: LoadBalancer`** → controller provisions an AWS **NLB**, mapping Layer 4 traffic directly to cluster nodes

---

## 14. Quick-Reference Key Points

**Blueprint dependency**: an ASG cannot launch instances without a Launch Template (or legacy Launch Configuration).

**Capacity boundaries**: Min (floor) / Desired (current target) / Max (ceiling) — scaling policies move Desired between Min and Max.

**Subnet distribution**: assign multiple AZ subnets; the ASG naturally balances instances across them for fault tolerance.

**Mixed Instances Policy**: a single ASG can blend On-Demand + Spot instances and multiple instance sizes (e.g., fall back from `t3.medium` to `c6g.large`) for cost savings + Spot-interruption resilience.

**Load Balancer binding**: attached instances auto-register to the target group on passing health checks, and the ASG respects Deregistration Delay (connection draining) when removing them.

**State management guardrail**: ASGs are built for **stateless** apps. If session/data lives only in local server memory (not Redis/RDS/etc.), scale-in **will** lose that data.

---

## Next Steps

- Want to build this yourself? → [`hands-on-labs.md`](./hands-on-labs.md)
- Need a command fast? → [`commands-cheatsheet.md`](./commands-cheatsheet.md)
- Something broke? → [`troubleshooting.md`](./troubleshooting.md)
