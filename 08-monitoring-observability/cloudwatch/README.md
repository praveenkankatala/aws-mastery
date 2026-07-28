# Amazon CloudWatch — End-to-End Guide

> **What:** AWS's native monitoring and observability service — a central repository for metrics, logs, traces, and events, with built-in visualization, alerting, and automated response.
>
> **Why:** You cannot fix what you cannot see. CloudWatch tells you *when* something breaks (alarms), *what* broke (metrics/dashboards), and *why* it broke (logs, traces, insights) — without deploying a separate ELK/Prometheus stack.
>
> **How:** Data flows through a four-stage pipeline: **Collect → Monitor → Act → Analyze.** This document walks each stage end to end.

---

## Table of Contents

1. [High-Level Architecture & Service Flow](#1-high-level-architecture--service-flow)
2. [Stage 1 — Collect: Data Ingestion](#2-stage-1--collect-data-ingestion)
3. [Stage 2 — Monitor: Visualization](#3-stage-2--monitor-visualization)
4. [Stage 3 — Act: Alarms & Automated Response](#4-stage-3--act-alarms--automated-response)
5. [Stage 4 — Analyze: Deep Insights](#5-stage-4--analyze-deep-insights)
6. [Service Metrics Deep-Dive (EC2 · S3 · Lambda · API Gateway)](#6-service-metrics-deep-dive)
7. [EKS / Kubernetes Monitoring](#7-eks--kubernetes-monitoring)
8. [Custom Metrics (PutMetricData vs EMF, Cardinality, Cost)](#8-custom-metrics)
9. [Distributed Tracing — X-Ray & Application Signals (APM)](#9-distributed-tracing--x-ray--application-signals)
10. [Trace-to-Log Correlation Architecture](#10-trace-to-log-correlation)
11. [Cost Optimization & Storage Classes](#11-cost-optimization--storage-classes)
12. [Additional Capabilities (Synthetics, RUM, Cross-Account)](#12-additional-capabilities)
13. [Step-by-Step Configuration Guide](#13-step-by-step-configuration-guide)
14. [Where to Use It — Target Use Cases](#14-where-to-use-it--target-use-cases)

---

## 1. High-Level Architecture & Service Flow

```
 ┌────────────────────────  COLLECT  ─────────────────────────┐
 │  EC2 / EKS / Lambda / RDS / S3 / API GW / On-prem servers  │
 │        │ vended metrics      │ logs (agent / native)       │
 │        ▼                     ▼                             │
 │  ┌──────────────┐      ┌──────────────────┐                │
 │  │  CW Metrics  │      │  CW Logs         │   ┌──────────┐ │
 │  │  (namespace/ │      │  (log groups /   │   │ Events / │ │
 │  │  dimensions) │      │   log streams)   │   │EventBridge│ │
 │  └──────┬───────┘      └────────┬─────────┘   └────┬─────┘ │
 └─────────┼───────────────────────┼──────────────────┼───────┘
           ▼                       ▼                  ▼
 ┌──────────────────────  MONITOR  ───────────────────────────┐
 │  Dashboards · ServiceLens · Container/Lambda Insights ·    │
 │  Application Signals (service map, SLOs)                   │
 └─────────┬──────────────────────────────────────────────────┘
           ▼
 ┌──────────────────────  ACT  ───────────────────────────────┐
 │  Alarms (static / anomaly-detection / composite)           │
 │     → SNS (email, SMS, Slack)                              │
 │     → Auto Scaling (add/remove instances)                  │
 │     → Lambda (auto-remediation)                            │
 │     → EventBridge rules → any target                       │
 └─────────┬──────────────────────────────────────────────────┘
           ▼
 ┌──────────────────────  ANALYZE  ───────────────────────────┐
 │  Logs Insights (query TBs in seconds) · Contributor        │
 │  Insights (top talkers) · Metric Streams (export to        │
 │  Datadog/S3) · X-Ray traces · Metric Math                  │
 └────────────────────────────────────────────────────────────┘
```

**The workflow loop, narrated:**
1. Your application generates a log and a metric.
2. CloudWatch ingests these via the **Unified CloudWatch Agent**.
3. An **Alarm** detects a metric crossed a safety threshold.
4. An **Action** fires — notifies the on-call engineer and scales infrastructure.
5. The engineer uses **Logs Insights** to query the logs and find root cause.

---

## 2. Stage 1 — Collect: Data Ingestion

CloudWatch is a central repository for all operational data, in three categories:

### 2.1 Metrics — numeric time-series data

A metric is a numeric data point representing performance (CPU utilization, disk I/O, request count).

| Type | Source | Cost | Example |
|---|---|---|---|
| **Vended (default) metrics** | Sent automatically by AWS services (EC2, RDS, S3, Lambda…) | Free | `AWS/EC2 CPUUtilization` |
| **Custom metrics** | Sent by *you* via CLI/SDK/EMF | $0.30/metric/month (first 10k) | `ECommerceApp/SuccessfulLogins` |

**Anatomy of every metric data point:**
- **Namespace** — logical container isolating metrics (e.g., `ECommerceApp/Production`). Never starts with `AWS/` (reserved).
- **Metric Name** — the behavior tracked (`OrderValue`, `ProcessingDelay`).
- **Dimensions** — name/value pairs for filtering & aggregation (`Environment=Prod`, `PaymentMethod=CreditCard`).
- **Value + Unit** — the number and its scale (`45.2 Seconds`, `Bytes`, `Count`).

**Resolution:**
- **Standard:** 1-minute minimum granularity (5-minute default for basic EC2 monitoring; enable *Detailed Monitoring* for 1-minute).
- **High-Resolution:** down to **1-second** granularity — set `StorageResolution: 1` when publishing. Use for flash-traffic, financial processing, circuit-breakers.

**Retention (automatic tiering — metrics can never be deleted, they expire):**
| Resolution | Retained for |
|---|---|
| < 60 s (high-res) | 3 hours |
| 1 minute | 15 days |
| 5 minutes | 63 days |
| 1 hour | 15 months |

### 2.2 Logs — granular event records

**CloudWatch Logs** collects from EC2 instances (via agent), Lambda (native), ECS/EKS (Fluent Bit), CloudTrail, Route 53, VPC Flow Logs, and on-prem servers.

**Hierarchy:** `Log Group` (application/resource level, where retention & permissions live) → `Log Stream` (a single source, e.g., one instance) → `Log Events` (timestamped lines).

Key features on top of raw logs:
- **Metric Filters** — turn log patterns into metrics ("count every `ERROR` line") → alarm on them.
- **Subscription Filters** — stream logs in real time to Lambda, Kinesis, Firehose, or OpenSearch.
- **Retention policies** — default is *Never Expire* (a cost trap!). Set 30/90-day retention explicitly.
- **Export to S3** — batch archive for long-term compliance.

### 2.3 Events — EventBridge

A stream of system events describing changes in AWS resources (e.g., EC2 state `running → stopped`). CloudWatch Events evolved into **Amazon EventBridge**: rules match event patterns or schedules (cron) and route to targets (Lambda, SQS, SNS, Step Functions).

---

## 3. Stage 2 — Monitor: Visualization

| Tool | What it gives you |
|---|---|
| **Dashboards** | Custom single-pane-of-glass views combining metrics across resources and even **regions**. Support metric math, text widgets, alarms status, log widgets. Shareable (public link or SSO). |
| **ServiceLens** | End-to-end application health view mapping dependencies and highlighting bottlenecks (combines metrics + logs + X-Ray traces). |
| **Container Insights** | Auto-aggregated diagnostics for ECS/EKS/Fargate — cluster/node/pod/container-level metrics. |
| **Lambda Insights** | Cold starts, memory usage, init duration per function. |
| **Automatic Dashboards** | Pre-built per-service dashboards, zero configuration. |

---

## 4. Stage 3 — Act: Alarms & Automated Response

CloudWatch isn't a passive observer — it triggers actions.

### Alarm types

| Type | How it works | Example |
|---|---|---|
| **Static threshold** | Fixed value comparison over N periods | "CPU > 80% for 5 consecutive minutes" |
| **Anomaly detection** | ML model learns the metric's historical band; alarms on deviation | "Request count outside expected band" |
| **Composite alarms** | Boolean combination (AND/OR) of other alarms — reduces alert noise | "Alarm only if (HighCPU AND HighLatency)" |
| **Metric math alarms** | Alarm on an expression | "Error rate = Errors/Invocations > 1%" |

### Alarm states

`OK` → threshold not breached · `ALARM` → breached · `INSUFFICIENT_DATA` → not enough data points (common with new/sparse metrics — see troubleshooting doc).

### Actions on state change

- **Notify** via Amazon SNS (email, SMS, → Slack/Teams via Lambda or Chatbot)
- **Auto Scaling** — add/remove EC2 instances or ECS tasks
- **EC2 actions** — stop, terminate, reboot, or **recover** an instance (pairs with `StatusCheckFailed_System`)
- **Lambda / SSM OpsItems** — automated remediation runbooks

---

## 5. Stage 4 — Analyze: Deep Insights

| Tool | Purpose | Example question |
|---|---|---|
| **Logs Insights** | Purpose-built query language; search & analyze TBs of logs in seconds — regex, aggregations, sorting — no ELK stack needed | "Find all 404 errors in my web server logs" |
| **Contributor Insights** | Identifies "top talkers" — the users/hosts/IPs driving load | "Which 5 IPs generate 90% of my 4xx errors?" |
| **Metric Streams** | Near-real-time export of metrics to third parties (Datadog, New Relic) or S3 data lake via Kinesis Firehose | Feed an external observability platform without API polling |
| **Metric Math** | Combine/transform metrics with expressions (`SUM`, `RATE`, `FILL`, `ANOMALY_DETECTION_BAND`) | Compute error-rate % from two raw counters |

**Logs Insights starter query:**
```sql
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as errorCount by bin(5m)
| sort @timestamp desc
| limit 50
```

---

## 6. Service Metrics Deep-Dive

The metrics that matter most for a classic serverless/microservices stack, with recommended alarm thresholds.

### 6.1 Amazon EC2 (`AWS/EC2`)

Default metrics are **hypervisor-level**. ⚠️ **Memory utilization and disk-space % are NOT available by default** — install the Unified CloudWatch Agent on the OS to get them (they land in the `CWAgent` namespace).

| Metric | Tracks | Alarm guidance |
|---|---|---|
| `CPUUtilization` | % of allocated compute in use | > 80% for 5 consecutive minutes |
| `StatusCheckFailed` | Instance (OS/software) + System (AWS hardware) health | > 0 → trigger EC2 recovery / page engineer |
| `NetworkIn` / `NetworkOut` | Bytes received/sent | Sudden drops or spikes (DDoS, disconnects) |
| `DiskReadOps` / `DiskWriteOps` | Local disk I/O operations | Storage bottleneck identification |
| `CPUCreditBalance` | Burst credits (T-type instances only) | < 50 credits (throttles hard at 0) |

### 6.2 Amazon S3 (`AWS/S3`)

Two families: **Storage metrics** (daily, free) and **Request metrics** (1-minute, must be **explicitly enabled** in the S3 console per bucket/prefix).

| Metric | Tracks | Alarm guidance |
|---|---|---|
| `BucketSizeBytes` | Total stored volume across storage classes | Daily budget/capacity tracking |
| `NumberOfObjects` | Object count | Detect runaway loops creating millions of files |
| `AllRequests` | Total GET/PUT/DELETE/HEAD hits | Spikes = traffic load or scanning activity |
| `4xxErrors` | Client errors (403 denied, 404 not found) | Rise = broken app code or expired IAM keys |
| `5xxErrors` | Server-side errors / throttling | > 0 (likely hitting per-prefix request-rate limits) |
| `TotalRequestLatency` | First byte in → last byte out | > 200 ms (tune to your baseline) |

### 6.3 AWS Lambda (`AWS/Lambda`)

| Metric | Tracks | Alarm guidance |
|---|---|---|
| `Invocations` | Executions of your function | Rapid drop to 0 or unexplained spikes |
| `Errors` | Failed executions (unhandled exceptions) | > 1, or error rate > 1% of traffic |
| `Duration` | Runtime in ms | Compare against configured timeout |
| `Throttles` | Rejected invocations (concurrency exceeded) | > 0 → adjust reserved concurrency / raise quota |
| `ConcurrentExecutions` | Simultaneous active instances | Chart vs regional pool limit (default 1,000) |
| `IteratorAge` | Lag of oldest record (Kinesis/DynamoDB Streams) | High = consumer too slow for the stream |

### 6.4 Amazon API Gateway (`AWS/ApiGateway`)

These represent the **end-user experience** — your front door.

> **Pro tip:** enable *Detailed CloudWatch Metrics* in Stage settings to break metrics down per route/method instead of one API-wide aggregate.

| Metric | Tracks | Alarm guidance |
|---|---|---|
| `Count` | Total requests served | Load variation / traffic trends |
| `4XXError` | Client errors (401, 429) | Elevated = auth failures or rate limits |
| `5XXError` | Backend/API GW infrastructure errors | > 0 → immediate alarm; API is failing users |
| `Latency` | Request in → response out (total) | End-user frustration tracker |
| `IntegrationLatency` | Time waiting on backend (Lambda/HTTP) | Compare vs `Latency` to isolate slowdown |

### 6.5 The Golden Metrics Trio (dashboard pattern)

For an `API Gateway → Lambda → S3` pattern, group these three charts side by side:

1. **Traffic:** API GW `Count` next to Lambda `Invocations`
2. **Errors:** API GW `5XXError` stacked with Lambda `Errors` + `Throttles`
3. **Latency breakdown:** one line graph with `Latency` vs `IntegrationLatency` vs `Duration`
   - `IntegrationLatency` + `Duration` rise together → **your Lambda code is the bottleneck**
   - Only `Latency` rises → **API GW config or authorizers are dragging it down**

---

## 7. EKS / Kubernetes Monitoring

Kubernetes adds ephemeral abstraction layers (**Cluster → Node → Namespace → Pod → Container**), so you monitor both AWS infrastructure *and* internal Kubernetes state.

### 7.1 The three telemetry pillars

**Pillar A — Control Plane Logs** (must be explicitly enabled; AWS runs the control plane but you own visibility):
- **API Server & Audit logs** — security: who did what inside the cluster
- **Controller Manager & Scheduler** — debug scheduling failures (why is a pod `Pending`?)
- *Pro tip:* never leave these on infinite retention — set 30/90-day CloudWatch retention to control ingestion cost.

**Pillar B — Infrastructure & Kube-State Metrics:**
- **Node metrics:** CPU/memory utilization, disk pressure, network I/O
- **Kube-state metrics:** pod phase (Running/Failed/Pending), container restarts, HPA scaling decisions

**Pillar C — Application Telemetry (RED method):**
- **R**ate — requests per second
- **E**rrors — HTTP 5xx / unhandled exceptions
- **D**uration — p50 / p95 / p99 latency
- *Why:* "node at 40% CPU" tells you nothing about user experience.

### 7.2 Choosing your stack

| Feature | AWS Native | Cloud-Native OSS |
|---|---|---|
| Collection agent | `amazon-cloudwatch-observability` EKS add-on | Prometheus Operator / OpenTelemetry Collector |
| Metrics storage | CloudWatch Container Insights | Amazon Managed Prometheus (AMP) / Grafana Mimir |
| Log routing | Fluent Bit → CloudWatch Logs | Fluent Bit / Vector → OpenSearch or Grafana Loki |
| Visualization | CloudWatch Automatic Dashboards | Grafana |
| Management | Fully managed (set-and-forget, higher AWS cost) | Self-managed OSS standard (max flexibility) |

### 7.3 Critical EKS metrics & alerts (Container Insights, Enhanced Observability)

**Cluster & Node layer:**
| Metric | Meaning | Alert |
|---|---|---|
| `node_cpu_utilization` / `node_memory_utilization` | Total consumption per worker node | > 85% → autoscaler (Karpenter) may not be provisioning fast enough |
| `node_disk_utilization` | Worker root-volume usage (uncleaned container logs, image caches) | > 85% → triggers `DiskPressure` taint → pod evictions |

**Pod & Container layer:**
| Metric | Meaning | Alert |
|---|---|---|
| `pod_number_of_container_restarts` | Container repeatedly crashing | > 3 restarts in 10 min → `CrashLoopBackOff` (code errors / missing env vars) |
| `container_memory_oom_killed` | Exceeded manifest memory limit; kernel killed it | > 0 → memory limits undersized |
| `container_cpu_cfs_throttled_periods` | Hitting CPU limit; Kubernetes artificially slowing it | Steep rises → raise container CPU limits |

### 7.4 Architectural best practices

1. **Define resource Requests and Limits early.** Kubernetes schedules on *requests* and caps on *limits*. Without them, one leaking microservice can starve a whole worker node and take down unrelated neighbor pods.
2. **Aggressively filter logs at the source.** Container Insights ingestion costs spiral with verbose containers. Drop noisy namespaces in Fluent Bit:
   ```ini
   [FILTER]
       Name kubernetes
       Match kube.*
       K8S-Logging.Exclude On
       K8S-Logging.Exclude_Namespaces kube-system monitoring ingress-nginx
   ```
3. **Stream Kubernetes Events into logs.** Standard metrics won't show *why* an image failed to pull. Ship `kubectl get events` output so you can search `ErrImagePull`, `FailedScheduling`, and failed readiness probes alongside metrics.

---

## 8. Custom Metrics

AWS out-of-the-box metrics cannot track internal application performance or business logic (`SuccessfulCheckouts`, `PaymentProcessingTime`, `ActiveUserSessions`). Custom metrics fill that gap — but there are two ingestion methods and one expensive trap.

### 8.1 Method A — Traditional `PutMetricData` API call

- **How:** synchronous, blocking HTTP POST via SDK/CLI.
- **When:** persistent infrastructure (EC2, ECS-on-EC2, on-prem) where a background daemon pushes metrics asynchronously.
- ⚠️ **The trap:** never call it inline inside a high-throughput **Lambda** — it adds **50–100 ms of latency to every invocation**, balloons your runtime bill, and risks API throttling.

### 8.2 Method B — Embedded Metric Format (EMF) — the modern standard

Instead of an API call, your app prints a structured JSON object to **stdout**. CloudWatch Logs captures it; a backend parser reads the `_aws` metadata block and extracts metrics automatically.

**When:** Lambda, ECS Fargate, Kubernetes — strongly recommended.

```json
{
  "_aws": {
    "Timestamp": 1717613023000,
    "CloudWatchMetrics": [
      {
        "Namespace": "RetailApp/Payments",
        "Dimensions": [["Environment", "Region"]],
        "Metrics": [
          { "Name": "TransactionProcessingTime", "Unit": "Milliseconds" }
        ]
      }
    ]
  },
  "Environment": "Production",
  "Region": "ap-south-1",
  "TransactionProcessingTime": 245,
  "OrderId": "ord-99831",
  "CustomerId": "cust-552"
}
```

**The secret benefit:** `OrderId` and `CustomerId` sit in the JSON body but are **not declared as dimensions** — so you can still query individual transactions later via Logs Insights *without paying for them as unique metrics*.

### 8.3 The cost trap: cardinality

Unique metrics bill at **$0.30/metric/month** (first 10,000). CloudWatch treats **every unique combination of dimension values as a separate billed metric** — a dimension is *not* a database column.

**❌ High-cardinality (the bad setup):**
```
Metric: ApiLatency
  Dimension: Environment=Prod
  Dimension: CustomerId=12345   <- unique per user!
```
50,000 active users = 50,000 unique metric streams = **50,000 × $0.30 = $15,000/month.**

**✅ Low-cardinality (the correct setup):**
```
Metric: ApiLatency
  Dimension: Environment=Prod
  Dimension: Tier=Premium       <- only 3 values: Free/Standard/Premium
```
Exactly 3 unique metrics = **< $1.00/month**, still fully actionable. Keep unique IDs in log properties/EMF payloads for Logs Insights.

### 8.4 High-resolution metrics

- Standard resolution = 1-minute minimum. Too slow for flash-traffic or financial circuit-breakers?
- Set `StorageResolution: 1` (API or EMF schema) for **1-second** data points.

---

## 9. Distributed Tracing — X-Ray & Application Signals

### 9.1 CloudWatch Application Signals (APM)

Transforms CloudWatch into a full APM competing with Datadog/New Relic:
- **Zero-code instrumentation** — the CloudWatch agent auto-discovers and instruments apps on EKS, EC2, ECS (Java, Node.js, Python) with no code changes.
- **SLOs (Service Level Objectives)** — monitor business objectives directly: *"99.00% of checkout requests < 200 ms over a 7-day rolling window."*
- **Application Map** — auto-visualizes microservice communication across accounts/regions, surfacing anomalies and high-fault components.

### 9.2 AWS X-Ray (distributed tracing)

When an API call fails or slows in a microservice environment, metrics won't show *why*. X-Ray follows the request's entire journey:
- Injects a unique **`X-Amzn-Trace-Id`** header into incoming HTTP requests.
- Every hop (API GW → Lambda → S3 → external API) records a **span**; spans compose into a **trace** with a segment-timeline breakdown.
- **Sampling rules** control volume/cost — e.g., 5% of successful requests but **100% of errors**.
- **Log correlation** — tie traces directly to the exact log lines written during that execution window, slashing MTTR.

---

## 10. Trace-to-Log Correlation

The ultimate debugging setup: open the exact trace timeline, click the bottleneck, and see the line-by-line logs from that microsecond.

### 10.1 The concept

Traces and logs must share one common identifier: the **X-Ray Trace ID**. The tracing agent injects the ID into request context; your job is to stamp it onto **every log line**. CloudWatch detects the string and builds a clickable link between the tools.

### 10.2 Step 1 — Inject the Trace ID into application logs

**Python (manual, standard `logging` + X-Ray SDK):**
```python
import logging
from aws_xray_sdk.core import xray_recorder

logger = logging.getLogger()
logger.setLevel(logging.INFO)

formatter = logging.Formatter(
    '[%(levelname)s] [%(asctime)s] [TraceID: %(xray_trace_id)s] %(message)s'
)

class XRayContextFilter(logging.Filter):
    def filter(self, record):
        current_segment = xray_recorder.get_current_segment()
        record.xray_trace_id = current_segment.trace_id if current_segment else "N/A"
        return True
```

**Serverless (AWS Lambda Powertools — zero boilerplate):**
```python
from aws_lambda_powertools import Logger
logger = Logger()   # auto-includes "xray_trace_id" in structured JSON logs
```

### 10.3 Step 2 — Route via the Unified CloudWatch Agent

On EC2/EKS, deploy the unified agent (or the `amazon-cloudwatch-observability` add-on). It runs a local daemon listening on **OTLP** (OpenTelemetry protocol) ports, accepting traces from your code and shipping them to CloudWatch.

### 10.4 Step 3 — Query the correlation

**Traces → Logs:** CloudWatch → Application Signals / X-Ray Traces → click a slow/faulted trace → segment timeline (API GW 10 ms, Lambda 200 ms, DynamoDB 150 ms) → **"Logs" tab** auto-runs a Logs Insights query fetching only rows stamped with that `X-Amzn-Trace-Id`.

**Logs → Traces:** from Logs Insights, parse the ID out and deep-link back:
```sql
fields @timestamp, @message, @logStream
| filter @message like /TraceID:/
| parse @message "TraceID: *" as TraceId
| display @timestamp, TraceId, @message
```

### 10.5 Implementation checklist

- **Time sync (NTP):** clocks drifting > a few seconds between app servers and AWS will break chronological stitching in the UI.
- **Sampling alignment:** match your logging verbosity to trace sampling — don't log debug rows for transactions that were never traced.
- **EKS shortcut:** enabling the Application Signals flag on the cluster sets up the whole loop automatically for Java/Node.js/Python.

---

## 11. Cost Optimization & Storage Classes

Observability data grows exponentially; unchecked, CloudWatch ingestion can outpace your compute bill.

| Lever | What it does |
|---|---|
| **Logs Standard class** | Full feature set: real-time streaming, metric filters, live anomaly detection |
| **Logs Infrequent Access (Logs-IA)** | For massive-ingest, rarely-queried logs (VPC Flow Logs, debug logs) — dramatically cheaper ingestion; query only during incidents/audits |
| **Retention policies** | Default is *Never Expire* — always set explicit 30/90-day retention per log group |
| **Metric Streams** | Push metrics continuously to Kinesis Firehose → S3 lake or third parties, avoiding `GetMetricData` polling costs and API throttling |
| **Low cardinality** | See §8.3 — the single biggest custom-metrics cost lever |
| **Fluent Bit filtering** | Drop noisy namespaces at the source (see §7.4) |

---

## 12. Additional Capabilities

Standard concepts worth knowing that round out the service:

- **CloudWatch Synthetics (Canaries):** scheduled headless-browser/API scripts that probe your endpoints from the *outside* — catch outages before users do (uptime, broken links, UI flows, latency from user perspective).
- **CloudWatch RUM (Real User Monitoring):** JavaScript snippet capturing actual browser-side performance — page load, Core Web Vitals, JS errors, by geography/browser.
- **CloudWatch Evidently:** feature flags and A/B experiments with built-in metrics evaluation.
- **Cross-Account Observability:** designate a central *monitoring account*; source accounts share metrics/logs/traces — a true single pane of glass for AWS Organizations.
- **Internet Monitor / Network Monitor:** visibility into internet-path and hybrid-network performance affecting your workloads.
- **OpenTelemetry (ADOT):** the AWS Distro for OpenTelemetry is the vendor-neutral collector that can feed CloudWatch, AMP, and third parties simultaneously.

---

## 13. Step-by-Step Configuration Guide

A condensed production setup sequence (full walkthroughs in [hands-on-labs.md](hands-on-labs.md)):

1. **IAM first:** attach `CloudWatchAgentServerPolicy` to the EC2/EKS node role.
2. **Install the Unified CloudWatch Agent** on EC2 (gets you memory + disk %, which are missing by default).
3. **Set log-group retention** the moment a log group is created (30/90 days).
4. **Create metric filters** on error patterns → back them with alarms.
5. **Build alarms:** static for known limits, anomaly detection for traffic-shaped metrics, composite to de-noise paging.
6. **Wire SNS** topics → email/Slack; test with `set-alarm-state`.
7. **Build one dashboard per service** using the Golden Trio pattern (§6.5).
8. **Enable Container Insights** (EKS add-on) and control-plane logs for Kubernetes clusters.
9. **Adopt EMF** for all serverless custom metrics; audit dimensions for cardinality.
10. **Enable X-Ray / Application Signals** and inject trace IDs into logs (§10).
11. **Review costs monthly:** ingestion bytes per log group, custom-metric count, Logs-IA candidates.

---

## 14. Where to Use It — Target Use Cases

| Scenario | CloudWatch features to reach for |
|---|---|
| "Page me when prod CPU is pegged" | Metrics + static alarm + SNS |
| "Why did the app crash at 2 AM?" | CloudWatch Logs + Logs Insights |
| "Scale out on traffic spikes" | Alarm → Auto Scaling policy |
| "Auto-restart a hung instance" | `StatusCheckFailed` alarm → EC2 recover action |
| "Track checkout success rate" | Custom metrics via EMF + metric math + SLO |
| "Which customer is hammering the API?" | Contributor Insights (+ IDs in EMF payload via Logs Insights) |
| "Where in the microservice chain is it slow?" | X-Ray traces + Application Map + trace-log correlation |
| "Feed Datadog/Grafana without polling" | Metric Streams → Kinesis Firehose |
| "Monitor the site like a user would" | Synthetics canaries + RUM |
| "One dashboard for 40 AWS accounts" | Cross-Account Observability |

---

**Next:** open the [commands cheatsheet](commands-cheatsheet.md) · run the [hands-on labs](hands-on-labs.md) · bookmark [troubleshooting](troubleshooting.md).
