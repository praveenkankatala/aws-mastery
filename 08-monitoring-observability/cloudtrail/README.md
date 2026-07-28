# AWS CloudTrail — End-to-End Guide

> **What:** the security camera and compliance auditor of your AWS account. CloudTrail records **every API action** — console logins, EC2 launches, security-group edits, S3 bucket deletions — as immutable JSON audit events.
>
> **Why:** when something suspicious or destructive happens, CloudTrail answers the forensic questions: **Who** made the request, **what** resource was targeted, **when** did it happen, and **from where** (IP address) was it initiated. Without it, a breach investigation is guesswork; with it, it's a SQL query.
>
> **How:** events flow through a five-stage pipeline: **Generate → Capture → Deliver → Secure → Analyze.** This document walks each stage end to end.

**CloudWatch vs CloudTrail in one line:** CloudWatch = *observability* (is my CPU pegged?). CloudTrail = *governance* (who changed the security group at midnight?).

---

## Table of Contents

1. [High-Level Architecture & Service Flow](#1-high-level-architecture--service-flow)
2. [Stage 1 — Generate: The Three Event Types](#2-stage-1--generate-the-three-event-types)
3. [Stage 2 — Capture: Trails, Scope & Configuration](#3-stage-2--capture-trails-scope--configuration)
4. [Stage 3 — Deliver: Where the Data Lands](#4-stage-3--deliver-where-the-data-lands)
5. [Stage 4 — Secure: Log Hardening & Integrity](#5-stage-4--secure-log-hardening--integrity)
6. [Stage 5 — Analyze: Querying Audit Logs](#6-stage-5--analyze-querying-audit-logs)
7. [Anatomy of a CloudTrail Event](#7-anatomy-of-a-cloudtrail-event)
8. [CloudTrail Lake (Modern Alternative)](#8-cloudtrail-lake)
9. [Real-Time Security Alerting Pattern](#9-real-time-security-alerting-pattern)
10. [CloudWatch vs CloudTrail Summary](#10-cloudwatch-vs-cloudtrail-summary)
11. [Step-by-Step Configuration Guide](#11-step-by-step-configuration-guide)
12. [Where to Use It — Target Use Cases](#12-where-to-use-it--target-use-cases)

---

## 1. High-Level Architecture & Service Flow

```
  User / Role / Service          ┌──────────────────────────────┐
  runs an API call ─────────────►│  AWS Service (IAM, EC2, S3…) │
  (console, CLI, SDK)            └──────────────┬───────────────┘
                                                │ every call recorded
                                                ▼
                                 ┌──────────────────────────────┐
                                 │        AWS CloudTrail        │
                                 │  Management │ Data │ Insights│
                                 │   events    │events│ events  │
                                 └──────┬───────────────┬───────┘
                        (permanent      │               │  (real-time
                         audit archive) │               │   alerting)
                                        ▼               ▼
                       ┌────────────────────┐   ┌────────────────────┐
                       │  Amazon S3 bucket  │   │  CloudWatch Logs   │
                       │  .json.gz, KMS-    │   │  → metric filter   │
                       │  encrypted, Object │   │  → alarm → SNS     │
                       │  Lock, digest files│   │  → Slack/PagerDuty │
                       └─────────┬──────────┘   └────────────────────┘
                                 ▼
                       ┌────────────────────┐   ┌────────────────────┐
                       │  Amazon Athena     │   │  EventBridge rules │
                       │  (SQL forensics)   │   │  (event-driven     │
                       └────────────────────┘   │   automation)      │
                                                └────────────────────┘
```

**The complete workflow loop, narrated:**
1. An engineer runs `aws iam create-user --user-name Alice` from a terminal.
2. The AWS IAM service validates and executes the API call.
3. **CloudTrail** catches the control-plane activity, formatting an event packet with Alice's details, the engineer's IAM identity, the source IP, and the exact timestamp.
4. The log is bundled, cryptographically signed, and delivered to an encrypted **S3 security bucket**, while a copy streams to **CloudWatch Logs**.
5. A CloudWatch **metric filter** spots the `CreateUser` event and triggers a **Slack notification via SNS**, alerting the security team of a new identity in the cloud.

---

## 2. Stage 1 — Generate: The Three Event Types

Not all AWS operations are equal. CloudTrail categorizes activity into three groups:

### 2.1 Management Events (Control Plane)

- **What:** administrative operations — `CreateBucket`, `RunInstances`, IAM policy changes, console sign-ins.
- **Default:** **enabled automatically** in every account, viewable for a rolling **90 days**, at **no cost** (Event History).
- **Read vs Write:** you can log Read-only, Write-only, or All — writes are what auditors care about; reads add volume.

### 2.2 Data Events (Data Plane)

- **What:** high-frequency operations *within* a resource:
  - S3 object-level: `GetObject`, `PutObject`, `DeleteObject`
  - Lambda: `Invoke`
  - DynamoDB: item-level mutations
- **Default: disabled** — they generate massive volume (and cost). Enable selectively via **event selectors** / **advanced event selectors** (filter by bucket, prefix, event name).
- **Enterprise feature — Data Events Aggregation:** groups high-volume object access into **5-minute summaries** so security teams monitor data-access patterns without drowning in individual log lines.

### 2.3 Insights Events

- **What:** a machine-learning extension (opt-in). CloudTrail learns your account's normal API baselines and logs an **anomaly event** on unusual bursts — e.g., a developer suddenly launching 50 large EC2 instances (crypto-mining pattern) or an unexpected spike in IAM permission updates (privilege-escalation pattern).
- **Types:** API call-rate anomalies and API error-rate anomalies.

### 2.4 Network Activity Events *(newer, worth knowing)*

- **What:** API activity traversing **VPC endpoints** — catches calls made from inside your VPC that bypass the public internet, including denied attempts. Useful for data-perimeter monitoring.

---

## 3. Stage 2 — Capture: Trails, Scope & Configuration

Event History gives you 90 days free — but production needs a **permanent Trail**.

### 3.1 Multi-Region Trails — mandatory production standard

Even if you only deploy in `ap-south-1`, an attacker who compromises your keys can spin up resources in `us-east-1` to mine crypto. A multi-region trail routes API activity from **every global region** back to a single logging location. (This is the default for new trails — keep it that way.)

Also enable **global service events** (IAM, STS, CloudFront) — these are recorded in one region but affect everything.

### 3.2 Organization Trails — enterprise standard

Configured via **AWS Organizations** from the management account:
- Enforces a **unified trail across every member account**.
- Member-account admins and root users **cannot disable or modify it** — a malicious insider can't wipe their own footprints.
- All accounts deliver to one central, locked-down S3 bucket in a dedicated log-archive account.

### 3.3 How many trails?

- First copy of management events per region: **free**. Additional trails bill per event.
- Common pattern: one org-wide multi-region trail (security team) + optionally one team-scoped trail with data events for a specific workload.

---

## 4. Stage 3 — Deliver: Where the Data Lands

CloudTrail doesn't persist logs in its own console — it formats **JSON** and pipes it out:

### 4.1 Amazon S3 — the permanent home

- Compressed `.json.gz` packages delivered **every ~5 minutes** to your secure bucket.
- Key structure: `AWSLogs/<account-id>/CloudTrail/<region>/<yyyy>/<mm>/<dd>/…`
- This is your **immutable audit trail** for long-term compliance / SOC storage.

### 4.2 Amazon CloudWatch Logs — real-time alerting

- Stream incoming events to a CloudWatch Log Group.
- Layer **metric filters + alarms** to fire **immediately** on forbidden API calls (root account usage, security-group changes, CloudTrail being turned off). See §9.

### 4.3 Amazon EventBridge — event-driven automation

- Every management event also flows to EventBridge in near-real time — build rules like *"on `AuthorizeSecurityGroupIngress` with port 22 open to 0.0.0.0/0 → trigger Lambda to revert it."*

---

## 5. Stage 4 — Secure: Log Hardening & Integrity

CloudTrail logs are the definitive legal proof of what happened during a breach — which makes them a **primary target** for attackers covering their tracks. Harden the pipeline:

| Control | What it does |
|---|---|
| **Log File Integrity Validation** | CloudTrail generates a cryptographic **hash digest file every hour**. Verify with the CLI (`validate-logs`): if anyone modifies or deletes a log row in S3, the cryptographic check fails immediately. |
| **KMS encryption (SSE-KMS with CMK)** | Encrypts log files at rest with a customer-managed key — even if bucket permissions are misconfigured, unauthorized users can't read the content. |
| **S3 Object Lock (Compliance Mode)** | WORM — Write Once, Read Many. **Nobody, including root**, can delete records until the legal retention period expires. |
| **Restrictive bucket policy** | Only the CloudTrail service principal may write; only the security team may read. Enable S3 Block Public Access. |
| **MFA Delete + versioning** | Extra guard on the audit bucket itself. |
| **Organization trail** | Members can't switch logging off (see §3.2). |
| **Alert on `StopLogging`** | The very first thing many attackers do is disable CloudTrail — alarm on `StopLogging`, `DeleteTrail`, and `UpdateTrail` events (see §9). |

---

## 6. Stage 5 — Analyze: Querying Audit Logs

Raw JSON in S3 is unreadable at scale. Three analysis paths:

### Method A — CloudTrail Event History (console)

Native console search over the last **90 days** of management events. Good for quick lookups: *"filter by username `admin-praveen` — what did they delete in the last 2 hours?"* Limited filters, one attribute at a time.

### Method B — Amazon Athena (the production standard)

Run standard **SQL directly against the compressed JSON in S3** — no data movement, pay per TB scanned.

```sql
-- Who deleted a DynamoDB table?
SELECT
    eventTime,
    userIdentity.arn        AS user_arn,
    sourceIPAddress,
    requestParameters
FROM cloudtrail_logs_prod
WHERE eventName = 'DeleteTable'
  AND eventTime > '2026-07-01T00:00:00Z'
ORDER BY eventTime DESC;
```

> **Performance tip:** create the Athena table with **partition projection** on region/date — otherwise every query scans the whole bucket history and costs balloon.

### Method C — CloudTrail Lake (see §8)

Managed, SQL-queryable event data store — no S3/Athena plumbing.

---

## 7. Anatomy of a CloudTrail Event

Learn to read these five fields first — they answer 95% of investigations:

```json
{
  "eventTime": "2026-07-28T14:23:11Z",              // WHEN
  "eventSource": "iam.amazonaws.com",               // which service
  "eventName": "CreateUser",                        // WHAT action
  "awsRegion": "ap-south-1",
  "sourceIPAddress": "203.0.113.42",                // FROM WHERE
  "userAgent": "aws-cli/2.15.0",
  "userIdentity": {                                 // WHO
    "type": "IAMUser",
    "arn": "arn:aws:iam::111122223333:user/tharun",
    "accountId": "111122223333",
    "sessionContext": { "mfaAuthenticated": "true" }
  },
  "requestParameters": { "userName": "Alice" },     // inputs
  "responseElements": { "user": { "arn": "..." } }, // outputs (null on read-only calls)
  "errorCode": "AccessDenied",                      // present ONLY on failed calls
  "readOnly": false,
  "eventType": "AwsApiCall",
  "recipientAccountId": "111122223333"
}
```

**Investigation shortcuts:**
- `errorCode: AccessDenied` bursts = someone probing permissions.
- `userIdentity.type: Root` = root account usage — should almost never appear.
- `sourceIPAddress` ending in `.amazonaws.com` = an AWS service made the call on your behalf.
- `userIdentity.sessionContext.sessionIssuer` reveals the **role** behind temporary credentials.

---

## 8. CloudTrail Lake

A modern, fully-managed alternative to the S3+Athena pipeline:

- **Event Data Stores** with retention up to **10 years** (extendable pricing tiers).
- **SQL queries directly in the CloudTrail console** — no Glue tables, no partitions to manage.
- Can ingest **non-AWS sources** (custom audit events from your own apps) and events from outside integrations.
- **Trade-off:** simpler operations, but ingestion+storage pricing differs from raw S3 — S3+Athena stays cheaper at very large volume; Lake wins on time-to-query and multi-account simplicity.

```sql
-- Lake query example (console)
SELECT eventTime, userIdentity.arn, eventName, sourceIPAddress
FROM <event-data-store-id>
WHERE eventName = 'ConsoleLogin' AND eventTime > '2026-07-01 00:00:00'
ORDER BY eventTime DESC
```

---

## 9. Real-Time Security Alerting Pattern

The CIS AWS Foundations Benchmark codifies the must-have alarms. Pipeline: **CloudTrail → CloudWatch Logs → metric filter → alarm → SNS.**

Minimum viable alert set:

| Alert on | Filter pattern targets | Why |
|---|---|---|
| Root account usage | `userIdentity.type = "Root"` | Root should be dormant |
| Console login without MFA | `ConsoleLogin` + `MFAUsed != "Yes"` | Credential-theft indicator |
| CloudTrail tampering | `StopLogging`, `DeleteTrail`, `UpdateTrail` | Attackers hide tracks first |
| IAM policy changes | `Put*Policy`, `Attach*Policy`, `Create*Policy` | Privilege escalation |
| Security-group changes | `AuthorizeSecurityGroupIngress/Egress` | Perimeter drift |
| Unauthorized calls | `errorCode = "AccessDenied*"` | Permission probing |
| KMS key deletion | `ScheduleKeyDeletion`, `DisableKey` | Data-destruction prep |
| S3 policy changes | `PutBucketPolicy`, `PutBucketAcl` | Data-exfil setup |

(Full CLI implementation in [hands-on-labs.md](hands-on-labs.md) Lab 4.)

---

## 10. CloudWatch vs CloudTrail Summary

| Feature | Amazon CloudWatch | AWS CloudTrail |
|---|---|---|
| **Primary focus** | Resource performance & health | Identity actions & API auditing |
| **Answers** | "Is my CPU pegged?" / "Why is the app throwing 500s?" | "Who changed the Security Group rules at midnight?" |
| **Core ingestion** | System metrics, application logs, memory stats | JSON audit records of AWS API executions |
| **Automation** | Triggers Auto Scaling and SNS pages | Triggers alerts on unauthorized activity / compliance drift |
| **Retention model** | Metrics auto-tier (up to 15 months); logs per policy | 90 days free (Event History); unlimited via S3/Lake |
| **Better together** | Hosts CloudTrail's real-time stream, filters, alarms | Feeds CloudWatch the security-event dimension |

---

## 11. Step-by-Step Configuration Guide

Production setup sequence (full walkthroughs in [hands-on-labs.md](hands-on-labs.md)):

1. **Create a dedicated, private S3 audit bucket** (Block Public Access ON, versioning ON) with the CloudTrail service bucket policy.
2. **Create a multi-region trail** with global service events and log file validation enabled.
3. **Encrypt with a customer-managed KMS key**; grant CloudTrail encrypt rights and the security team decrypt rights.
4. **Enable Object Lock (Compliance mode)** on a fresh bucket if you have regulatory WORM requirements.
5. **Stream to CloudWatch Logs** (create the IAM role CloudTrail assumes) and set retention (e.g., 90 days — S3 remains the long-term archive).
6. **Deploy the CIS alarm set** (§9) via metric filters + SNS.
7. **Enable Insights events** (call-rate + error-rate) on the trail.
8. **Selectively enable data events** only for sensitive buckets/functions using advanced event selectors.
9. **Stand up Athena** with partition projection (or a CloudTrail Lake event data store).
10. **Convert to an Organization trail** from the management account; deliver to a log-archive account.
11. **Quarterly:** run `validate-logs` integrity checks and rehearse one forensic query end-to-end.

---

## 12. Where to Use It — Target Use Cases

| Scenario | CloudTrail capability |
|---|---|
| "Who deleted the production S3 bucket?" | Event History / Athena on `DeleteBucket` |
| "Alert me the second anyone uses root" | CW Logs stream + metric filter + alarm |
| "Prove to auditors nobody tampered with logs" | Log file integrity validation + Object Lock |
| "Track who reads objects in the PII bucket" | S3 data events with advanced event selectors |
| "Detect crypto-mining / API abuse bursts" | Insights events (call-rate anomalies) |
| "One audit pipeline for 40 accounts" | Organization trail → log-archive account |
| "Auto-revert risky security-group changes" | EventBridge rule on the event → Lambda remediation |
| "7-year queryable compliance archive" | CloudTrail Lake (10-yr retention) or S3 lifecycle + Athena |
| "Find every action from a suspicious IP" | Athena `WHERE sourceIPAddress = '...'` |

---

**Next:** open the [commands cheatsheet](commands-cheatsheet.md) · run the [hands-on labs](hands-on-labs.md) · bookmark [troubleshooting](troubleshooting.md).
