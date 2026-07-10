# Amazon EC2 — End-to-End Concepts

A practical, concept-by-concept walkthrough of Amazon EC2: how it's built, how to size it, how it talks to the network, how to automate it, and how to keep it available. This is the conceptual reference — see `commands-cheatsheet.md` for copy-paste commands, `hands-on-labs.md` for guided demos, and `troubleshooting.md` for failure recovery playbooks.

## Table of Contents
1. [EC2 Architecture](#1-ec2-architecture)
2. [What EC2 Is Suitable For](#2-what-ec2-is-suitable-for)
3. [EC2 Instance Types](#3-ec2-instance-types)
4. [EC2 Network & DNS Architecture](#4-ec2-network--dns-architecture)
5. [Second ENI & MAC-Based Licensing](#5-second-eni--mac-based-licensing)
6. [Instance Status Checks](#6-instance-status-checks)
7. [Instance Metadata Service (IMDS)](#7-instance-metadata-service-imds)
8. [High Availability, Fault Tolerance & Disaster Recovery](#8-high-availability-fault-tolerance--disaster-recovery)
9. [User Data & Bootstrapping](#9-user-data--bootstrapping)
10. [User Data in CloudFormation](#10-user-data-in-cloudformation)
11. [EC2 Placement Groups](#11-ec2-placement-groups)
12. [AMI Lifecycle](#12-ami-lifecycle)

---

## 1. EC2 Architecture

EC2 architecture has to be understood at two levels: the physical layer AWS manages, and the logical/network layer you design.

### The Infrastructure Layer (under the hood)
- **Physical Host** — a physical blade server inside an AWS data center.
- **AWS Nitro System** — a custom hardware/software hypervisor. It offloads networking, storage, and security functions onto dedicated cards, so almost all of the physical server's CPU/RAM goes to your instance (better performance, better isolation).
- **Guest OS** — your Linux/Windows VM runs on top of the Nitro hypervisor.

### The Network Layer (what you actually design)
Traffic flow from the outside world down to your application:

1. **Region & Availability Zone (AZ)** — you deploy into a Region (e.g. `ap-south-1`) and pin the instance to an AZ (a distinct physical data center or set of them).
2. **VPC** — your isolated logical network.
3. **Subnet** — a slice of the VPC's IP range.
   - **Public subnet** — routes to an Internet Gateway (IGW); instances can get public IPs.
   - **Private subnet** — no direct internet route; instances only get private IPs.
4. **Security Group** — the stateful, instance-level firewall attached to the instance's network interface (ENI).
5. **EBS** — the network-attached virtual hard drive holding the OS and data.

**Takeaway:** clicking "Launch Instance" makes AWS find a slot on a Nitro host in your AZ, carve a private IP from your subnet, attach an EBS volume, and wrap it all in a Security Group.

---

## 2. What EC2 Is Suitable For

EC2 gives you raw, resizable, configurable compute — a blank-slate server you control end to end.

### Good fits
- **Traditional web & app hosting** — CMS platforms (WordPress/Drupal), e-commerce stacks (LAMP/MEAN), backend APIs/microservices (Java, .NET, Node.js, Python, Go).
- **HPC & big data** — Hadoop/Spark/Presto, financial/weather/molecular simulation, ML training and inference on GPU instance families.
- **CI/CD & DevOps automation** — self-managed Jenkins/TeamCity/GitLab CI, Ansible control nodes, self-managed Kubernetes control planes.
- **Database & storage control planes** — when you need OS-level or kernel tuning control (Oracle, SQL Server, PostgreSQL with custom plugins) or self-managed NoSQL clusters (Cassandra, MongoDB, Couchbase) where you want explicit control of sharding/replication.

### When EC2 is *not* the best choice
- **Simple containers** — if you just want to run a container without managing OS patching/scaling, use **Fargate** or **ECS/EKS** instead.
- **Event-driven code** — a snippet that runs on an API call or file upload is cheaper and simpler on **Lambda** (serverless) than on a 24/7 EC2 box.

---

## 3. EC2 Instance Types

### Decoding the naming convention
Example: `c8gd.2xlarge`

| Segment | Meaning |
|---|---|
| `c` | Instance family (Compute-optimized) |
| `8` | Generation — higher = newer, usually cheaper per unit of performance |
| `g` | Graviton (ARM) processor — great performance-per-dollar |
| `d` | Includes local NVMe instance store |
| `i` / `a` | Intel / AMD processor |
| `2xlarge` | Size — resources double linearly (xlarge → 2xlarge → 4xlarge...) |

### The 5 core workload families (mnemonic: **M-C-R-I-P**)

| Family | Type | Analogy | Best For | Examples |
|---|---|---|---|---|
| M / T | General Purpose | Family sedan | Balanced compute/memory/network; web servers, small DBs, dev boxes (T = burstable) | `t4g.medium`, `m8g.xlarge` |
| C | Compute Optimized | Sports car | High CPU vs memory; batch processing, video encoding, gaming servers | `c7i.xlarge`, `c8g.2xlarge` |
| R / X | Memory Optimized | Moving truck | Large in-memory datasets, production DB engines, Redis caching | `r7i.xlarge`, `r8g.xlarge` |
| I / D / H | Storage Optimized | Cargo train | High sequential local I/O; NoSQL, data warehousing | `i4i.2xlarge`, `is4gen.xlarge` |
| P / G / Trn | Accelerated Computing | Rocket ship | GPU/Trainium workloads; deep learning, 3D rendering | `g6.xlarge`, `p6.24xlarge` |

### Practical sizing strategy
- Start with **T/M** for dev, test, and generic app layers.
- CPU maxed out, RAM free → move to **C**.
- OOM crashes / memory pressure → move to **R**.

---

## 4. EC2 Network & DNS Architecture

### Network components
- **VPC** — logically isolated network with its own CIDR block (e.g. `10.0.0.0/16`).
- **Subnets** — tied to a specific AZ.
  - Public subnet → route to IGW → public/Elastic IPs.
  - Private subnet → no internet route → outbound internet only via a **NAT Gateway** in a public subnet.
- **ENI (Elastic Network Interface)** — the virtual NIC; holds the primary private IP, MAC address, and Security Group associations. Multiple ENIs can be attached for advanced designs.

### The two-tier firewall model
| Layer | Scope | State |
|---|---|---|
| **Security Groups** | Instance (ENI) | Stateful — allow inbound port 80, the return traffic is auto-allowed |
| **Network ACLs** | Subnet | Stateless — inbound and outbound rules must both be explicitly defined |

### DNS architecture
- **Default resolver**: every VPC gets a built-in DNS resolver at the VPC's base network address **+2** (e.g. VPC `10.0.0.0/16` → resolver at `10.0.0.2`).
- **Public DNS hostname** (`ec2-x-x-x-x.compute-1.amazonaws.com`) — resolves to the **public IP** from outside the VPC, but to the **private IP** when resolved from inside (saves data transfer cost).
- **Private DNS hostname** (`ip-10-0-1-25.ec2.internal`) — resolves only inside the VPC, never reachable from the internet.
- **Route 53 Private Hosted Zones** — create clean internal names (`api.internal.company.local`) mapped to private IPs, instead of relying on the auto-generated hostnames. Traffic never leaves the private AWS network.

**Production blueprint:** keep app/DB servers in private subnets, use Security Groups as the primary least-privilege firewall, and enable `enableDnsHostnames` for internal service discovery via private DNS.

---

## 5. Second ENI & MAC-Based Licensing

### Why attach a second ENI (dual-homing)
- **Network/management segregation** — primary ENI (`eth0`) handles application traffic; secondary ENI (`eth1`) sits in a restricted management subnet for SSH/config-management/logging.
- **Network virtual appliances** — firewalls, proxies, load balancers running on EC2 need one ENI facing external traffic and one facing internal backend traffic.
- **Low-cost HA / hot-swap** — detach an ENI from a failed instance and reattach it to a standby; the standby instantly inherits the same private/public IPs and security groups, avoiding DNS propagation delays.

### MAC address = license (the legacy software problem)
Some enterprise software (SAP, Oracle, legacy CAD/engineering tools) binds its license to the network card's **MAC address**. In the cloud, a stopped/replaced/terminated instance can get a new MAC address, silently invalidating the license.

**The fix:**
1. Create a **standalone ENI** — its MAC address is tied to your AWS account, independent of any instance's lifecycle.
2. Attach it as the **secondary** interface (`eth1`).
3. Register the software license against `eth1`'s MAC address.
4. If the instance dies, detach the ENI and reattach it to the replacement — the license keeps validating because the MAC address followed the ENI, not the instance.

**Gotcha:** Linux won't auto-configure routing for the second interface — you need policy-based routing (`iproute2`/`netplan`) so return traffic on `eth1` actually exits via `eth1`.

---

## 6. Instance Status Checks

AWS runs automated health checks every minute, split into two independent tiers.

| Check | Monitors | Failure Means |
|---|---|---|
| **System Status Check** | Physical host, power, NIC, top-of-rack switch (AWS side) | Underlying hardware/infrastructure problem |
| **Instance Status Check** | OS, kernel network stack, software inside the VM | Your OS/config is broken; AWS hardware is fine |

### Recovery playbooks
**System check failed (hardware fault):**
1. **Stop** the instance — releases the broken physical host.
2. **Start** it — AWS schedules a new, healthy physical host in the same AZ.
   - ⚠️ Instance Store data is lost during this cycle.

**Instance check failed (OS fault):**
1. Pull the **system log** (`Actions → Monitor and troubleshoot → Get system log`) — look for kernel panics, failed mounts, network misconfiguration.
2. Use the **EC2 Serial Console** for raw terminal access even when the network stack is down.
3. **Rescue-volume pattern**: stop the broken instance → detach its root EBS volume → attach it as a secondary disk on a healthy rescue instance in the same AZ → fix the config → reattach as `/dev/sda1` on the original instance.

### Enterprise automation: Auto-Recovery
A CloudWatch alarm on `StatusCheckFailed_System` can trigger EC2 **Auto-Recovery**, which migrates the instance to healthy hardware and reattaches EBS volumes and network identity automatically — no manual intervention.

---

## 7. Instance Metadata Service (IMDS)

IMDS is a built-in, in-instance API the OS can query to learn about itself.

- **Endpoint**: link-local address `169.254.169.254` (IPv4) or `fd00:ec2::254` (IPv6, Nitro only). Link-local means the traffic never leaves the physical host — it isn't routable or interceptable from outside.
- **`meta-data/`** — AWS-generated facts: instance ID, IPs, IAM role credentials, security groups, AZ.
- **`user-data/`** — your own bootstrap script, re-readable by the instance.

### IMDSv1 vs IMDSv2

| | IMDSv1 (legacy) | IMDSv2 (modern standard) |
|---|---|---|
| Model | Direct GET request | Session token required first |
| Risk | Vulnerable to SSRF — any process/proxy can pull IAM credentials | PUT-based token step blocks most SSRF vectors |

IMDSv2 flow:
```bash
# Step 1: get a 6-second session token
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 6")

# Step 2: use the token to query metadata
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

### The container hop-limit gotcha
The IMDSv2 token response has a default **hop limit of 1**. Crossing into a Docker/Kubernetes network bridge adds a hop, so the packet gets dropped by design (prevents a compromised container from stealing host IAM credentials). If a container genuinely needs metadata access, raise the hop limit explicitly (see cheatsheet).

**Production blueprint:** enforce IMDSv2-only, enable `InstanceTags` in metadata options if your automation reads tags, and query metadata dynamically instead of hardcoding config.

---

## 8. High Availability, Fault Tolerance & Disaster Recovery

These are often lumped together as "business continuity" but solve different problems.

### High Availability (HA) — minimize downtime
- Redundancy + automated failover. A load balancer detects a failed node and routes traffic to a healthy one.
- Brief disruption is acceptable (seconds).
- **EC2 pattern**: multiple instances across AZs behind an ALB, managed by an Auto Scaling Group.

### Fault Tolerance (FT) — zero downtime, zero data loss
- Real-time synchronous replication; the secondary processes the same instructions at the same instant.
- Extremely expensive and complex due to bandwidth/latency demands of synchronous writes.
- **EC2 pattern**: Multi-AZ synchronous mirroring on Provisioned IOPS (io2 Block Express), active-active clustering.

### Disaster Recovery (DR) — surviving a regional catastrophe
Measured by:
- **RPO** (Recovery Point Objective) — how much data you can afford to lose.
- **RTO** (Recovery Time Objective) — how fast you must be back online.

| DR Pattern | Description | Recovery Time | Cost |
|---|---|---|---|
| Backup & Restore | Nightly snapshots copied to another region; rebuild from scratch on disaster | Hours | Lowest |
| Pilot Light | Core data replicates 24/7; compute is off/dormant until needed | 10–30 min | Low |
| Warm Standby | Small-scale duplicate runs 24/7; scale up on failover | Minutes | Medium |
| Multi-Region Active-Active | Full production runs in 2+ regions concurrently; Route 53 shifts traffic | Instant | Highest |

### Summary

| | HA | FT | DR |
|---|---|---|---|
| Protects against | Component/AZ failure | Immediate hardware failure | Entire region outage |
| Downtime | Seconds–minutes | Zero | Minutes–hours (by design) |
| Data loss | Minimal | Zero | Per RPO |
| Cost | Standard baseline | Very high (2x+) | Scales with RTO/RPO target |

---

## 9. User Data & Bootstrapping

Bootstrapping = auto-executing configuration on first boot instead of manually SSHing into every server.

### Key rules
- **One-shot by default** — User Data runs once, on first boot. Stopping/starting won't re-trigger it (unless you explicitly re-arm it).
- **Runs as root** — no `sudo` needed inside the script.
- **16 KB size limit**, transmitted Base64-encoded (the console handles encoding).
- **Needs a shebang** — the first line must be something like `#!/bin/bash`.
- **Logs**: check `/var/log/cloud-init-output.log` if a bootstrap script silently fails — the instance will still show 2/2 status checks passed even if your app never started.

### Manual vs. User Data vs. AMI

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Manual config** | SSH in and configure by hand | Fast for one-off prototyping | Unscalable, error-prone, incompatible with ASGs |
| **User Data (bootstrap)** | Base OS + script that installs/configures at boot | Flexible — always pulls latest code/config | Slower boot (minutes) — bad for rapid auto-scaling |
| **AMI (baking)** | Pre-install everything, save as a golden image | Near-instant launch, fully ready | Rigid — re-bake on every change |

### The hybrid golden standard
**Bake the core, bootstrap the rest:**
- Bake heavy, rarely-changing dependencies (Docker engine, monitoring/logging agents, OS patches) into the AMI.
- Use User Data at launch only for fast-changing variables (latest app code, runtime config, cluster registration).

---

## 10. User Data in CloudFormation

CloudFormation templates are YAML/JSON, so a raw shell script needs two intrinsic functions:

- **`Fn::Base64`** — encodes your script as required by the EC2 API.
- **`Fn::Sub`** — substitutes CloudFormation parameters/attributes directly into the script text.

```yaml
Parameters:
  EnvironmentName:
    Type: String
    Default: Production

Resources:
  MyWebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t4g.small
      ImageId: ami-0c55b159cbfafe1f0
      SecurityGroupIds:
        - !Ref WebServerSecurityGroup
      UserData:
        Fn::Base64:
          Fn::Sub: |
            #!/bin/bash
            yum update -y
            yum install -y httpd
            systemctl start httpd
            systemctl enable httpd
            echo "<h1>Welcome to the ${EnvironmentName} Web Server!</h1>" >> /var/www/html/index.html

  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP traffic
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
```

- The YAML literal block `|` preserves line breaks for a proper multi-line script.
- `${EnvironmentName}` is resolved by `Fn::Sub` at deploy time.

### Advanced: `cfn-init` / `cfn-signal`
For large templates that hit the 16 KB limit or need real error handling:
- Define a declarative `AWS::CloudFormation::Init` metadata block (files, packages, services).
- User Data becomes a short 2-line script that just invokes `cfn-init`.
- `cfn-signal` reports success/failure back to CloudFormation — a failure triggers an automatic stack rollback instead of silently reporting a healthy instance.

---

## 11. EC2 Placement Groups

Placement Groups control the *physical* placement of instances on AWS hardware.

| Type | Strategy | Best For | Network | Limits |
|---|---|---|---|---|
| **Cluster** | Pack instances as physically close as possible (same rack) | HPC, tightly-coupled big data, low-latency clusters | Up to 100 Gbps, single-digit µs latency | Single AZ; high blast radius on rack failure |
| **Spread** | Every instance on a separate rack/power source | Small-scale critical nodes (primary/secondary DB, control planes) | Standard | Max 7 running instances per AZ |
| **Partition** | Hardware split into logical partitions; instances spread across partitions | Large distributed systems (HDFS, Cassandra, Kafka) | Standard | Multiple instances per partition; up to 7 partitions per AZ |

### Rules to remember
- **Same instance type** for Cluster groups — mixed sizes can cause `InsufficientCapacity` errors.
- **Launch all at once** — adding instances to an existing Cluster group later may fail if the rack is full.
- **Stop/Start trap** — restarting an instance in a placement group can fail if the rack no longer has room; you may need to remove it from the group first.

---

## 12. AMI Lifecycle

An AMI (Amazon Machine Image) is the master blueprint for an EC2 instance.

### Anatomy of an AMI
- **Root volume snapshot** — the OS and system files.
- **Block Device Mapping (BDM)** — what extra volumes to attach on launch.
- **Launch permissions** — who can use the AMI (private / shared / public).

### The 5 lifecycle phases
1. **Creation ("baking")**
   - From a running instance: `aws ec2 create-image ...` (installs/config already done manually).
   - Automated pipeline: HashiCorp Packer or EC2 Image Builder — declarative, repeatable, CI/CD-friendly.
2. **Storage & state** — AMI moves `pending → available` as EBS snapshots complete. You're billed for the underlying snapshot data, not the AMI entry itself.
3. **Modification & sharing** — private by default; can be shared to specific AWS account IDs or made public.
4. **Cross-region copy** — AMIs are region-bound; explicitly copy to another region for global deployment or DR. AWS assigns a new AMI ID in the destination region.
5. **Retirement**
   - **Deregister** the AMI (blocks new launches from it).
   - **Delete the associated snapshot(s)** — deregistering alone does *not* stop storage billing.
   - **Deprecation** (vs. hard deregistration) — flag an AMI as deprecated with a future date so existing Auto Scaling Groups can keep using it temporarily while new projects are steered away from it.

---

## Related Files
- [`commands-cheatsheet.md`](./commands-cheatsheet.md) — every CLI/Linux command from this guide, grouped by topic.
- [`hands-on-labs.md`](./hands-on-labs.md) — step-by-step demos (AMI baking/decommission, CloudFormation bootstrap, etc.).
- [`troubleshooting.md`](./troubleshooting.md) — status check failures, IMDS/container issues, placement group errors, bootstrap failures.
