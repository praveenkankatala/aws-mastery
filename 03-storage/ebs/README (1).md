# Amazon EBS — End-to-End Concepts

A practical, concept-by-concept walkthrough of Amazon Elastic Block Store: the storage terminology, performance mechanics, underlying architecture, volume types, instance store trade-offs, snapshots, and encryption. This is the conceptual reference — see `commands-cheatsheet.md` for copy-paste commands, `hands-on-labs.md` for guided demos, and `troubleshooting.md` for failure recovery playbooks.

## Table of Contents
1. [EC2 Storage Key Terms](#1-ec2-storage-key-terms)
2. [Storage Performance](#2-storage-performance)
3. [EBS Architecture](#3-ebs-architecture)
4. [Types of EBS Volumes](#4-types-of-ebs-volumes)
5. [Instance Store Volumes](#5-instance-store-volumes)
6. [Choosing: Instance Store vs. EBS](#6-choosing-instance-store-vs-ebs)
7. [EBS Snapshots End-to-End](#7-ebs-snapshots-end-to-end)
8. [Snapshot & Volume Performance Interactions](#8-snapshot--volume-performance-interactions)
9. [EBS Encryption](#9-ebs-encryption)

---

## 1. EC2 Storage Key Terms

EC2 storage splits into two fundamentally different worlds: **network-attached** (EBS — persistent, like a USB drive over a very fast network) and **physically-attached** (Instance Store — temporary, built into the host).

### Root Volume vs. Data Volume
- **Root Volume** — holds the OS boot files/config. By default it's **deleted on termination** (configurable).
- **Data Volume** — additional volumes for app data/DB/logs. Typically persist independently of the instance's own lifecycle.

### EBS core metrics
- **IOPS** — read/write operations per second; critical for databases doing many small transactions.
- **Throughput (MiB/s)** — total data moved per second; critical for large sequential transfers (log processing, big data).
- **EBS-Optimized Instance** — a dedicated network channel just for EBS traffic, so storage I/O doesn't compete with your regular application network traffic.

### EBS volume types at a glance

| Type | Full Name | Characteristic | Best For |
|---|---|---|---|
| `gp3` | General Purpose SSD | Balanced; 3,000 IOPS baseline included | Boot volumes, dev, medium workloads |
| `io2` / `io2 Block Express` | Provisioned IOPS SSD | Highest performance, sub-ms latency, pay for exact IOPS | Critical production DBs (Oracle, SAP HANA) |
| `st1` | Throughput Optimized HDD | Low-cost, throughput-focused | Streaming/big-data/log workloads |
| `sc1` | Cold HDD | Lowest cost, rarely accessed | Archive/backup data |

### Instance Store (ephemeral)
Physical NVMe/SSD drives welded into the host server itself.
- **Ephemeral / non-persistent** — data is lost on stop or terminate; survives only a plain OS reboot.
- **Why use it** — physically attached, so it's extremely fast and low-latency; great for swap space, caching, or distributed DBs that self-replicate (e.g. MongoDB).

### Data protection terms
- **Snapshot** — incremental, point-in-time backup of an EBS volume stored in S3; only changed blocks are saved after the first snapshot.
- **AMI** — a packaged blueprint (root volume config + apps) that can be built from an EBS snapshot to clone identical servers.

**Takeaway:** for a typical web app, use `gp3` for the OS/data root, and reach for an instance type with an Instance Store (look for a `d` in the name) only when you need raw caching speed and can tolerate data loss.

---

## 2. Storage Performance

### The big three metrics
- **IOPS** — how many individual read/write operations per second; matters most for transactional workloads (MySQL, PostgreSQL, NoSQL).
- **Throughput (MiB/s)** — raw data volume moved per second; matters most for sequential workloads (log processing, Hadoop/Spark, backups).
- **Latency (ms)** — time for a single I/O round trip; high latency makes an app feel sluggish even with high IOPS.

### The relationship formula
```
Throughput = IOPS × I/O Block Size
```
Example: 3,000 IOPS at an 8 KB block size = 24 MB/s. The same 3,000 IOPS at a 256 KB block size = 768 MB/s.

### gp3 vs. gp2

| | gp2 (older) | gp3 (current) |
|---|---|---|
| Performance model | Tied to volume size — 3 IOPS/GB | Fully decoupled from size |
| To get 3,000 IOPS | Must provision a 1,000 GB volume | Included free on any size (10 GB or 1 TB) |
| Scaling further | Requires buying more storage | Scale IOPS/throughput independently, up to 16,000 IOPS / 1,000 MiB/s, for a small fee |

### The hidden bottleneck: EBS-optimized instance limits
Every EC2 instance type has a **maximum EBS bandwidth limit**. Provisioning a gp3 volume for 1,000 MiB/s but attaching it to a small instance (e.g. `t3.medium`, capped around 312 MiB/s) means the *instance* — not the volume — becomes the bottleneck, wasting the performance you paid for.

**Practical rules:**
- Default to `gp3` over `gp2` for general workloads.
- Cross-check your **instance's EBS throughput limit** against your **volume's provisioned throughput**.
- For sub-millisecond, extreme-IOPS needs where data loss is tolerable, skip EBS and use **Instance Store**.

---

## 3. EBS Architecture

Think of EBS as a distributed, replicated storage network, not a single physical disk.

### The distributed replication layer (inside the AZ)
- **AZ lock** — an EBS volume lives in exactly one Availability Zone and can only attach to instances in that same AZ.
- **Automatic replication** — within that AZ, AWS silently replicates your volume across multiple physical storage nodes.
- **Blast radius protection** — if one physical drive fails, EBS shifts I/O to a mirror without your instance seeing corruption or an outage — this is why EBS has a very low annual failure rate (0.1%–0.2%).

### The network fabric layer
- **EBS-optimized network fabric** — modern instance types get a dedicated network channel just for EBS traffic, separate from regular application traffic.
- **NVMe protocol** — modern volume types (gp3, io2) use NVMe over this network fabric, making network storage feel nearly as responsive as local storage.

### The control plane: snapshots and S3
- Taking a snapshot scans for changed blocks since the last one and moves just those blocks out of the EBS cluster into **Amazon S3**.
- S3 spans multiple AZs, giving backups far higher durability (99.999999999%) than the live volume itself.

### Multi-Attach (advanced)
Standard EBS is a 1:1 volume-to-instance relationship. **Provisioned IOPS (io2)** volumes support **Multi-Attach**, letting a single volume be attached concurrently to multiple instances in the same AZ — each with full read/write access. Used for clustered applications that manage their own storage concurrency (e.g. Oracle RAC).

---

## 4. Types of EBS Volumes

### SSD family (IOPS-focused)

| Volume | Best For | Architecture Notes |
|---|---|---|
| **gp3** (default) | Boot volumes, dev/test, medium workloads | 3,000 IOPS / 125 MiB/s baseline free; scalable independently of size |
| **gp2** (legacy) | Systems not yet migrated | Performance tied to size (3 IOPS/GB) — avoid for new builds |
| **io2 / io2 Block Express** | Critical production DBs (SAP HANA, Oracle, SQL Server), massive NoSQL | Sub-ms latency, 99.999% durability, up to 256,000 IOPS, supports Multi-Attach |

### HDD family (throughput-focused, cannot boot)

| Volume | Best For | Architecture Notes |
|---|---|---|
| **st1** | Big data (Hadoop/Spark), data warehousing, log processing | Burst-bucket model — accumulates credits, bursts on demand |
| **sc1** | Infrequently accessed archive/backup data | Lowest cost per GB, lowest performance tier |

### Summary comparison

| Type | Designation | Max IOPS | Max Throughput | Primary Metric | Can Boot? |
|---|---|---|---|---|---|
| General Purpose SSD | gp3 | 16,000 | 1,000 MiB/s | IOPS & Throughput | Yes |
| Provisioned IOPS SSD | io2 Block Express | 256,000 | 4,000 MiB/s | IOPS & Latency | Yes |
| Throughput Optimized HDD | st1 | 500 | 500 MiB/s | Throughput | No |
| Cold HDD | sc1 | 250 | 250 MiB/s | Throughput | No |

### Decision rule
- Needs to boot an OS or run an app → **gp3**.
- Critical DB choking on disk queue length → upgrade to **io2**.
- Massive Kafka/log-analytics cluster moving terabytes → **st1**.

---

## 5. Instance Store Volumes

Physical NVMe/SSD disks built directly into the host server — no network hop, so latency is ultra-low.

### Lifecycle rules (the golden rule)

| Action | Effect on Instance Store data |
|---|---|
| OS Reboot | **Saved** — same physical host |
| Stop Instance | **Lost** — hardware released; restart lands on a different host |
| Terminate Instance | **Lost** — disks wiped for the next customer |
| Hardware Failure | **Lost** — no automatic background replication |

### Why use it anyway
It's incredibly fast and effectively free (bundled into the instance's hourly price). Good fits:
- **Caching layers** (Redis/Memcached) — cache can be rebuilt from the source DB after a restart.
- **Temporary swap/scratch space** — rendering, build artifacts.
- **Distributed databases with their own replication** (MongoDB, Cassandra, Elasticsearch) — if one node loses local data, peer nodes replicate it back.

### Identifying instance store support
Look for a `d` in the instance type name:
- `m8g.xlarge` → EBS-only.
- `m8gd.xlarge` → includes a built-in local NVMe Instance Store.

### Practical strategy
- Boot the OS on a **gp3 EBS** root volume (so the server can be stopped/started safely).
- Mount the **Instance Store** to a scratch/cache directory.
- Automate recreation of that directory structure (User Data/Ansible) so the app doesn't break after a stop/start cycle wipes the local disk.

---

## 6. Choosing: Instance Store vs. EBS

The core trade-off: **extreme speed vs. data durability.**

| Requirement | Choose EBS 💾 | Choose Instance Store ⚡ |
|---|---|---|
| Data lifecycle | Must survive stop/restart | Loss on stop/failure is acceptable |
| Primary metric | Durability, snapshots, availability | Ultra-low latency, max IOPS/throughput |
| Location | Network-attached, AZ-replicated | Physically attached to the host |
| Backups | Native point-in-time snapshots to S3 | None native — must be handled at the app layer |
| Flexibility | Resizable, movable between instances | Fixed to the instance size chosen |

### Real-world scenarios
- **Production PostgreSQL DB** → **EBS** (gp3/io2) — cannot risk losing customer records to a hardware fault or resize-related stop.
- **High-throughput Redis cache** → **Instance Store** — sub-ms speed matters more than persistence; cache repopulates from the DB after restart.
- **Distributed NoSQL cluster (Cassandra/MongoDB)** → **Instance Store** — the database's own replication protects against node loss, so raw local speed wins.

### The hybrid strategy (industry standard)
Combine both on the same instance:
- **gp3 EBS** as the root volume (`/dev/sda1`) for the OS — survives restarts, easily snapshotted.
- **Instance Store** (a `d`-family type like `c8gd.xlarge`) mounted at `/mnt/fast-cache` or `/mnt/scratch` for temporary, high-speed processing.

---

## 7. EBS Snapshots End-to-End

### How snapshots work
- **First snapshot** — copies the entire volume's data blocks to S3.
- **Subsequent snapshots** — only copy changed blocks since the last snapshot; unchanged blocks are referenced, not duplicated.
- **Deletion handling** — deleting an older snapshot doesn't break newer ones; AWS re-maps any blocks the newer snapshot still needs.

### The lifecycle, step by step

1. **Creation & crash consistency**
   - Taking a snapshot while an app is actively writing risks a "crash-inconsistent" backup.
   - Best practice: stop the instance or freeze/flush the filesystem first for full data integrity.
2. **Processing (`pending` → `available`)**
   - You can keep using and modifying the live volume immediately after triggering the snapshot — it's captured as a point-in-time image the instant you hit "Create."
3. **Restoration & lazy loading**
   - A volume created from a snapshot is usable almost instantly, but data is pulled from S3 on first read of each block ("lazy loading"), causing a brief latency spike on first touch.
   - **Fast Snapshot Restore (FSR)** eliminates this by pre-hydrating volumes created from an enabled snapshot — ideal for Auto Scaling Groups that need full performance immediately.

### Advanced features
- **Cross-region copy** — snapshots are region-bound; copy explicitly for DR into another region.
- **Encryption inheritance** — a snapshot of an encrypted volume is automatically encrypted with the same KMS key (or a different one, if re-encrypting during a copy).
- **Sharing** — share with specific AWS accounts, or (rarely recommended) make public.

### Enterprise automation: Data Lifecycle Manager (DLM)
Define a tag-based policy instead of manual snapshots:
- **Target**: volumes tagged `Environment: Production`.
- **Schedule**: nightly at 2:00 AM.
- **Retention**: keep the last 14, purge older automatically.
- **Cross-region copy**: replicate to a DR region, retain 7 days there.

**Takeaway:** an AMI is the blueprint for an entire server; an EBS Snapshot is the raw backup of one disk. Use DLM for routine data backups; bake an AMI when you need to launch many identical servers from a golden OS state.

---

## 8. Snapshot & Volume Performance Interactions

### The restoration bottleneck ("lazy loading" / first-touch latency)
A volume restored from a snapshot reports `available` almost instantly, but the data physically lives in S3 until each block is first requested. The first read of any block incurs a latency spike (ms instead of µs); subsequent reads of that block run at full SSD speed. Launching a high-traffic production DB straight from a snapshot can trigger thousands of simultaneous lazy loads and tank performance.

### Two fixes
1. **Pre-warming (manual)** — force a full sequential read before serving traffic:
   ```bash
   sudo dd if=/dev/xvdf of=/dev/null bs=1M
   ```
   Free, but slow — a 1 TB volume can take hours depending on the instance's EBS throughput limit.
2. **Fast Snapshot Restore (FSR)** — enable per snapshot/AZ; AWS pre-allocates infrastructure so restored volumes deliver 100% of provisioned performance immediately, with zero initialization latency. Ideal for Auto Scaling Groups and fast disaster recovery.

### Does taking a snapshot slow down the live instance?
No, or negligibly. Snapshot data transfer to S3 happens over a **dedicated backend management network**, separate from your application's live read/write traffic. The `pending` state simply means the point-in-time block map is locked; you can keep modifying the live volume immediately.

### The performance blueprint
- Dev/staging/simple backups → standard snapshots, accept lazy-loading latency.
- Production CI/CD gold images or ASG targets → enable **FSR**.
- If FSR is too costly → run a `dd` pre-warm script in User Data before marking the node healthy in the load balancer.

---

## 9. EBS Encryption

Zero-overhead, transparent encryption handled at the hypervisor (Nitro) level.

### What gets encrypted
- **Data at rest** — all blocks on the physical disk in the AZ.
- **Data in transit** — traffic between the EC2 host and the EBS storage nodes.
- **All snapshots** taken from an encrypted volume.
- **All volumes created from those encrypted snapshots** (encryption is inherited).
- **Performance impact**: none — encryption runs on dedicated Nitro hardware, not your instance's vCPU.

### Envelope encryption with AWS KMS
1. Launching an encrypted volume triggers a request to **KMS** for a unique data key.
2. KMS returns a **plaintext data key** and an **encrypted (wrapped) data key**.
3. The EC2 host loads the plaintext key into secure hypervisor memory to encrypt/decrypt blocks with **AES-256-XTS**.
4. The encrypted key is stored in the volume's metadata; the plaintext key is wiped from host memory on stop/terminate, and re-derived via KMS on restart.

### AWS Managed Key vs. Customer Managed Key (CMK)

| | AWS Managed Key (`aws/ebs`) | Customer Managed Key (CMK) |
|---|---|---|
| Who manages it | AWS — auto-created, free, auto-rotated | You — define IAM key policies and rotation |
| Cross-account sharing | ❌ No | ✅ Yes |
| Use case | Single-account standard workloads | Multi-account CI/CD, strict compliance auditing |

### Enterprise best practice: encryption by default
Enable **account/region-level default encryption** so every new volume is forced encrypted, even if a script or engineer forgets the flag:
```bash
aws ec2 enable-ebs-encryption-by-default --region ap-south-1
```

### Converting an existing unencrypted volume
**You cannot directly encrypt a live volume in place.** The required workflow:
1. **Snapshot** the unencrypted live volume.
2. **Copy the snapshot**, enabling encryption (and choosing a KMS key) during the copy.
3. **Create a new volume** from the encrypted snapshot copy.
4. **Detach** the old unencrypted volume and **attach** the new encrypted one.

---

## Related Files
- [`commands-cheatsheet.md`](./commands-cheatsheet.md) — every CLI/Linux command from this guide, grouped by topic.
- [`hands-on-labs.md`](./hands-on-labs.md) — step-by-step demos (attach/detach, cross-AZ migration, snapshots, resizing, instance store lifecycle tests, encryption conversion).
- [`troubleshooting.md`](./troubleshooting.md) — lazy-loading latency, modification limits, mount/fstab issues, and more.
