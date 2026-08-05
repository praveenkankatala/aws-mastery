# Amazon DynamoDB — The Complete Practical Learning Guide

> A from-zero-to-production guide to Amazon DynamoDB. Written the way you'd actually want it explained: what the thing is, why it behaves the way it does, and exactly which button to press.

![AWS](https://img.shields.io/badge/AWS-DynamoDB-4053D6?style=flat-square&logo=amazon-dynamodb&logoColor=white)
![Level](https://img.shields.io/badge/Level-Beginner%20→%20Advanced-success?style=flat-square)
![Type](https://img.shields.io/badge/Type-Theory%20%2B%20Hands--On-orange?style=flat-square)

---

## 📚 Repository Map

| File | What's inside |
|------|---------------|
| **README.md** *(you are here)* | Concepts, architecture, internals, feature deep-dive, configuration walkthrough, use cases, cost & security |
| **[commands-cheatsheet.md](./commands-cheatsheet.md)** | Every AWS CLI command you'll realistically need, grouped by task |
| **[hands-on-labs.md](./hands-on-labs.md)** | 12 build-it-yourself labs, from first table to a deployed serverless API |
| **[troubleshooting.md](./troubleshooting.md)** | Error messages, root causes, and fixes |

---

## Table of Contents

1. [What Is DynamoDB (And Why Should You Care)](#1-what-is-dynamodb-and-why-should-you-care)
2. [Prerequisites](#2-prerequisites)
3. [The Mental Model: How It Actually Works](#3-the-mental-model-how-it-actually-works)
4. [High-Level Architecture & Service Flow](#4-high-level-architecture--service-flow)
5. [Core Building Blocks](#5-core-building-blocks)
6. [Primary Keys & Partitioning — The Most Important Chapter](#6-primary-keys--partitioning--the-most-important-chapter)
7. [Data Types Reference](#7-data-types-reference)
8. [Capacity Modes, Throughput & Auto Scaling](#8-capacity-modes-throughput--auto-scaling)
9. [Read Consistency Models](#9-read-consistency-models)
10. [Reading Data: GetItem vs Query vs Scan](#10-reading-data-getitem-vs-query-vs-scan)
11. [Writing Data: Puts, Updates, Conditions & Atomicity](#11-writing-data-puts-updates-conditions--atomicity)
12. [Expressions — The DynamoDB Query Language](#12-expressions--the-dynamodb-query-language)
13. [Secondary Indexes: LSI vs GSI](#13-secondary-indexes-lsi-vs-gsi)
14. [Transactions (ACID in a NoSQL World)](#14-transactions-acid-in-a-nosql-world)
15. [DynamoDB Streams & Change Data Capture](#15-dynamodb-streams--change-data-capture)
16. [Time to Live (TTL)](#16-time-to-live-ttl)
17. [DAX — The Microsecond Cache](#17-dax--the-microsecond-cache)
18. [Global Tables & Multi-Region](#18-global-tables--multi-region)
19. [Backup, Restore & Point-in-Time Recovery](#19-backup-restore--point-in-time-recovery)
20. [Import & Export with S3](#20-import--export-with-s3)
21. [Security: IAM, Encryption & Network](#21-security-iam-encryption--network)
22. [Data Modeling & Single-Table Design](#22-data-modeling--single-table-design)
23. [Step-by-Step Configuration & Implementation Guide](#23-step-by-step-configuration--implementation-guide)
24. [Monitoring, Metrics & Observability](#24-monitoring-metrics--observability)
25. [Cost Model & Optimization](#25-cost-model--optimization)
26. [Quotas & Hard Limits](#26-quotas--hard-limits)
27. [How to Use & Where to Use (Target Use Cases)](#27-how-to-use--where-to-use-target-use-cases)
28. [When NOT to Use DynamoDB](#28-when-not-to-use-dynamodb)
29. [DynamoDB vs Other Databases](#29-dynamodb-vs-other-databases)
30. [Best Practices Checklist](#30-best-practices-checklist)
31. [Glossary](#31-glossary)
32. [Learning Path & Resources](#32-learning-path--resources)

---

## 1. What Is DynamoDB (And Why Should You Care)

Amazon DynamoDB is a **fully managed, serverless, key-value and document NoSQL database**. You give AWS a table name and a key schema; AWS gives you back a database that will happily serve single-digit-millisecond responses whether you have 10 items or 10 billion.

### The one-paragraph pitch

Traditional databases scale *up* — you buy a bigger machine until you can't. DynamoDB scales *out* — your data is automatically sliced across many storage nodes, and adding more data or more traffic just means more slices. Because of that design, DynamoDB gives you **predictable latency at any scale**, but it asks something in return: you must know your access patterns *before* you design your table. That trade is the entire story of DynamoDB.

### What "fully managed" actually means here

| You never do this | Because AWS does it |
|---|---|
| Provision servers or storage | Storage grows automatically, no pre-allocation |
| Patch, upgrade, or version | There are no DynamoDB versions or maintenance windows |
| Configure replication | Every write is synchronously stored in 3 Availability Zones |
| Tune indexes for the storage engine | Partitioning and rebalancing are automatic |
| Plan downtime | Schema/capacity changes are online operations |

### Headline characteristics

- **Serverless** — scales to zero on on-demand pricing; you pay per request.
- **Single-digit millisecond latency** at virtually any request rate (microseconds with DAX).
- **Availability SLA of 99.99%** for single-Region tables and **99.999%** for global tables.
- **Durability**: every write is replicated across three AZs before it's acknowledged.
- **ACID transactions** across up to 100 items, within and across tables.
- **Event-driven by default** via DynamoDB Streams → Lambda.
- **Unlimited table size** — a table can grow to petabytes without a redesign.

### The honest trade-offs

DynamoDB is not "a faster SQL database." You give up:

- Ad-hoc queries (no `SELECT * WHERE anything`)
- Joins (you denormalize instead)
- `GROUP BY` / aggregations (you maintain counters or stream to analytics)
- Schema flexibility *after* launch on the key attributes (they're immutable)

If those hurt, read [§28: When NOT to Use DynamoDB](#28-when-not-to-use-dynamodb) before you build anything.

---

## 2. Prerequisites

### Accounts & access

- An **AWS account** (the Free Tier includes 25 GB storage + 25 WCU + 25 RCU per month, always-free).
- An **IAM user or role** — never use the account root user.
- Basic understanding of JSON.

### Tooling

| Tool | Why | Install |
|---|---|---|
| **AWS CLI v2** | Every command in the cheatsheet | [Install guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) |
| **Python 3.9+ & boto3** | SDK labs | `pip install boto3` |
| **Node.js 18+ & AWS SDK v3** | Optional JS labs | `npm i @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb` |
| **NoSQL Workbench** | Visual data modeling, free desktop app | [Download](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/workbench.html) |
| **Docker** | Run DynamoDB Local for offline dev | `docker pull amazon/dynamodb-local` |
| **jq** | Pretty-print CLI JSON output | `apt install jq` / `brew install jq` |

### Verify your setup

```bash
aws --version                 # expect aws-cli/2.x
aws configure                 # set access key, secret, default region, output=json
aws sts get-caller-identity   # confirms who you are
aws dynamodb list-tables      # confirms DynamoDB permissions
```

### Minimum IAM policy to follow this guide

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBLearningAccess",
      "Effect": "Allow",
      "Action": ["dynamodb:*", "dax:*", "application-autoscaling:*"],
      "Resource": "*"
    },
    {
      "Sid": "SupportingServices",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricData", "cloudwatch:PutMetricAlarm",
        "iam:PassRole", "lambda:*", "logs:*", "s3:*"
      ],
      "Resource": "*"
    }
  ]
}
```

> ⚠️ `dynamodb:*` is fine for a sandbox account. Production policies belong in [§21](#21-security-iam-encryption--network).

### Cost expectation for this guide

Following every lab with the recommended settings and cleaning up afterwards costs **well under $1**, and mostly $0 if your account is inside the Free Tier. The only lab that can surprise you is DAX (cluster nodes bill per hour) — delete it when you're done.

---

## 3. The Mental Model: How It Actually Works

Before any commands, internalize this. Everything else in DynamoDB is a consequence of it.

### Step 1 — Your table is not one thing

A DynamoDB table is a **logical name over many physical partitions**. Each partition is roughly 10 GB of storage and can sustain about **3,000 RCU / 1,000 WCU**.

### Step 2 — The partition key decides where an item lives

When you write an item, DynamoDB runs the **partition key value through an internal hash function**. The hash output determines which partition stores the item. This is why the partition key is sometimes called the *hash key*.

```
PutItem { UserId: "U-1042", ... }
             │
             ▼
      hash("U-1042")  ──►  0x7A3F...  ──►  Partition #14
```

### Step 3 — Reads are only fast if you know the key

Because placement is hash-based, DynamoDB can jump straight to the right partition **only if you supply the partition key**. Give it the key → O(1) lookup. Don't give it the key → it must read every partition (a Scan).

```
Know the partition key?  ──YES──►  GetItem / Query   →  fast, cheap, scales
                         ──NO───►  Scan              →  slow, expensive, does not scale
```

### Step 4 — Traffic must spread across partitions

If 90% of your requests hit one partition key value, you've created a **hot partition**. That single partition's ~3,000 RCU ceiling becomes your whole application's ceiling, no matter how much capacity the table has. Adaptive capacity absorbs some of this automatically, but design still matters.

### Step 5 — Therefore: model queries first, data second

In SQL you model entities, then write queries. In DynamoDB you **list every query your application will ever make**, then design a key schema that answers all of them. This inversion is the single biggest adjustment for people coming from relational databases.

---

## 4. High-Level Architecture & Service Flow

### 4.1 Where DynamoDB sits in an application

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            YOUR APPLICATION                              │
│   Mobile App  │  Web SPA  │  Microservice  │  Batch Job  │  IoT Device   │
└───────┬──────────────┬───────────┬──────────────┬──────────────┬─────────┘
        │              │           │              │              │
        └──────────────┴─────┬─────┴──────────────┴──────────────┘
                             │  HTTPS + SigV4 signed request
                             ▼
                ┌────────────────────────────┐
                │   API Gateway / AppSync     │  (optional front door)
                └─────────────┬───────────────┘
                              ▼
                ┌────────────────────────────┐
                │      AWS Lambda / ECS       │  (your business logic)
                └─────────────┬───────────────┘
                              │
                   ┌──────────┴───────────┐
                   ▼                      ▼
        ┌────────────────────┐   ┌──────────────────┐
        │   DAX Cluster      │   │  DynamoDB API    │
        │ (microsecond cache)│──►│    Endpoint      │
        └────────────────────┘   └────────┬─────────┘
                                          │
        ╔═════════════════════════════════▼══════════════════════════════╗
        ║                    DYNAMODB SERVICE (regional)                 ║
        ║  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  ║
        ║  │ Request      │  │ Authentication│  │ Metering & Throttle  │  ║
        ║  │ Router       │─►│ & Authorization│─►│ (RCU/WCU accounting)│  ║
        ║  └──────┬───────┘  └──────────────┘  └──────────────────────┘  ║
        ║         ▼                                                       ║
        ║  ┌────────────────────────────────────────────────────────┐    ║
        ║  │              PARTITION / STORAGE LAYER                  │    ║
        ║  │  P1        P2        P3        P4     ...    Pn         │    ║
        ║  │ ┌────┐    ┌────┐    ┌────┐    ┌────┐        ┌────┐      │    ║
        ║  │ │AZ-a│    │AZ-a│    │AZ-a│    │AZ-a│        │AZ-a│      │    ║
        ║  │ │AZ-b│    │AZ-b│    │AZ-b│    │AZ-b│        │AZ-b│      │    ║
        ║  │ │AZ-c│    │AZ-c│    │AZ-c│    │AZ-c│        │AZ-c│      │    ║
        ║  │ └────┘    └────┘    └────┘    └────┘        └────┘      │    ║
        ║  │  Each partition = 3 synchronous replicas across AZs      │    ║
        ║  └────────────────────────────────────────────────────────┘    ║
        ╚════════════════════╤═══════════════════════╤═══════════════════╝
                             │                       │
                   ┌─────────▼────────┐    ┌─────────▼─────────┐
                   │ DynamoDB Streams │    │  Global Tables    │
                   │ (24h change log) │    │  cross-Region     │
                   └─────────┬────────┘    └───────────────────┘
                             ▼
              Lambda · Kinesis · OpenSearch · Redshift (Zero-ETL)
```

### 4.2 Anatomy of a single write

```
 1. Client                  SDK builds JSON, signs with SigV4
        │
 2. TLS  ─────────────────► dynamodb.<region>.amazonaws.com
        │
 3. Request Router          Validates syntax, resolves table metadata
        │
 4. IAM Authorization       Evaluates identity + resource policies,
        │                   including fine-grained condition keys
 5. Metering                Calculates WCU cost; throttles if over budget
        │
 6. Partition Selection     hash(partition key) → target partition
        │
 7. Leader Replica          Writes to leader in one AZ
        │
 8. Quorum Replication      Waits for durable write on 2 of 3 AZ replicas
        │
 9. HTTP 200 to client      Typically < 10 ms end to end
        │
10. Async fan-out           → Streams record
                            → GSI propagation (eventually consistent)
                            → Global Tables replication (MREC)
```

**Why this matters:** steps 7–8 are why writes are durable but GSIs are only *eventually* consistent. The base table commit is synchronous; index and replica updates happen after the acknowledgment.

### 4.3 Anatomy of a read

```
GetItem (key supplied)
   └─► hash key → one partition → one item        ~1–5 ms   1 RCU (eventually consistent, ≤4KB: 0.5)

Query (partition key + optional sort condition)
   └─► one partition → contiguous range of items  ~1–10 ms  billed on total data read

Scan (no key)
   └─► every partition → filter applied AFTER read  slow    billed on the ENTIRE table read
```

> 🔑 **The most expensive misunderstanding in DynamoDB:** a `FilterExpression` does *not* reduce cost. DynamoDB reads the data, charges you for it, and *then* discards non-matching items.

### 4.4 Feature landscape at a glance

```
                          ┌───────────────────┐
                          │  DynamoDB Table   │
                          └─────────┬─────────┘
        ┌───────────────┬───────────┼───────────┬────────────────┐
        ▼               ▼           ▼           ▼                ▼
  ┌───────────┐  ┌────────────┐ ┌────────┐ ┌──────────┐  ┌──────────────┐
  │  ACCESS   │  │  INDEXES   │ │ CHANGE │ │ RESILIENCE│  │ INTEGRATIONS │
  ├───────────┤  ├────────────┤ ├────────┤ ├──────────┤  ├──────────────┤
  │ GetItem   │  │ LSI        │ │ Streams│ │ PITR     │  │ Lambda       │
  │ Query     │  │ GSI        │ │ Kinesis│ │ Backups  │  │ DAX          │
  │ Scan      │  │ Projections│ │ TTL    │ │ Global   │  │ S3 Import    │
  │ Batch*    │  │            │ │ delete │ │ Tables   │  │ S3 Export    │
  │ Transact* │  │            │ │ events │ │ AWS      │  │ Zero-ETL →   │
  │ PartiQL   │  │            │ │        │ │ Backup   │  │ OpenSearch,  │
  │           │  │            │ │        │ │ MRSC     │  │ Redshift     │
  └───────────┘  └────────────┘ └────────┘ └──────────┘  └──────────────┘
```

---

## 5. Core Building Blocks

### 5.1 The hierarchy

```
AWS Account
 └── Region  (tables are regional resources)
      └── Table            "Orders"
           └── Item        one record  (max 400 KB)
                └── Attribute   name/value pair  ("Status": "SHIPPED")
```

| Concept | Relational equivalent | Notes |
|---|---|---|
| **Table** | Table | No fixed schema beyond the key attributes |
| **Item** | Row | Max **400 KB** including attribute names |
| **Attribute** | Column | Can be scalar, document, or set; can be absent entirely |
| **Primary key** | Primary key | Only mandatory schema element; immutable after creation |
| **Item collection** | — | All items sharing one partition key value |

### 5.2 Schema-less, but not structure-less

DynamoDB enforces only the primary key. Two items in the same table can look completely different:

```json
// Item 1
{ "PK": "USER#42",  "SK": "PROFILE",        "Email": "a@b.com", "Tier": "GOLD" }

// Item 2 — same table, totally different attributes
{ "PK": "USER#42",  "SK": "ORDER#2026-001", "Total": 149.99, "Items": [ ... ] }
```

That flexibility is what makes [single-table design](#22-data-modeling--single-table-design) possible.

### 5.3 Item collections

Every item that shares the same partition key value forms an **item collection**, stored contiguously and sorted by sort key. This is the physical basis of the `Query` operation.

```
Partition key: "USER#42"
┌──────────────────────────────────────────────────────┐
│ SK = PROFILE                                          │  ← Query begins_with("")
│ SK = ADDRESS#HOME                                     │     returns all of these
│ SK = ORDER#2026-001                                   │     in one fast call
│ SK = ORDER#2026-002                                   │
│ SK = ORDER#2026-003                                   │  ← Query begins_with("ORDER#")
└──────────────────────────────────────────────────────┘     returns just the orders
```

> ⚠️ **LSI constraint:** if a table has a Local Secondary Index, a single item collection cannot exceed **10 GB**. Without LSIs, there is no item collection size limit.

---

## 6. Primary Keys & Partitioning — The Most Important Chapter

### 6.1 Two flavours of primary key

**Simple primary key (partition key only)**

```
┌───────────────┬───────────────────────────────────┐
│ UserId (PK)   │  other attributes                 │
├───────────────┼───────────────────────────────────┤
│ U-1042        │  name, email, createdAt ...       │
│ U-1043        │  name, email, createdAt ...       │
└───────────────┴───────────────────────────────────┘
Every partition key value must be unique.
Access: GetItem only. No Query on this table's base key.
```

**Composite primary key (partition key + sort key)**

```
┌──────────────┬────────────────────┬─────────────────────┐
│ UserId (PK)  │ OrderDate (SK)     │  other attributes   │
├──────────────┼────────────────────┼─────────────────────┤
│ U-1042       │ 2026-01-05         │  total, status ...  │
│ U-1042       │ 2026-02-11         │  total, status ...  │
│ U-1042       │ 2026-03-30         │  total, status ...  │
│ U-1043       │ 2026-01-09         │  total, status ...  │
└──────────────┴────────────────────┴─────────────────────┘
PK+SK together must be unique. Items sorted by SK inside each partition.
Access: GetItem (both keys) OR Query (PK + optional SK condition).
```

### 6.2 What the sort key buys you

The sort key unlocks **range queries within a partition**, at essentially no extra cost because the items are already stored in order:

| Condition | Example | Meaning |
|---|---|---|
| `=` | `SK = "2026-01-05"` | Exact match |
| `<`, `<=`, `>`, `>=` | `SK >= "2026-01-01"` | Everything after a point |
| `BETWEEN` | `SK BETWEEN "2026-01" AND "2026-03"` | Inclusive range |
| `begins_with()` | `begins_with(SK, "ORDER#")` | Prefix — the workhorse of single-table design |

> ❗ `begins_with` works on the sort key only. There is **no** `contains` or `ends_with` for key conditions — those are filters, applied after reading.

### 6.3 Choosing a good partition key

| Property | Why | Good | Bad |
|---|---|---|---|
| **High cardinality** | Spreads data widely | `UserId`, `OrderId`, `DeviceId` | `Status`, `Country`, `IsActive` |
| **Uniform access** | No single hot key | `SessionId` | `TenantId` where one tenant is 80% of traffic |
| **Known at read time** | You must supply it | `CustomerId` from JWT | A value you'd have to look up first |
| **Immutable** | Keys can't be updated | Natural IDs | Anything editable, like `Email` |

### 6.4 Hot partitions and how to fix them

**The symptom:** `ProvisionedThroughputExceededException` while CloudWatch shows consumed capacity far below provisioned.

**Adaptive capacity** helps automatically in two ways:
- **Instant adaptive capacity** — reallocates unused table throughput to hot partitions in real time.
- **Split for heat** — DynamoDB physically splits a hot partition into two, spreading the key range.

But adaptive capacity cannot save you from a **single hot key value** — one item, or one partition key, that's overwhelmingly popular. Splitting can't split a single key.

**Remedy 1 — Write sharding (calculated suffix)**

```
Before:  PK = "2026-08-04"           ← all of today's events in ONE partition

After:   PK = "2026-08-04#3"         ← suffix 0..9 chosen at random on write
         Reads: query all 10 shards in parallel, merge results
```

**Remedy 2 — Write sharding (deterministic suffix)**

Compute the suffix from an attribute you'll always know at read time (e.g. `hash(OrderId) % 10`). Then a read needs only one shard, not all ten.

**Remedy 3 — Cache the hot key**

Put DAX in front, or an application-level cache. Reads that never reach DynamoDB can't create a hot partition.

**Remedy 4 — Redesign the key**

Often the real fix. `PK = Status` was never going to work; `PK = OrderId` with a GSI on status will.

### 6.5 The partition math you should know

```
Partitions needed = MAX(
      (Provisioned RCU / 3,000)  +  (Provisioned WCU / 1,000),
      Table size in GB / 10
)
```

You never manage partitions directly — but this formula explains why a table that once held 500 GB and was scaled down still has many partitions, and why each one now gets a small slice of your provisioned throughput. **Partitions are never merged back.**

---

## 7. Data Types Reference

### 7.1 Scalar types

| Type | Wire code | Notes |
|---|---|---|
| **String** | `S` | UTF-8, non-empty for key attributes, up to 400 KB item limit. Sorted by UTF-8 bytes (so `"10" < "9"` — pad your numbers!) |
| **Number** | `N` | Sent as a string in JSON. 38 digits of precision, range 1E-130 to 9.9999E+125. Trailing zeros trimmed |
| **Binary** | `B` | Base64-encoded on the wire |
| **Boolean** | `BOOL` | `true` / `false` |
| **Null** | `NULL` | An explicit "this attribute has no value" — different from the attribute being absent |

### 7.2 Document types

| Type | Code | Notes |
|---|---|---|
| **Map** | `M` | Nested key/value object; up to 32 levels deep |
| **List** | `L` | Ordered array, mixed types allowed |

### 7.3 Set types

| Type | Code | Notes |
|---|---|---|
| **String Set** | `SS` | Unordered, unique values, non-empty |
| **Number Set** | `NS` | Same |
| **Binary Set** | `BS` | Same |

Sets support atomic `ADD` and `DELETE` — great for tags, followers, permissions.

### 7.4 The two JSON dialects

DynamoDB's raw API uses **descriptor JSON**. The document SDKs (boto3 `resource`, JS `lib-dynamodb`) let you use plain JSON.

```jsonc
// Low-level (DynamoDB JSON) — what the CLI and client SDKs use
{
  "OrderId":  { "S": "O-1001" },
  "Total":    { "N": "149.99" },
  "Paid":     { "BOOL": true },
  "Tags":     { "SS": ["urgent", "gift"] },
  "Address":  { "M": { "City": { "S": "Hyderabad" }, "Zip": { "S": "500081" } } },
  "Lines":    { "L": [ { "S": "SKU-1" }, { "S": "SKU-2" } ] }
}

// Native JSON — what document clients use
{
  "OrderId": "O-1001",
  "Total": 149.99,
  "Paid": true,
  "Tags": ["urgent", "gift"],
  "Address": { "City": "Hyderabad", "Zip": "500081" },
  "Lines": ["SKU-1", "SKU-2"]
}
```

CLI shortcut: `--cli-input-json` and `aws dynamodb put-item ... --item file://item.json` need the low-level form. Use `aws dynamodb put-item` with `--cli-binary-format raw-in-base64-out` if you hit binary encoding errors on CLI v2.

### 7.5 Gotchas that bite everyone

- **Empty strings**: allowed for non-key attributes, **not allowed** for partition/sort keys or index keys.
- **Numeric sort keys as strings**: `"2" > "10"` lexicographically. Zero-pad: `"002"`, `"010"`.
- **Timestamps**: use ISO-8601 strings (`2026-08-04T09:15:00Z`) for readability and correct lexicographic sorting, or epoch numbers for arithmetic and TTL. TTL *requires* epoch seconds as a Number.
- **Reserved words**: `Name`, `Status`, `Size`, `Timestamp`, `Year`, `Data`, `Value`, and ~570 others cannot appear literally in expressions. Use `ExpressionAttributeNames` (`#s`) — [see §12](#12-expressions--the-dynamodb-query-language).

---

## 8. Capacity Modes, Throughput & Auto Scaling

### 8.1 The units

| Unit | Buys you |
|---|---|
| **1 WCU** | One write of up to **1 KB** per second |
| **1 RCU** | One **strongly consistent** read of up to **4 KB** per second, **or two eventually consistent** reads of 4 KB |
| **1 WRU / RRU** | Same amounts, but billed per request instead of per second (on-demand mode) |

**Rounding is always up.** A 4.5 KB write costs 5 WCU. A 0.5 KB read costs 1 RCU strongly consistent, 0.5 eventually consistent.

**Transactions cost double:** `TransactWriteItems` = 2× WCU per item, `TransactGetItems` = 2× RCU per item.

### 8.2 Worked capacity examples

```
Scenario A: 100 writes/sec, each item 3.2 KB
  ceil(3.2 / 1) = 4 WCU per write
  100 × 4 = 400 WCU

Scenario B: 200 strongly consistent reads/sec, each item 6 KB
  ceil(6 / 4) = 2 RCU per read
  200 × 2 = 400 RCU

Scenario C: same as B but eventually consistent
  400 / 2 = 200 RCU

Scenario D: 50 transactional writes/sec, each item 2 KB
  ceil(2 / 1) = 2 WCU × 2 (transaction) = 4 WCU per write
  50 × 4 = 200 WCU
```

### 8.3 On-demand vs Provisioned

| | **On-Demand** | **Provisioned** |
|---|---|---|
| Billing | Per request (RRU/WRU) | Per hour for reserved RCU/WCU |
| Scaling | Instant to any previously reached peak; doubles beyond that within ~30 min | Manual, or via Auto Scaling (reactive, minutes) |
| Scales to zero | ✅ Yes | ❌ No — you pay even at 0 traffic |
| Best for | New/unknown/spiky/dev workloads | Steady, forecastable, high-volume production |
| Cost at steady high load | ~5–7× more expensive | Cheapest, especially with Reserved Capacity |
| Throttling | Rare, but possible on sudden >2× spikes | When you exceed provisioned + burst |
| Ceiling control | **Maximum throughput** setting (optional) | Provisioned value is the ceiling |

**Rule of thumb:** start on-demand. Observe for 2–4 weeks. If utilization is consistently above ~40–50% of a stable peak, switch to provisioned with auto scaling.

You can switch modes **once every 24 hours** per table.

### 8.4 Burst capacity

Provisioned tables bank **up to 300 seconds** of unused capacity and can spend it on short spikes. It's a courtesy, not a guarantee — never architect around it.

### 8.5 Auto Scaling (provisioned mode)

Application Auto Scaling watches `ConsumedReadCapacityUnits` / `ConsumedWriteCapacityUnits` and adjusts provisioned capacity toward a target utilization.

```
      Consumed
        ▲
   1000 ┤                     ╭──────╮
        │           ╭─────────╯      ╰────╮
    700 ┤ - - - - - ╱ - - - - - - - - - - -╲ - - - -  target 70%
        │      ╭───╯                        ╰───╮
        └──────┴────────────────────────────────┴──────► time
              scale-up event            scale-down event
```

Settings that matter:

- **Target utilization**: 70% is a sane default. Lower for spiky traffic, higher (80–90%) for very smooth traffic.
- **Min capacity**: your floor; set it high enough to survive a cold start.
- **Max capacity**: your cost guardrail.
- **Scale-in cooldown**: prevents flapping. Scale-out is aggressive; scale-in is deliberately slow.

**Known weakness:** auto scaling is *reactive*. It takes minutes to respond. Flash sales and cron-triggered batch jobs will be throttled before it catches up. Use scheduled scaling or on-demand for those.

### 8.6 Warm throughput & pre-warming

Every table and GSI exposes a **warm throughput** value: the read/write rate the table's current partitions can absorb *immediately*. Before a known spike (product launch, sale, migration cutover), you can raise it proactively.

```bash
aws dynamodb update-table \
  --table-name Orders \
  --warm-throughput ReadUnitsPerSecond=40000,WriteUnitsPerSecond=20000
```

This works on single-Region tables, global tables (version 2019.11.21), and GSIs. Pre-warming takes time proportional to the requested value and table size — do it hours ahead, not minutes.

### 8.7 Maximum throughput for on-demand tables

You can cap an on-demand table so a runaway client or downstream service can't be swamped — and so a bug can't produce a shocking bill:

```bash
aws dynamodb update-table --table-name Orders \
  --on-demand-throughput MaxReadRequestUnits=10000,MaxWriteRequestUnits=5000
```

### 8.8 Table classes

| Class | Storage cost | Throughput cost | Use when |
|---|---|---|---|
| **Standard** | Baseline | Baseline | Normal workloads |
| **Standard-IA** (Infrequent Access) | ~60% cheaper | ~25% more expensive | Storage-dominant tables: logs, archives, old orders — where storage > 50% of the bill |

Switch anytime; it's an online operation, limited to twice per 30-day period.

---

## 9. Read Consistency Models

### 9.1 The two single-Region modes

```
                         WRITE lands on leader replica (AZ-a)
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
                 AZ-a             AZ-b             AZ-c
                 [new]            [new]            [old...]  ← replicating

  Eventually consistent read → may hit AZ-c → returns stale data (usually <1 s)
  Strongly consistent read   → routed to leader → always latest
```

| | Eventually Consistent | Strongly Consistent |
|---|---|---|
| Default? | ✅ Yes | No — set `ConsistentRead=true` |
| Cost | **0.5 RCU** per 4 KB | **1 RCU** per 4 KB |
| Latency | Lowest | Slightly higher |
| Staleness | Typically < 1 second | None |
| Works on GSI? | ✅ | ❌ **Never** |
| Works on LSI? | ✅ | ✅ |
| During AZ issues | Available | May return `500` |

### 9.2 Practical guidance

Use **strongly consistent** reads when a read immediately follows a write in the same logical operation and correctness depends on it — read-after-write in a checkout flow, reading a counter you just incremented, or any "did my update land?" check.

Use **eventually consistent** (the default) for everything else. It's half the price and covers the vast majority of reads: listing products, browsing history, feeds, dashboards.

### 9.3 Multi-Region consistency

For global tables, see [§18](#18-global-tables--multi-region). In short:
- **MREC** (multi-Region eventual consistency) — default, async replication, typically sub-second.
- **MRSC** (multi-Region strong consistency) — synchronous, RPO of zero, exactly three Regions.

---

## 10. Reading Data: GetItem vs Query vs Scan

### 10.1 The decision tree

```
Do you know the full primary key (PK + SK if composite)?
 │
 ├─ YES ────────────────────────────────► GetItem            (1 item, cheapest)
 │        Multiple such keys, ≤100?  ───► BatchGetItem       (parallel, 1 round trip)
 │
 ├─ You know only the partition key ────► Query              (item collection)
 │        …on a different attribute? ───► Query on a GSI/LSI
 │
 └─ NO / you need every item ───────────► Scan               (last resort)
```

### 10.2 GetItem

```bash
aws dynamodb get-item \
  --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-01-05"}}' \
  --consistent-read \
  --projection-expression "OrderId, #s, Total" \
  --expression-attribute-names '{"#s":"Status"}'
```

- Returns exactly one item, or nothing (no error if absent — check for a missing `Item` key).
- `ProjectionExpression` reduces payload but **not** RCU cost — the whole item is still read.
- `ReturnConsumedCapacity TOTAL` tells you exactly what it cost.

### 10.3 Query

```bash
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression "UserId = :u AND OrderDate BETWEEN :start AND :end" \
  --expression-attribute-values '{
      ":u":{"S":"U-1042"},
      ":start":{"S":"2026-01-01"},
      ":end":{"S":"2026-03-31"}}' \
  --scan-index-forward false \
  --limit 25
```

Key facts:
- **Partition key must use `=`.** No ranges, no `IN`, no `OR` on the partition key.
- **Sort key** may use `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `begins_with`.
- `--scan-index-forward false` reverses order — the idiomatic way to get "most recent N".
- Returns at most **1 MB** per call; use `LastEvaluatedKey` to paginate.
- `Limit` caps *items read*, applied **before** `FilterExpression`.

### 10.4 Scan

```bash
aws dynamodb scan --table-name Orders \
  --filter-expression "#s = :st" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":st":{"S":"PENDING"}}'
```

- Reads **every item** in the table (or index), then filters.
- Costs RCU for all data read, not for what's returned. A scan that returns 3 items from a 50 GB table costs the full 50 GB read.
- **Parallel Scan** splits the work: `--total-segments 8 --segment 0..7` run concurrently. Faster, but multiplies capacity consumption — throttle it deliberately.

**Legitimate uses of Scan:** one-off migrations, table exports, small config/lookup tables (< a few MB), analytics jobs you've budgeted for. Everything else is a modeling bug.

### 10.5 Pagination

Both Query and Scan return 1 MB max. The pattern:

```python
import boto3
ddb = boto3.client("dynamodb")
kwargs = {"TableName": "Orders",
          "KeyConditionExpression": "UserId = :u",
          "ExpressionAttributeValues": {":u": {"S": "U-1042"}}}

items, last = [], None
while True:
    if last:
        kwargs["ExclusiveStartKey"] = last
    resp = ddb.query(**kwargs)
    items.extend(resp["Items"])
    last = resp.get("LastEvaluatedKey")
    if not last:
        break
```

> ⚠️ **The empty-page trap:** a Query with a `FilterExpression` can return `Items: []` *and* a `LastEvaluatedKey`. That does **not** mean there are no results — it means this 1 MB page had none. Always loop until `LastEvaluatedKey` is absent.

### 10.6 BatchGetItem

- Up to **100 items** or **16 MB** per call, across multiple tables.
- Requests run in parallel — much faster than 100 serial GetItems.
- Returns `UnprocessedKeys` on partial throttling. **You must retry those with exponential backoff.** The SDKs' `resource`-level batch helpers do this for you; the low-level client does not.

### 10.7 PartiQL — SQL syntax over DynamoDB

```sql
SELECT OrderId, Total FROM "Orders" WHERE UserId = 'U-1042' AND OrderDate > '2026-01-01';
INSERT INTO "Orders" VALUE {'UserId':'U-1044','OrderDate':'2026-08-04','Total':99};
UPDATE "Orders" SET Status='SHIPPED' WHERE UserId='U-1042' AND OrderDate='2026-01-05';
DELETE FROM "Orders" WHERE UserId='U-1042' AND OrderDate='2026-01-05';
```

```bash
aws dynamodb execute-statement \
  --statement "SELECT * FROM \"Orders\" WHERE UserId = 'U-1042'"
```

**PartiQL is syntax, not magic.** If your `WHERE` clause doesn't match a key or index, it becomes a full Scan with the same cost. There are no joins. Use `--index-name` or add the index name to the `FROM` clause (`FROM "Orders"."StatusIndex"`) to target an index. Handy for the console and ad-hoc work; use the native API in application code where the cost is explicit.

---

## 11. Writing Data: Puts, Updates, Conditions & Atomicity

### 11.1 PutItem — full replacement

```bash
aws dynamodb put-item --table-name Orders \
  --item '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"},"Total":{"N":"75.50"}}' \
  --condition-expression "attribute_not_exists(UserId)" \
  --return-values ALL_OLD
```

`PutItem` **overwrites the entire item** if the key exists. Attributes you omit are deleted. Use `attribute_not_exists(<pk>)` to make it an insert-only operation.

### 11.2 UpdateItem — surgical modification

```bash
aws dynamodb update-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}' \
  --update-expression "SET #s = :new, UpdatedAt = :now ADD ViewCount :one REMOVE TempFlag" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":new":{"S":"SHIPPED"},":now":{"S":"2026-08-04T10:00:00Z"},":one":{"N":"1"}}' \
  --return-values ALL_NEW
```

**UpdateItem is an upsert** — if the item doesn't exist, it's created.

The four clauses:

| Clause | Purpose | Example |
|---|---|---|
| `SET` | Assign values, append to lists, arithmetic | `SET Total = Total + :x`, `SET Tags = list_append(Tags, :new)` |
| `REMOVE` | Delete attributes or list elements | `REMOVE Discount, Lines[2]` |
| `ADD` | Atomic number increment or set addition | `ADD Score :points`, `ADD Tags :newTagSet` |
| `DELETE` | Remove members from a set | `DELETE Tags :oldTagSet` |

### 11.3 Atomic counters

```bash
# Increment safely under concurrency — no read needed, no lost updates
--update-expression "ADD PageViews :inc" \
--expression-attribute-values '{":inc":{"N":"1"}}'
```

Atomic counters are **not idempotent** — a client-side retry after a network timeout can double-count. If exact counts matter, use a conditional update with a version number, or a transaction with a client request token.

### 11.4 Conditional writes & optimistic locking

```bash
# Only ship an order that is currently PENDING
--condition-expression "#s = :pending"

# Classic optimistic lock
--condition-expression "Version = :expected" \
--update-expression "SET #d = :data, Version = Version + :one"
```

A failed condition raises `ConditionalCheckFailedException` — and **still costs WCU**. That's a feature: it's how you implement compare-and-swap without a lock table.

Common condition functions: `attribute_exists`, `attribute_not_exists`, `attribute_type`, `begins_with`, `contains`, `size`, plus `=`, `<>`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN`, `AND`, `OR`, `NOT`.

### 11.5 DeleteItem

```bash
aws dynamodb delete-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}' \
  --condition-expression "#s <> :shipped" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":shipped":{"S":"SHIPPED"}}'
```

Deleting a non-existent item succeeds silently and still costs 1 WCU.

### 11.6 BatchWriteItem

- Up to **25 items** or **16 MB** per call.
- Supports `PutRequest` and `DeleteRequest` only — **no updates, no conditions**.
- **Not atomic**: some items can succeed while others fail.
- Returns `UnprocessedItems` — retry with exponential backoff, or you silently lose writes.

```python
import boto3
table = boto3.resource("dynamodb").Table("Orders")
with table.batch_writer() as batch:      # handles batching + retries for you
    for row in rows:
        batch.put_item(Item=row)
```

### 11.7 ReturnValues options

| Value | Returns | Available on |
|---|---|---|
| `NONE` | Nothing (default) | all |
| `ALL_OLD` | Item as it was before | Put, Update, Delete |
| `ALL_NEW` | Item after the change | Update |
| `UPDATED_OLD` | Only changed attributes, before | Update |
| `UPDATED_NEW` | Only changed attributes, after | Update |

Bonus: `ReturnValuesOnConditionCheckFailure=ALL_OLD` returns the item that caused a condition failure — invaluable for debugging concurrency issues without a second read.

---

## 12. Expressions — The DynamoDB Query Language

There are five expression types. Knowing which is which prevents most beginner errors.

| Expression | Used in | Purpose | Affects cost? |
|---|---|---|---|
| **KeyConditionExpression** | Query | Select the item range to read | ✅ Reduces cost |
| **FilterExpression** | Query, Scan | Discard items after reading | ❌ No saving |
| **ProjectionExpression** | all reads | Trim returned attributes | ❌ No saving (smaller payload only) |
| **ConditionExpression** | writes | Allow/reject the write | — |
| **UpdateExpression** | UpdateItem | Describe the mutation | — |

### 12.1 Placeholders — why `#` and `:`

```
#name   → ExpressionAttributeNames   → escapes reserved words & special characters
:value  → ExpressionAttributeValues  → supplies data, prevents injection
```

```bash
--filter-expression "#s = :st AND #sz > :n"
--expression-attribute-names  '{"#s":"Status","#sz":"Size"}'
--expression-attribute-values '{":st":{"S":"ACTIVE"},":n":{"N":"100"}}'
```

`Status` and `Size` are both reserved words. Without the `#` aliases you get `ValidationException: Invalid FilterExpression: Attribute name is a reserved keyword`.

**Practical habit:** alias *every* attribute name. It costs nothing and you'll never look up the reserved-word list again.

### 12.2 Nested attribute paths

```
--projection-expression "Address.City, Lines[0].SKU, Metadata.#src"
```

Dots for map members, `[n]` for list indexes. Each path segment that is a reserved word needs its own `#` alias.

### 12.3 Useful functions

| Function | Works in | Meaning |
|---|---|---|
| `attribute_exists(path)` | Condition, Filter | Attribute present |
| `attribute_not_exists(path)` | Condition, Filter | Attribute absent — the "insert only" idiom |
| `attribute_type(path, type)` | Condition, Filter | Type check (`S`, `N`, `SS`, `M`, ...) |
| `begins_with(path, substr)` | Key, Condition, Filter | Prefix match |
| `contains(path, operand)` | Condition, Filter | Substring, or set/list membership |
| `size(path)` | Condition, Filter | Length of string/binary/set/list/map |
| `list_append(l1, l2)` | Update | Concatenate lists |
| `if_not_exists(path, default)` | Update | Set only if absent — great for `CreatedAt` |

```bash
# Set CreatedAt only on first write, always bump UpdatedAt
--update-expression "SET CreatedAt = if_not_exists(CreatedAt, :now), UpdatedAt = :now"
```

---

## 13. Secondary Indexes: LSI vs GSI

An index is a **materialized, automatically maintained alternate view** of your table with a different key schema. Indexes are how you answer queries the base key can't.

### 13.1 Side-by-side

| | **Local Secondary Index (LSI)** | **Global Secondary Index (GSI)** |
|---|---|---|
| Partition key | **Same as base table** | **Any attribute** |
| Sort key | Different attribute | Any attribute (optional) |
| Create after table exists? | ❌ **Only at table creation** | ✅ Anytime, online |
| Delete later? | ❌ Never | ✅ Anytime |
| Max per table | 5 | 20 (soft limit, raisable) |
| Capacity | **Shares the base table's** | **Has its own** RCU/WCU |
| Strongly consistent reads | ✅ Supported | ❌ Never |
| Item collection limit | **10 GB per partition key** | No limit |
| Key uniqueness required | Yes (with base PK) | No — duplicates allowed |

### 13.2 Visual

```
BASE TABLE:  PK = UserId, SK = OrderDate
┌──────────┬────────────┬──────────┬────────┐
│ UserId   │ OrderDate  │ Status   │ Total  │
├──────────┼────────────┼──────────┼────────┤
│ U-1042   │ 2026-01-05 │ SHIPPED  │ 149.99 │
│ U-1042   │ 2026-02-11 │ PENDING  │  75.00 │
│ U-1043   │ 2026-01-09 │ PENDING  │ 220.00 │
└──────────┴────────────┴──────────┴────────┘

LSI "TotalIndex":  PK = UserId (same), SK = Total
  → "Show me U-1042's orders sorted by value"

GSI "StatusIndex": PK = Status, SK = OrderDate
┌──────────┬────────────┬──────────┐
│ Status   │ OrderDate  │ UserId   │   ← different partitioning entirely
├──────────┼────────────┼──────────┤
│ PENDING  │ 2026-01-09 │ U-1043   │
│ PENDING  │ 2026-02-11 │ U-1042   │
│ SHIPPED  │ 2026-01-05 │ U-1042   │
└──────────┴────────────┴──────────┘
  → "Show me all PENDING orders across all users, oldest first"
```

### 13.3 Projections — what gets copied into the index

| Type | Copies | Trade-off |
|---|---|---|
| `KEYS_ONLY` | Index keys + base table keys | Smallest & cheapest; forces a second read for other attributes |
| `INCLUDE` | Keys + a named list | The usual sweet spot |
| `ALL` | Every attribute | Fastest reads, but doubles storage and write cost |

> 💡 **A GSI read that isn't covered by the projection does not "fall back" to the table.** It returns nothing for that attribute. Your code must then GetItem the base table — extra latency and cost. Project deliberately.

### 13.4 The GSI write amplification you must budget for

```
Base table write (1 KB item)          =  1 WCU
+ GSI-1 (ALL projection, item in it)  =  1 WCU
+ GSI-2 (ALL projection, item in it)  =  1 WCU
────────────────────────────────────────────
Total                                  =  3 WCU  ← 3× the cost of the same write
```

A GSI update where the indexed key value changes costs even more: DynamoDB performs a **delete + insert** in the index (2 writes).

### 13.5 Sparse indexes — the best trick in DynamoDB

An item appears in a GSI **only if it has both index key attributes**. Exploit this:

```
Add attribute "NeedsReview" ONLY to items requiring review.
Create GSI: PK = NeedsReview.
The index now contains 200 items out of 50 million.
Querying the review queue is instant and nearly free.
When the item is reviewed → REMOVE NeedsReview → it vanishes from the index.
```

This is how you build queues, flags, and "unprocessed" views without scanning.

### 13.6 GSI overloading

In single-table design, index key attributes are given generic names (`GSI1PK`, `GSI1SK`) so different entity types can populate them with different meanings, all served by one index. See [§22](#22-data-modeling--single-table-design).

### 13.7 GSI throttling — the silent killer

If a GSI is under-provisioned, **writes to the base table get throttled**, even though the base table has plenty of capacity. DynamoDB will not let the index fall arbitrarily far behind.

**Always** monitor GSI capacity separately, and give GSIs headroom.

### 13.8 Backfilling

Creating a GSI on an existing table triggers a backfill. The table stays fully available; the index reports `CREATING` then `ACTIVE`. On a large table this can take hours and consumes write capacity. Check progress:

```bash
aws dynamodb describe-table --table-name Orders \
  --query 'Table.GlobalSecondaryIndexes[].{Name:IndexName,Status:IndexStatus,Backfilling:Backfilling}'
```

---

## 14. Transactions (ACID in a NoSQL World)

### 14.1 What you get

DynamoDB supports **serializable, all-or-nothing transactions** across up to **100 items**, spanning multiple tables in the same account and Region.

| Operation | Purpose |
|---|---|
| `TransactWriteItems` | Up to 100 `Put` / `Update` / `Delete` / `ConditionCheck` actions, atomically |
| `TransactGetItems` | Up to 100 consistent reads as a single snapshot |

`ConditionCheck` is special: it asserts something about an item **without modifying it**. Perfect for "only allow this order if the user account is active."

### 14.2 Example: transfer with a uniqueness guarantee

```json
{
  "TransactItems": [
    { "Update": {
        "TableName": "Accounts",
        "Key": {"AccountId": {"S": "A-1"}},
        "UpdateExpression": "SET Balance = Balance - :amt",
        "ConditionExpression": "Balance >= :amt",
        "ExpressionAttributeValues": {":amt": {"N": "100"}}
    }},
    { "Update": {
        "TableName": "Accounts",
        "Key": {"AccountId": {"S": "A-2"}},
        "UpdateExpression": "SET Balance = Balance + :amt",
        "ExpressionAttributeValues": {":amt": {"N": "100"}}
    }},
    { "Put": {
        "TableName": "Ledger",
        "Item": {"TxnId": {"S": "T-9001"}, "Amount": {"N": "100"}},
        "ConditionExpression": "attribute_not_exists(TxnId)"
    }}
  ],
  "ClientRequestToken": "idempotency-key-abc-123"
}
```

If *any* condition fails, **nothing** is written.

### 14.3 Rules and costs

- **2× capacity**: each item in a transaction costs double the normal RCU/WCU.
- **No duplicate items**: the same item cannot appear twice in one transaction.
- **Same Region, same account** for all tables involved.
- **`ClientRequestToken`** makes the transaction idempotent for **10 minutes** — essential for safe retries.
- Transactions are **not supported** by tables in the older Global Tables version 2017.11.29.

### 14.4 TransactionCanceledException

The one error you'll actually see. Read the `CancellationReasons` array — it has one entry per action, in order:

| Reason code | Meaning | Fix |
|---|---|---|
| `ConditionalCheckFailed` | Your business condition wasn't met | Expected; handle in app logic |
| `TransactionConflict` | Another transaction touched the same item | Retry with backoff |
| `ThrottlingError` / `ProvisionedThroughputExceeded` | Not enough capacity | Add capacity, back off |
| `ItemCollectionSizeLimitExceeded` | LSI item collection > 10 GB | Redesign |
| `ValidationError` | Malformed request | Fix the code |
| `None` | This particular action was fine | — |

### 14.5 When to use them (and when not)

**Use transactions for:** money movement, inventory decrement + order creation, enforcing uniqueness on a second attribute (the "unique email" pattern), state machines that must not tear.

**Don't use transactions for:** bulk loading (use `BatchWriteItem`), anything a single conditional `UpdateItem` can already do atomically, or high-throughput hot paths where 2× cost and conflict retries hurt.

---

## 15. DynamoDB Streams & Change Data Capture

### 15.1 What a stream is

An **ordered, immutable log of every item-level change** in a table, retained for **24 hours**. Turning it on makes your database event-driven.

```
  Table write ──► Stream record ──► Shard ──► Consumer
                                              ├─ Lambda (most common)
                                              ├─ Kinesis Client Library app
                                              └─ Managed integrations
```

### 15.2 StreamViewType — choose carefully

| Value | Record contains | Use for |
|---|---|---|
| `KEYS_ONLY` | Just the key | Cache invalidation |
| `NEW_IMAGE` | Item after the change | Search index sync, replication |
| `OLD_IMAGE` | Item before the change | Audit trail, undo, capturing TTL deletions |
| `NEW_AND_OLD_IMAGES` | Both | Diffing, change auditing, most flexible |

You **cannot change** the view type without disabling and re-enabling the stream (which resets the log).

### 15.3 Ordering guarantees

- Records for the **same partition key** are delivered in **strict order**. ✅
- Records **across different partition keys** have **no ordering guarantee**. ⚠️
- **Exactly-once delivery to the stream**, but **at-least-once delivery to consumers** — your Lambda **must be idempotent**.

### 15.4 Lambda trigger configuration

```bash
aws lambda create-event-source-mapping \
  --function-name ProcessOrderChanges \
  --event-source-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders/stream/2026-08-04T00:00:00.000 \
  --starting-position LATEST \
  --batch-size 100 \
  --maximum-batching-window-in-seconds 5 \
  --maximum-retry-attempts 3 \
  --bisect-batch-on-function-error \
  --maximum-record-age-in-seconds 3600 \
  --destination-config '{"OnFailure":{"Destination":"arn:aws:sqs:us-east-1:123456789012:stream-dlq"}}'
```

Settings that save you at 3 a.m.:

| Setting | Why |
|---|---|
| `--bisect-batch-on-function-error` | Isolates the one poison record instead of failing the whole batch forever |
| `--maximum-retry-attempts` | Without a cap, a bad record blocks the shard until it expires (24 h) |
| `--destination-config` (DLQ) | Failed batches land somewhere you can inspect |
| `--parallelization-factor` | Up to 10 concurrent Lambdas per shard, still ordered per key |
| `--filter-criteria` | Drop irrelevant events before they invoke Lambda — saves real money |

**Event filtering example** — only react to items becoming `SHIPPED`:

```json
{"Filters":[{"Pattern":"{\"dynamodb\":{\"NewImage\":{\"Status\":{\"S\":[\"SHIPPED\"]}}}}"}]}
```

### 15.5 A sample stream record

```json
{
  "eventID": "1b2c3d",
  "eventName": "MODIFY",
  "eventSource": "aws:dynamodb",
  "awsRegion": "us-east-1",
  "dynamodb": {
    "ApproximateCreationDateTime": 1785835200,
    "Keys":      { "UserId": {"S":"U-1042"}, "OrderDate": {"S":"2026-08-04"} },
    "OldImage":  { "Status": {"S":"PENDING"} },
    "NewImage":  { "Status": {"S":"SHIPPED"} },
    "SequenceNumber": "4421300000000012345",
    "SizeBytes": 112,
    "StreamViewType": "NEW_AND_OLD_IMAGES"
  },
  "userIdentity": {
    "type": "Service",
    "principalId": "dynamodb.amazonaws.com"
  }
}
```

> 💡 That `userIdentity` block with `principalId: dynamodb.amazonaws.com` is how you tell a **TTL deletion** apart from an application delete. Both arrive as `eventName: REMOVE`.

### 15.6 Kinesis Data Streams for DynamoDB

An alternative CDC target with different characteristics:

| | DynamoDB Streams | Kinesis Data Streams |
|---|---|---|
| Retention | 24 hours | Up to 365 days |
| Consumers | Limited (2 recommended per shard) | Many, via enhanced fan-out |
| Ordering | Strict per key | Best-effort; may contain duplicates |
| Cost | Free reads (Lambda polling) | Priced per shard/payload |
| Best for | Triggers, small pipelines | Analytics fan-out, long retention, multiple teams |

Both can be enabled on the same table simultaneously.

### 15.7 Zero-ETL integrations

DynamoDB can now feed analytics targets without you writing pipeline code:

- **→ Amazon OpenSearch Service** — full-text search over your items.
- **→ Amazon Redshift** — analytics and joins against warehouse data.
- **→ SageMaker Lakehouse / S3 Tables** — data-lake landing.

These handle the initial full load plus continuous change replication.

---

## 16. Time to Live (TTL)

### 16.1 What it does

TTL automatically deletes expired items **at no cost in write capacity**. You nominate one attribute holding a **Unix epoch timestamp in seconds** (Number type).

```bash
aws dynamodb update-time-to-live --table-name Sessions \
  --time-to-live-specification "Enabled=true,AttributeName=ExpiresAt"
```

```json
{ "SessionId": "S-901", "UserId": "U-1042", "ExpiresAt": 1785835200 }
```

### 16.2 The rules people get wrong

| Rule | Detail |
|---|---|
| **Seconds, not milliseconds** | `1785835200` ✅ · `1785835200000` ❌ (that's year 58,547 — it will never delete) |
| **Number type required** | A string `"1785835200"` is silently ignored |
| **Not instantaneous** | Deletion typically happens within 48 hours of expiry, sometimes longer on large tables |
| **Expired items still appear in reads** | You **must** filter them out in your app: `FilterExpression: "ExpiresAt > :now"` |
| **Deletions go to Streams** | With `userIdentity.principalId = "dynamodb.amazonaws.com"` |
| **Indexes are cleaned too** | GSI/LSI entries are removed along with the item |
| **One attribute per table** | You can't have two TTL attributes |
| **Missing attribute = never expires** | Items without the TTL attribute are untouched |

### 16.3 Classic patterns

- **Session store** — `ExpiresAt = now + 3600`.
- **Rate limiter** — counters that self-clean each minute.
- **Regulatory retention** — auto-purge PII after N days.
- **Archive-then-delete** — TTL deletion → Stream → Lambda → write to S3 Glacier.
- **Idempotency keys** — dedupe records that expire after the retry window.

### 16.4 Handy epoch calculations

```bash
# 30 days from now
date -d "+30 days" +%s          # Linux
date -v+30d +%s                 # macOS
python3 -c "import time; print(int(time.time()) + 30*86400)"
```

---

## 17. DAX — The Microsecond Cache

### 17.1 What it is

**Amazon DynamoDB Accelerator (DAX)** is a fully managed, in-memory, write-through cache that speaks the DynamoDB API. Point your SDK at the DAX endpoint instead of DynamoDB and cached reads drop from **milliseconds to microseconds**.

```
App ──► DAX Cluster ──(cache miss)──► DynamoDB
         │  ├─ Item cache  (GetItem/BatchGetItem results,  default TTL 5 min)
         │  └─ Query cache (Query/Scan result sets,        default TTL 5 min)
         └──(cache hit)──► microsecond response
```

### 17.2 Key facts

- **API-compatible** — usually a one-line client change, no query rewrite.
- **Write-through** — writes go to DynamoDB first, then update the item cache. The **query cache is not updated by writes**; it only expires by TTL.
- **Runs inside your VPC** — requires subnet group, security group, and an IAM role that lets DAX call DynamoDB.
- **Cluster of 1–11 nodes**; 3+ nodes across AZs for production, 1 node for dev (no HA).
- **Eventually consistent reads only.** A `ConsistentRead=true` request bypasses the cache entirely and goes straight to DynamoDB.
- **Billed per node-hour** — this is the one DynamoDB feature that costs money while idle.

### 17.3 When DAX is right

✅ Read-heavy, read-mostly workloads with repeated hot keys — product catalogs, game leaderboards, config lookups, sessions.
✅ Applications that need microsecond latency.
✅ Existing hot-partition read pressure you want to absorb.

❌ Write-heavy workloads (adds a hop, gains nothing).
❌ Applications needing strongly consistent reads.
❌ Low-traffic apps — the cluster cost exceeds the DynamoDB savings.

> 💡 **Alternative:** for serverless architectures where a VPC is unwelcome, consider ElastiCache Serverless or a simple in-process cache. DAX's advantage is transparency; its cost is the VPC and the always-on nodes.

---

## 18. Global Tables & Multi-Region

### 18.1 The concept

Global Tables give you a **multi-active, multi-Region** table: your application can read *and write* in every Region, and DynamoDB handles replication.

```
   us-east-1                    eu-west-1                  ap-south-1
  ┌───────────┐  ◄──────────►  ┌───────────┐  ◄─────────► ┌───────────┐
  │ Orders    │                │ Orders    │              │ Orders    │
  │ (replica) │  ◄──────────►  │ (replica) │  ◄─────────► │ (replica) │
  └───────────┘                └───────────┘              └───────────┘
     writes                       writes                     writes
```

### 18.2 Two consistency modes

| | **MREC** (default) | **MRSC** |
|---|---|---|
| Replication | Asynchronous, typically < 1 second | Synchronous to at least one other Region before ack |
| Conflict resolution | **Last-writer-wins** by timestamp | No conflicts — strongly consistent |
| Regions | Any number, any Region | **Exactly three** (or two replicas + one witness) |
| RPO | Near-zero, not zero | **Zero** |
| Write latency | Local-Region latency | Higher — cross-Region round trip |
| Configurable later? | Set at creation, **cannot be changed** | Set at creation, **cannot be changed** |
| Use for | Global read scaling, DR, latency reduction | Inventory, financial ledgers, user profiles where stale reads are unacceptable |

MRSC is available in a defined subset of Regions (US East N. Virginia/Ohio, US West Oregon, Europe Ireland/London/Paris/Frankfurt, Asia Pacific Tokyo/Seoul/Osaka at time of writing) — verify current availability before designing around it.

> 🧪 You can now use **AWS Fault Injection Service** to pause regional replication on an MRSC global table and rehearse a Region failure. Do this before you need it.

### 18.3 Requirements & caveats

- **DynamoDB Streams must be enabled** with `NEW_AND_OLD_IMAGES`.
- Table name and key schema must be **identical** in every Region.
- Replicas must be empty when added to an existing table (or use S3 export/import).
- **Replicated writes consume rWCU** in every replica Region — a 3-Region global table costs roughly 3× write capacity.
- **Last-writer-wins in MREC means silent data loss** if two Regions write the same item concurrently. Design around it: Region-scoped partition keys, or use MRSC.
- **TTL deletions replicate**, but the deletion is evaluated per Region.

### 18.4 Setting one up

```bash
# 1. Enable streams with the required view type
aws dynamodb update-table --table-name Orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

# 2. Add a replica (MREC)
aws dynamodb update-table --table-name Orders \
  --replica-updates '[{"Create":{"RegionName":"eu-west-1"}}]'

# 3. Verify
aws dynamodb describe-table --table-name Orders \
  --query 'Table.Replicas[].{Region:RegionName,Status:ReplicaStatus}'
```

Monitor `ReplicationLatency` and `PendingReplicationCount` in CloudWatch.

---

## 19. Backup, Restore & Point-in-Time Recovery

### 19.1 Three mechanisms

| Mechanism | Granularity | Retention | Impact |
|---|---|---|---|
| **On-demand backup** | Snapshot at the moment you take it | Until you delete it | Zero performance impact, instant |
| **Point-in-Time Recovery (PITR)** | Any second in the window | **Configurable 1–35 days** | Continuous, no capacity consumed |
| **AWS Backup** | Scheduled snapshots + lifecycle to cold storage | Per your backup plan | Centralized governance, cross-account/Region copy |

### 19.2 PITR

```bash
aws dynamodb update-continuous-backups --table-name Orders \
  --point-in-time-recovery-specification \
    "PointInTimeRecoveryEnabled=true,RecoveryPeriodInDays=14"

# What window is available right now?
aws dynamodb describe-continuous-backups --table-name Orders

# Restore to a moment before the incident
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Orders \
  --target-table-name Orders-Restored-20260804 \
  --restore-date-time 2026-08-04T09:15:00Z
```

**Enable PITR on every production table.** It's cheap insurance and the single best defence against an accidental bulk delete.

### 19.3 Restore behaviour — read this before an incident

Restores **always create a new table**. You cannot restore over an existing one. Plan a cutover.

What is **not** carried over automatically:

- ❌ Auto Scaling policies
- ❌ TTL settings
- ❌ Stream settings
- ❌ CloudWatch alarms
- ❌ IAM resource policies and tags (tags optional via flag)
- ✅ Local Secondary Indexes (always restored)
- ⚙️ Global Secondary Indexes — you can choose to include or skip them (skipping is much faster)

Restore duration scales with table size — plan for **hours** on large tables and rehearse it.

### 19.4 Deletion protection

The cheapest safety net in AWS:

```bash
aws dynamodb update-table --table-name Orders --deletion-protection-enabled
```

Now `DeleteTable` fails until someone explicitly disables it. Turn this on for every production table, and enforce it with an SCP if you can.

---

## 20. Import & Export with S3

### 20.1 Export to S3

A **full-table snapshot to S3 that consumes zero read capacity** — because it's served from the continuous backup, not the live table.

```bash
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --s3-bucket my-exports \
  --s3-prefix orders/2026-08-04 \
  --export-format DYNAMODB_JSON \
  --export-time 2026-08-04T00:00:00Z
```

- Requires **PITR to be enabled**.
- Formats: `DYNAMODB_JSON` or `ION`.
- **Incremental export** is supported — export only the changes between two points in time, which is dramatically cheaper for ongoing pipelines.
- Query the result directly with **Athena**, load into **Redshift/Glue/EMR**, or use it to seed another table.

### 20.2 Import from S3

Create a **brand new table** directly from S3 data — no write capacity consumed, no loader script.

```bash
aws dynamodb import-table \
  --s3-bucket-source S3Bucket=my-imports,S3KeyPrefix=orders/ \
  --input-format CSV \
  --input-compression-type GZIP \
  --table-creation-parameters '{
      "TableName":"OrdersImported",
      "AttributeDefinitions":[
        {"AttributeName":"UserId","AttributeType":"S"},
        {"AttributeName":"OrderDate","AttributeType":"S"}],
      "KeySchema":[
        {"AttributeName":"UserId","KeyType":"HASH"},
        {"AttributeName":"OrderDate","KeyType":"RANGE"}],
      "BillingMode":"PAY_PER_REQUEST"}'
```

- Input formats: `CSV`, `DYNAMODB_JSON`, `ION`.
- **Target table must not already exist** — import always creates it.
- Malformed rows are logged to CloudWatch Logs and skipped; check `import-table-description` for the error count.
- Enormously cheaper than `BatchWriteItem` for bulk loading — often the difference between $400 and $4.

---

## 21. Security: IAM, Encryption & Network

### 21.1 Encryption at rest — always on

Every table is encrypted. You pick the key:

| Option | Key | Cost | Control |
|---|---|---|---|
| **AWS owned key** (default) | AWS-managed, invisible | Free | None |
| **AWS managed key** (`aws/dynamodb`) | In your account, visible in KMS | KMS charges | Auditable via CloudTrail |
| **Customer managed key (CMK)** | Yours | KMS charges | Full — rotation, key policies, revoke access |

Choose a CMK when compliance requires you to prove key control or be able to cryptographically disable access.

### 21.2 Encryption in transit

All API traffic is HTTPS/TLS. Enforce it in policy:

```json
{ "Effect": "Deny", "Action": "dynamodb:*", "Resource": "*",
  "Condition": { "Bool": { "aws:SecureTransport": "false" } } }
```

### 21.3 Least-privilege IAM

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AppReadWriteScopedToTableAndIndexes",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem", "dynamodb:BatchGetItem", "dynamodb:Query",
        "dynamodb:PutItem", "dynamodb:UpdateItem", "dynamodb:DeleteItem",
        "dynamodb:BatchWriteItem", "dynamodb:ConditionCheckItem"
      ],
      "Resource": [
        "arn:aws:dynamodb:us-east-1:123456789012:table/Orders",
        "arn:aws:dynamodb:us-east-1:123456789012:table/Orders/index/*"
      ]
    },
    {
      "Sid": "NoScansInProduction",
      "Effect": "Deny",
      "Action": "dynamodb:Scan",
      "Resource": "*"
    }
  ]
}
```

Note the `/index/*` resource — **index access is a separate permission**. Forgetting it is a top-three cause of `AccessDeniedException`.

### 21.4 Fine-grained access control (row and column level)

This is DynamoDB's standout security feature: restrict a principal to **only their own items**, enforced by the service.

```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem", "dynamodb:Query", "dynamodb:PutItem", "dynamodb:UpdateItem"],
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/UserData",
  "Condition": {
    "ForAllValues:StringEquals": {
      "dynamodb:LeadingKeys": ["${cognito-identity.amazonaws.com:sub}"],
      "dynamodb:Attributes": ["UserId", "Profile", "Preferences"]
    },
    "StringEqualsIfExists": { "dynamodb:Select": "SPECIFIC_ATTRIBUTES" }
  }
}
```

Condition keys worth knowing:

| Key | Restricts |
|---|---|
| `dynamodb:LeadingKeys` | Which partition key values the principal may touch |
| `dynamodb:Attributes` | Which attributes may be read or written (column-level) |
| `dynamodb:Select` | Forces `SPECIFIC_ATTRIBUTES` so attribute filtering can't be bypassed |
| `dynamodb:ReturnValues` | Prevents leaking data via `ALL_OLD` |
| `dynamodb:EnclosingOperation` | Restricts use inside `BatchGetItem`/`BatchWriteItem` |

This lets a mobile app talk **directly** to DynamoDB with Cognito credentials, with no backend, and still be safe.

### 21.5 Resource-based policies

Attach a policy to the table itself for cross-account access without role assumption:

```bash
aws dynamodb put-resource-policy \
  --resource-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --policy file://table-policy.json
```

### 21.6 Network — VPC endpoints

DynamoDB has a public endpoint by default. To keep traffic off the internet:

- **Gateway endpoint** (free) — route-table based, works from within a VPC, no ENI. The traditional choice.
- **Interface endpoint / PrivateLink** (hourly + data charges) — gives a private IP, works from on-premises over Direct Connect/VPN, and supports endpoint policies.

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0abc123 \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-0def456
```

Then lock the table down so it is *only* reachable through your VPC:

```json
{ "Effect": "Deny", "Action": "dynamodb:*", "Resource": "*",
  "Condition": { "StringNotEquals": { "aws:SourceVpce": "vpce-0123456789abcdef0" } } }
```

### 21.7 Auditing

- **CloudTrail** logs control-plane calls (`CreateTable`, `UpdateTable`, `DeleteTable`) by default.
- **CloudTrail data events** must be explicitly enabled to log `PutItem`/`GetItem`-level activity — high volume, so scope it to sensitive tables.
- **DynamoDB Streams with `OLD_IMAGE`** is often the more practical audit trail for data changes.

---

## 22. Data Modeling & Single-Table Design

### 22.1 The method

```
STEP 1  Draw the entity-relationship diagram (yes, even for NoSQL)
STEP 2  Write down EVERY access pattern as a sentence
STEP 3  For each, note: input parameters, expected output, frequency, latency need
STEP 4  Design a key schema that satisfies as many as possible
STEP 5  Add GSIs for the leftovers
STEP 6  Validate in NoSQL Workbench with realistic sample data
STEP 7  Load-test before you commit
```

### 22.2 An access pattern worksheet

| # | Access pattern | Parameters | Operation | Key / Index |
|---|---|---|---|---|
| 1 | Get user profile | `userId` | GetItem | Table: `PK=USER#<id>, SK=PROFILE` |
| 2 | List a user's orders, newest first | `userId` | Query | Table: `PK=USER#<id>, begins_with(SK,'ORDER#')`, reverse |
| 3 | Get one order with its line items | `orderId` | Query | GSI1: `PK=ORDER#<id>` |
| 4 | List all pending orders | — | Query | GSI2 (sparse): `PK=PENDING` |
| 5 | Find user by email | `email` | Query | GSI3: `PK=EMAIL#<email>` |
| 6 | Orders in a date range for a user | `userId, from, to` | Query | Table: `SK BETWEEN` |

Write this table **before writing any code**. If you can't fill it in, you're not ready to create the table.

### 22.3 Single-table design

Rather than one table per entity, put every entity in one table with generic key names and a type discriminator.

```
Table: AppData      PK (string)      SK (string)
┌──────────────────┬──────────────────────┬───────────────────────────────────┐
│ PK               │ SK                   │ Attributes                        │
├──────────────────┼──────────────────────┼───────────────────────────────────┤
│ USER#1042        │ PROFILE              │ Type=User, Email, Name, Tier      │
│ USER#1042        │ ORDER#2026-08-04#001 │ Type=Order, Total, Status         │
│ USER#1042        │ ADDRESS#HOME         │ Type=Address, Line1, City         │
│ ORDER#001        │ ITEM#SKU-9           │ Type=LineItem, Qty, Price         │
│ ORDER#001        │ ITEM#SKU-3           │ Type=LineItem, Qty, Price         │
│ PRODUCT#SKU-9    │ METADATA             │ Type=Product, Name, Price, Stock  │
└──────────────────┴──────────────────────┴───────────────────────────────────┘

GSI1 (overloaded):  GSI1PK / GSI1SK
  Order items  → GSI1PK=ORDER#001,   GSI1SK=ITEM#SKU-9
  Products     → GSI1PK=CATEGORY#EL, GSI1SK=PRICE#00099
  Users        → GSI1PK=EMAIL#a@b.com, GSI1SK=USER#1042
  → one index serves three completely different lookups
```

**Why bother?**

- ✅ One `Query` can fetch a user *and* their orders *and* their addresses — one round trip instead of three.
- ✅ One table to provision, monitor, back up, and secure.
- ✅ Transactions across entities are simpler.

**The costs:**

- ❌ Cryptic to read raw. Invest in a data-access layer.
- ❌ Adding an unforeseen access pattern later can be genuinely painful.
- ❌ Harder to onboard teammates.

> 🎯 **Pragmatic verdict:** single-table design is right for high-scale, well-understood, latency-critical services. For an early-stage product with evolving requirements, multi-table is often the better *engineering* decision even if it's the worse *DynamoDB* decision. Don't cargo-cult it.

### 22.4 Standard modeling patterns

| Pattern | Problem it solves | How |
|---|---|---|
| **Composite sort key** | Multi-level hierarchy queries | `SK = "2026#08#04#ORDER#001"` → query by year, month, or day with `begins_with` |
| **Sparse GSI** | Queue / flagged-item lookup | Only write the index key on relevant items |
| **GSI overloading** | Too many indexes | Generic `GSI1PK`/`GSI1SK` reused per entity type |
| **Adjacency list** | Many-to-many relationships | Both directions stored as items; GSI inverts PK/SK |
| **Inverted index** | Reverse lookup | GSI with `PK=SK` and `SK=PK` of the base table |
| **Write sharding** | Hot partition | Suffix the partition key `0..N` |
| **Vertical partitioning** | Item > 400 KB, or hot small attributes | Split one entity into several items in the collection |
| **Materialized aggregate** | No `COUNT`/`SUM` in DynamoDB | Stream → Lambda → atomic counter item |
| **Unique constraint on a 2nd attribute** | "Email must be unique" | Transaction: put `USER#id` + put `EMAIL#addr` with `attribute_not_exists` |
| **Version history** | Audit / time travel | `SK = "v0"` for current, `v1..vN` for history |
| **Large object offload** | 400 KB limit | Store the blob in S3, keep the S3 key in DynamoDB |

### 22.5 The unique-email pattern in full

```json
{ "TransactItems": [
  { "Put": { "TableName":"AppData",
      "Item": {"PK":{"S":"USER#1042"},"SK":{"S":"PROFILE"},"Email":{"S":"a@b.com"}},
      "ConditionExpression":"attribute_not_exists(PK)" }},
  { "Put": { "TableName":"AppData",
      "Item": {"PK":{"S":"EMAIL#a@b.com"},"SK":{"S":"EMAIL#a@b.com"},"UserId":{"S":"1042"}},
      "ConditionExpression":"attribute_not_exists(PK)" }}
]}
```

Both succeed or neither does. The second item is the lock that enforces uniqueness — DynamoDB has no `UNIQUE` constraint, so you build one.

---

## 23. Step-by-Step Configuration & Implementation Guide

A complete, ordered walkthrough. Each step includes both console and CLI paths.

### Step 1 — Design before you build

Fill in the access-pattern worksheet from §22.2. Decide:
- Partition key (high cardinality, known at read time, immutable)
- Sort key (do you need range queries or item collections?)
- Which GSIs, with which projections
- Capacity mode

### Step 2 — Create the table

**Console:** DynamoDB → Tables → Create table → name, partition key, sort key → *Customize settings* → capacity mode → Create.

**CLI:**

```bash
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
      AttributeName=UserId,AttributeType=S \
      AttributeName=OrderDate,AttributeType=S \
      AttributeName=Status,AttributeType=S \
  --key-schema \
      AttributeName=UserId,KeyType=HASH \
      AttributeName=OrderDate,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes '[{
      "IndexName": "StatusIndex",
      "KeySchema": [
        {"AttributeName":"Status","KeyType":"HASH"},
        {"AttributeName":"OrderDate","KeyType":"RANGE"}],
      "Projection": {"ProjectionType":"INCLUDE","NonKeyAttributes":["UserId","Total"]}
  }]' \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --sse-specification Enabled=true,SSEType=KMS \
  --deletion-protection-enabled \
  --table-class STANDARD \
  --tags Key=Environment,Value=Production Key=Owner,Value=Platform

aws dynamodb wait table-exists --table-name Orders
```

> ⚠️ **`AttributeDefinitions` must list exactly the key attributes** — every attribute used in the table key or any index key, and nothing else. Listing a non-key attribute is the most common `create-table` validation error.

### Step 3 — Enable protection features

```bash
# PITR with a 14-day window
aws dynamodb update-continuous-backups --table-name Orders \
  --point-in-time-recovery-specification \
    "PointInTimeRecoveryEnabled=true,RecoveryPeriodInDays=14"

# TTL, if applicable
aws dynamodb update-time-to-live --table-name Orders \
  --time-to-live-specification "Enabled=true,AttributeName=ExpiresAt"
```

### Step 4 — Configure capacity

**On-demand with a safety ceiling:**

```bash
aws dynamodb update-table --table-name Orders \
  --on-demand-throughput MaxReadRequestUnits=20000,MaxWriteRequestUnits=10000
```

**Or provisioned with auto scaling:**

```bash
aws dynamodb update-table --table-name Orders \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=50,WriteCapacityUnits=25

aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
  --min-capacity 50 --max-capacity 2000

aws application-autoscaling put-scaling-policy \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
  --policy-name OrdersReadScaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
      "TargetValue": 70.0,
      "PredefinedMetricSpecification": {"PredefinedMetricType":"DynamoDBReadCapacityUtilization"},
      "ScaleOutCooldown": 60,
      "ScaleInCooldown": 300}'
```

Repeat for `WriteCapacityUnits`, and separately for each GSI (`resource-id "table/Orders/index/StatusIndex"`).

### Step 5 — Write least-privilege IAM

Create a dedicated role per application. Scope to the table ARN **and** `/index/*`. Deny `Scan` in production unless a specific job needs it. See §21.3.

### Step 6 — Load data

```bash
# Single item
aws dynamodb put-item --table-name Orders --item file://order.json

# Bulk (25 max per call)
aws dynamodb batch-write-item --request-items file://batch.json

# Very large loads: import from S3 instead (§20.2)
```

### Step 7 — Wire your application

```python
import boto3
from boto3.dynamodb.conditions import Key, Attr

ddb   = boto3.resource("dynamodb", region_name="us-east-1")
table = ddb.Table("Orders")

# Write
table.put_item(
    Item={"UserId": "U-1042", "OrderDate": "2026-08-04",
          "Status": "PENDING", "Total": 149.99},
    ConditionExpression=Attr("UserId").not_exists()
)

# Query the base table
resp = table.query(
    KeyConditionExpression=Key("UserId").eq("U-1042")
                           & Key("OrderDate").begins_with("2026-08"),
    ScanIndexForward=False,
    Limit=20
)

# Query the GSI
pending = table.query(
    IndexName="StatusIndex",
    KeyConditionExpression=Key("Status").eq("PENDING")
)

# Atomic update
table.update_item(
    Key={"UserId": "U-1042", "OrderDate": "2026-08-04"},
    UpdateExpression="SET #s = :new ADD Revisions :one",
    ConditionExpression=Attr("Status").eq("PENDING"),
    ExpressionAttributeNames={"#s": "Status"},
    ExpressionAttributeValues={":new": "SHIPPED", ":one": 1},
    ReturnValues="ALL_NEW"
)
```

**Client configuration for production:**

```python
from botocore.config import Config
cfg = Config(
    retries={"max_attempts": 10, "mode": "adaptive"},  # respects throttling signals
    connect_timeout=1,
    read_timeout=3,
    max_pool_connections=50,
)
ddb = boto3.resource("dynamodb", config=cfg)
```

### Step 8 — Add stream processing (if needed)

Create the Lambda, grant it `AWSLambdaDynamoDBExecutionRole`, then create the event source mapping with a DLQ, retry cap, and `--bisect-batch-on-function-error` (§15.4).

### Step 9 — Set up monitoring

Alarms you should have on day one:

| Alarm | Threshold |
|---|---|
| `ThrottledRequests` | > 0 for 5 minutes |
| `SystemErrors` | > 0 |
| `UserErrors` | Sustained spike (indicates a code bug) |
| `ConsumedReadCapacityUnits` | > 80% of provisioned |
| `SuccessfulRequestLatency` p99 | > 50 ms |
| `ReplicationLatency` (global tables) | > 5 seconds |
| `AccountProvisionedWriteCapacityUtilization` | > 80% |

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name Orders-Throttled \
  --namespace AWS/DynamoDB --metric-name ThrottledRequests \
  --dimensions Name=TableName,Value=Orders \
  --statistic Sum --period 300 --evaluation-periods 1 \
  --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

Enable **CloudWatch Contributor Insights** to see your hottest partition keys — the fastest way to diagnose a hot partition:

```bash
aws dynamodb update-contributor-insights \
  --table-name Orders --contributor-insights-action ENABLE
```

### Step 10 — Test, load-test, then go live

- Run functional tests against **DynamoDB Local** (`docker run -p 8000:8000 amazon/dynamodb-local`).
- Load-test against a real table sized like production; verify no throttling at 2× expected peak.
- Pre-warm throughput if a launch spike is expected (§8.6).
- Rehearse a PITR restore end to end.

### Step 11 — Infrastructure as Code

Never leave a production table click-configured. Terraform example:

```hcl
resource "aws_dynamodb_table" "orders" {
  name                        = "Orders"
  billing_mode                = "PAY_PER_REQUEST"
  hash_key                    = "UserId"
  range_key                   = "OrderDate"
  deletion_protection_enabled = true
  stream_enabled              = true
  stream_view_type            = "NEW_AND_OLD_IMAGES"

  attribute { name = "UserId"    type = "S" }
  attribute { name = "OrderDate" type = "S" }
  attribute { name = "Status"    type = "S" }

  global_secondary_index {
    name               = "StatusIndex"
    hash_key           = "Status"
    range_key          = "OrderDate"
    projection_type    = "INCLUDE"
    non_key_attributes = ["UserId", "Total"]
  }

  ttl {
    attribute_name = "ExpiresAt"
    enabled        = true
  }

  point_in_time_recovery { enabled = true }
  server_side_encryption { enabled = true }

  tags = { Environment = "Production", Owner = "Platform" }
}
```

---

## 24. Monitoring, Metrics & Observability

### 24.1 The metrics that matter (namespace `AWS/DynamoDB`)

| Metric | Watch for | Meaning |
|---|---|---|
| `ConsumedReadCapacityUnits` / `ConsumedWriteCapacityUnits` | Sustained high % | Capacity planning signal |
| `ProvisionedReadCapacityUnits` / `Write` | — | Baseline for utilization |
| `ThrottledRequests` | **Any value > 0** | Requests rejected. Investigate immediately |
| `ReadThrottleEvents` / `WriteThrottleEvents` | > 0 | Per-table/index throttle detail |
| `SuccessfulRequestLatency` | p99 rising | Latency regression; dimension by `Operation` |
| `SystemErrors` | > 0 | AWS-side 5xx; retry |
| `UserErrors` | Spike | Your 4xx — validation, access denied, bad keys |
| `ConditionalCheckFailedRequests` | Spike | Contention or a logic bug |
| `TransactionConflict` | Spike | Too many transactions on the same items |
| `ReturnedItemCount` vs `ScannedCount` | Big gap | You're filtering after reading — modeling problem |
| `OnlineIndexPercentageProgress` | — | GSI backfill progress |
| `ReplicationLatency` | > seconds | Global table lag |
| `PendingReplicationCount` | Growing | Replication backlog |
| `AgeOfOldestUnreplicatedRecord` | Growing | Kinesis CDC falling behind |
| `TimeToLiveDeletedItemCount` | — | TTL working as expected |
| `AccountProvisioned*CapacityUtilization` | > 80% | You're approaching an account-level quota |

### 24.2 Contributor Insights

Enable it per table or per index. It surfaces:
- **Most accessed partition keys** — your hot keys, named.
- **Most throttled keys** — exactly which key is being rejected.

This turns "the table is throttling somewhere" into "key `TENANT#acme` is 71% of traffic" in about thirty seconds.

### 24.3 Inline cost telemetry

Add `--return-consumed-capacity INDEXES` to any request and DynamoDB tells you the exact RCU/WCU cost, broken out by table and index. Log this in staging — it makes cost regressions visible before they reach production.

### 24.4 CloudTrail

Control-plane calls are logged automatically. Data events (`GetItem`, `PutItem`) require explicit configuration and generate significant volume; scope them to tables holding sensitive data.

---

## 25. Cost Model & Optimization

### 25.1 What you actually pay for

| Component | Billed as |
|---|---|
| **Reads** | RCU-hours (provisioned) or RRU (on-demand) |
| **Writes** | WCU-hours (provisioned) or WRU (on-demand) |
| **Storage** | Per GB-month, after the 25 GB free tier |
| **Backups** | On-demand backups per GB-month; PITR per GB-month |
| **Restores** | Per GB restored |
| **Streams** | Free for Lambda; Kinesis priced separately |
| **Global Tables** | Replicated writes (rWCU) + cross-Region data transfer |
| **DAX** | Per node-hour |
| **Export/Import** | Per GB processed |
| **Data transfer out** | Standard AWS egress rates |

> Prices vary by Region and change over time — always confirm on the [official pricing page](https://aws.amazon.com/dynamodb/pricing/).

### 25.2 The optimization checklist

1. **Eliminate Scans.** The single biggest lever. Add a GSI instead.
2. **Use eventually consistent reads** wherever correctness allows — instant 50% read saving.
3. **Right-size projections.** `ALL` on a wide table doubles storage and write cost. Use `INCLUDE`.
4. **Delete unused GSIs.** Every one multiplies your write bill.
5. **Reserved Capacity** for stable provisioned workloads — up to ~50–75% off with a 1- or 3-year commit.
6. **Standard-IA table class** when storage dominates the bill.
7. **TTL instead of delete jobs** — TTL deletions are free; `DeleteItem` costs WCU.
8. **Compress large attributes** (gzip + base64) before storing; smaller items = fewer WCU.
9. **Offload blobs to S3**, keep only the pointer in DynamoDB.
10. **Import from S3** instead of `BatchWriteItem` for bulk loads.
11. **Filter Lambda triggers** so you don't invoke on irrelevant events.
12. **Set `MaxReadRequestUnits`/`MaxWriteRequestUnits`** on on-demand tables as a runaway-cost circuit breaker.
13. **Batch aggressively** — `BatchGetItem`/`BatchWriteItem` don't reduce capacity cost but cut network round trips and Lambda duration, which *is* money.
14. **Review with Cost Explorer** grouped by `TableName` tag monthly.

### 25.3 The classic cost mistakes

| Mistake | Consequence |
|---|---|
| Scan + FilterExpression on a large table | Paying to read the whole table on every request |
| Provisioned capacity on a dev table that idles 23 h/day | Paying 24/7 for nothing — use on-demand |
| Six GSIs "just in case" | 7× write cost |
| `ALL` projection on every index | Double storage, double writes |
| Strongly consistent reads by default | 2× read cost with no benefit |
| Storing 300 KB items | Every write costs 300 WCU |

---

## 26. Quotas & Hard Limits

### 26.1 The ones you'll actually hit

| Limit | Value |
|---|---|
| **Item size** | **400 KB** (attribute names + values) |
| **Partition key value** | 1–2048 bytes |
| **Sort key value** | 1–1024 bytes |
| **Query / Scan result page** | 1 MB |
| **BatchGetItem** | 100 items or 16 MB |
| **BatchWriteItem** | 25 items or 16 MB |
| **TransactWriteItems / TransactGetItems** | 100 items, 4 MB |
| **LSIs per table** | 5, defined at creation only |
| **GSIs per table** | 20 (soft, raisable) |
| **Item collection with LSI** | 10 GB |
| **Nesting depth** | 32 levels |
| **Expression length** | 4 KB |
| **Expression attribute name/value substitutions** | 2 MB total |
| **Concurrent table operations** (create/update/delete) | 500 per account/Region (soft) |
| **Tables per account/Region** | 2,500 (soft) |
| **Provisioned capacity per table/account** | 40,000 RCU / 40,000 WCU default in most Regions (soft) |
| **Billing mode switches** | Once per 24 hours per table |
| **Table class switches** | Twice per 30-day period |
| **Stream retention** | 24 hours |
| **PITR window** | 1–35 days (configurable) |
| **On-demand backups per table** | No practical limit |
| **Tags per table** | 50 |

### 26.2 No limit on

- Table size
- Number of items in a table
- Item collection size (when there are no LSIs)
- Number of distinct partition key values

### 26.3 Checking your actual quotas

```bash
aws service-quotas list-service-quotas --service-code dynamodb --output table

aws dynamodb describe-limits   # account-level provisioned capacity limits
```

Soft limits are raised through **Service Quotas** or a support case — often within hours.

---

## 27. How to Use & Where to Use (Target Use Cases)

### 27.1 The strong-fit scenarios

**🛒 E-commerce — carts, orders, inventory**
Predictable key-based access (`cartId`, `orderId`), enormous traffic spikes on sale days, transactions for inventory decrement + order creation. On-demand or pre-warmed provisioned capacity handles Black Friday without a database team.

**🎮 Gaming — player state, leaderboards, sessions**
`PlayerId` is a perfect partition key. Atomic counters for scores. GSI on `Score` for leaderboards. DAX for the read-hot top-100. TTL for session cleanup. Global tables for regional player bases.

**📱 Mobile & web backends — user profiles, preferences, notifications**
Fine-grained access control with Cognito lets the app talk directly to DynamoDB, no backend tier. Scales to zero when nobody's using your app at 4 a.m.

**🌐 IoT & telemetry — device state and time series**
`DeviceId` partition key + timestamp sort key gives you natural time-range queries. TTL ages out raw data. Streams push to analytics.

**🎬 Media & content — metadata, watch history, recommendations**
Global tables put content metadata close to every user. Item-level reads at millisecond latency for millions of concurrent viewers.

**💰 Financial services — ledgers, transaction logs, fraud checks**
Transactions with condition expressions for consistency. MRSC global tables for zero-RPO multi-Region. PITR for compliance. Streams to real-time fraud scoring.

**🔐 Session & token stores**
The textbook use case. Key lookup, TTL expiry, huge throughput, no relationships. Replaces Redis for many teams.

**🚦 Rate limiting & idempotency keys**
Atomic counters + TTL. Conditional writes for exactly-once semantics.

**📊 SaaS multi-tenant applications**
`TenantId#EntityId` keys with fine-grained IAM for tenant isolation. Watch for the noisy-neighbour hot partition on your largest tenant.

**⚡ Event sourcing & CQRS**
Append-only items in an item collection; Streams project into read models in OpenSearch, Redshift, or another DynamoDB table.

**🔄 Serverless microservices**
The Lambda + DynamoDB pairing is the canonical serverless stack: both scale to zero, both scale up automatically, neither needs a VPC.

### 27.2 A decision flowchart

```
Do you need joins, ad-hoc queries, or complex aggregations?
  ├─ YES ──► RDS / Aurora (or Aurora + DynamoDB together)
  └─ NO
      │
      Do you know your access patterns in advance?
      ├─ NO ───► RDS / Aurora / DocumentDB (or prototype in RDS, migrate later)
      └─ YES
          │
          Do you need <10 ms latency at unpredictable scale?
          ├─ YES ──► ✅ DynamoDB
          └─ NO
              │
              Is the workload spiky, or does it idle for long stretches?
              ├─ YES ──► ✅ DynamoDB (on-demand — scales to zero)
              └─ NO ───► Either works; compare cost at your steady state
```

---

## 28. When NOT to Use DynamoDB

Be honest about this. The wrong database choice is expensive to unwind.

| Requirement | Why DynamoDB struggles | Better fit |
|---|---|---|
| **Ad-hoc / exploratory queries** | Every query needs a matching key or index | RDS, Aurora, Athena |
| **Complex joins across entities** | No join support; denormalization only goes so far | Aurora, PostgreSQL |
| **Aggregations (`SUM`, `AVG`, `GROUP BY`)** | Must be pre-computed or streamed out | Redshift, Athena, Aurora |
| **Full-text search** | No text search engine | OpenSearch (via Zero-ETL) |
| **Objects > 400 KB** | Hard item limit | S3, with a pointer in DynamoDB |
| **Graph traversal** | No native traversal | Neptune |
| **Reporting / BI directly on the OLTP store** | Scans are ruinous | Export to S3 → Athena / Redshift |
| **Rapidly changing, unknown requirements** | Key schema is immutable | Start relational, migrate later |
| **Strict cross-entity referential integrity** | No foreign keys | Aurora |
| **Team with zero NoSQL experience and a tight deadline** | Real learning curve | Aurora Serverless v2 |

> 💡 **Hybrid is normal and good.** DynamoDB for the hot transactional path, Aurora for reporting, OpenSearch for search, S3+Athena for history. Use the right store for each access pattern.

---

## 29. DynamoDB vs Other Databases

| | **DynamoDB** | **Aurora / RDS** | **MongoDB / DocumentDB** | **Cassandra / Keyspaces** | **Redis / ElastiCache** |
|---|---|---|---|---|---|
| Model | Key-value + document | Relational | Document | Wide-column | In-memory KV |
| Ops burden | None (serverless) | Instances to manage | Cluster to manage | Cluster to manage | Nodes to manage |
| Scaling | Automatic, unlimited | Vertical + read replicas | Sharding (manual design) | Horizontal | Vertical + cluster |
| Ad-hoc queries | ❌ | ✅ | ✅ (with indexes) | Limited | ❌ |
| Joins | ❌ | ✅ | Limited (`$lookup`) | ❌ | ❌ |
| Transactions | ✅ up to 100 items | ✅ Full | ✅ | Limited | ✅ (MULTI) |
| Latency | Single-digit ms | ms to tens of ms | ms | ms | Microseconds |
| Multi-Region active-active | ✅ Native | Aurora Global (1 writer) | Manual | ✅ | Global Datastore |
| Scales to zero | ✅ | Aurora Serverless v2 (near) | ❌ | ❌ | ❌ |
| Durability | 3 AZs, always | Multi-AZ optional | Configurable | Configurable | Optional persistence |
| Best at | Known patterns at any scale | Anything relational | Flexible documents | Write-heavy at scale | Caching, ephemeral state |

---

## 30. Best Practices Checklist

### Design
- [ ] Every access pattern written down before the table is created
- [ ] Partition key has high cardinality and uniform access
- [ ] Sort key enables the range queries you need
- [ ] Zero Scans in the request path
- [ ] Projections are `INCLUDE` or `KEYS_ONLY`, not reflexively `ALL`
- [ ] Item sizes well under 400 KB; large blobs in S3
- [ ] Sparse indexes used for queue/flag patterns
- [ ] Numeric sort keys are zero-padded strings, or true Numbers

### Capacity & performance
- [ ] Capacity mode matches the traffic shape
- [ ] Auto scaling configured on tables **and** every GSI
- [ ] Warm throughput raised ahead of known spikes
- [ ] `MaxReadRequestUnits`/`MaxWriteRequestUnits` set on on-demand tables
- [ ] Eventually consistent reads by default
- [ ] Client retry policy set to `adaptive` mode with sensible timeouts
- [ ] Connection reuse enabled (`AWS_NODEJS_CONNECTION_REUSE_ENABLED=1` for Node)

### Reliability
- [ ] PITR enabled on all production tables
- [ ] Deletion protection enabled
- [ ] Restore procedure documented **and rehearsed**
- [ ] Stream consumers are idempotent
- [ ] Stream event source mappings have a DLQ, retry cap, and bisect-on-error
- [ ] `UnprocessedItems` / `UnprocessedKeys` handled with exponential backoff everywhere

### Security
- [ ] IAM scoped to specific table ARNs **and** `/index/*`
- [ ] `dynamodb:Scan` denied in production roles
- [ ] Fine-grained access control (`LeadingKeys`) for multi-tenant or direct-client access
- [ ] CMK encryption where compliance requires it
- [ ] VPC endpoint in place; non-VPC access denied by policy
- [ ] `aws:SecureTransport` enforced

### Operations
- [ ] Alarms on `ThrottledRequests`, `SystemErrors`, `UserErrors`, latency p99
- [ ] Contributor Insights enabled on high-traffic tables
- [ ] Tables tagged for cost allocation
- [ ] Everything defined in Terraform/CDK/CloudFormation, not the console
- [ ] Monthly cost review by table

---

## 31. Glossary

| Term | Meaning |
|---|---|
| **Adaptive capacity** | Automatic redistribution of throughput toward hot partitions |
| **Attribute** | A single name/value pair inside an item |
| **Backfill** | The process of populating a newly created GSI from existing data |
| **Burst capacity** | Up to 300 seconds of banked unused throughput |
| **Composite key** | Partition key + sort key together |
| **DAX** | DynamoDB Accelerator — in-memory write-through cache |
| **Eventually consistent read** | Cheaper read that may return slightly stale data |
| **GSI** | Global Secondary Index — independent key schema and capacity |
| **Hot partition** | A partition receiving disproportionate traffic |
| **Item** | A single record; max 400 KB |
| **Item collection** | All items sharing one partition key value |
| **LSI** | Local Secondary Index — same partition key, alternate sort key |
| **MREC / MRSC** | Multi-Region eventual / strong consistency for global tables |
| **PartiQL** | SQL-compatible query syntax over DynamoDB |
| **PITR** | Point-in-Time Recovery — continuous backup with second-level granularity |
| **Projection** | The set of attributes copied into a secondary index |
| **RCU / WCU** | Read / Write Capacity Unit |
| **RRU / WRU** | Read / Write Request Unit (on-demand billing) |
| **Sparse index** | An index containing only items that have the index key attributes |
| **Split for heat** | Automatic partition split in response to concentrated traffic |
| **Streams** | 24-hour ordered change log of item-level modifications |
| **TTL** | Time To Live — automatic, free expiry-based deletion |
| **Warm throughput** | The rate a table's current partitions can absorb instantly |
| **Witness** | An MRSC component holding replication data without being a full replica |
| **Write sharding** | Adding a suffix to a partition key to spread load |

---

## 32. Learning Path & Resources

### Suggested order

```
Week 1  §1–7    Concepts, keys, partitioning, data types
        Labs 1–3  Create a table, CRUD, queries vs scans
Week 2  §8–12   Capacity, consistency, reads, writes, expressions
        Labs 4–6  Capacity tuning, conditional writes, expressions
Week 3  §13–17  Indexes, transactions, streams, TTL, DAX
        Labs 7–9  GSIs, streams + Lambda, TTL
Week 4  §18–26  Multi-Region, backup, security, modeling, cost
        Labs 10–12 Single-table design, backup/restore, full serverless API
```

### Official documentation
- [DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)
- [Best Practices for DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [API Reference](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/)
- [Pricing](https://aws.amazon.com/dynamodb/pricing/)
- [Service Quotas](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/ServiceQuotas.html)

### Highly recommended
- **NoSQL Workbench** — model visually, generate code, run queries locally
- **The DynamoDB Book** by Alex DeBrie — the definitive data modeling text
- **DynamoDB Local** — free offline development
- **AWS re:Invent DAT session track** — the deep-dive talks are genuinely excellent
- The original **Dynamo paper** (2007) — for understanding *why* it works this way

---

## Next Steps

➡️ **[commands-cheatsheet.md](./commands-cheatsheet.md)** — every CLI command, organized by task
➡️ **[hands-on-labs.md](./hands-on-labs.md)** — 12 labs from first table to deployed API
➡️ **[troubleshooting.md](./troubleshooting.md)** — when things break

---

<div align="center">

**⭐ If this helped you, star the repo.**

Built as a practical reference — corrections and additions welcome via issues or PRs.

</div>
