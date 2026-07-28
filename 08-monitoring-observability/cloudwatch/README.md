# AWS Observability & Governance — CloudWatch + CloudTrail

A complete, practical, end-to-end learning repository covering **Amazon CloudWatch** (monitoring & observability) and **AWS CloudTrail** (auditing & governance) — from core concepts to production-grade hands-on labs, CLI cheatsheets, and real-world troubleshooting.

> **Who is this for?** DevOps engineers, cloud engineers, SREs, and anyone preparing for AWS interviews or building production monitoring/audit pipelines.

---

## The One-Line Difference

| | Amazon CloudWatch | AWS CloudTrail |
|---|---|---|
| **What** | Performance & health monitoring (observability) | API activity recording (governance & audit) |
| **Answers** | *"Is my CPU pegged? Why is the app throwing 500s?"* | *"WHO changed the Security Group rules at midnight?"* |
| **Ingests** | Metrics, logs, traces, events | JSON audit records of every AWS API call |
| **Acts** | Auto Scaling, SNS alerts, Lambda remediation | Security alerts on unauthorized/compliance drift |

**Think of it this way:** CloudWatch is the *dashboard and alarm system* of your car; CloudTrail is the *dashcam* recording who drove it, when, and where.

---

## Repository Structure

```
aws-observability/
├── README.md                          <- You are here
├── cloudwatch/
│   ├── README.md                      <- What / Why / How, architecture, features, config guide
│   ├── commands-cheatsheet.md         <- All CloudWatch CLI commands, organized
│   ├── hands-on-labs.md               <- 10 labs: from first alarm to trace-log correlation
│   └── troubleshooting.md             <- Common errors, causes, fixes
└── cloudtrail/
    ├── README.md                      <- What / Why / How, event types, trail design, security
    ├── commands-cheatsheet.md         <- All CloudTrail CLI commands, organized
    ├── hands-on-labs.md               <- 8 labs: from first trail to Athena forensics
    └── troubleshooting.md             <- Common errors, causes, fixes
```

---

## How They Work Together (End-to-End Flow)

```
                        ┌──────────────────────────────────────────────┐
                        │                YOUR AWS ACCOUNT              │
                        │                                              │
   User/App API call ──►│  IAM / EC2 / S3 / Lambda / EKS / RDS ...     │
                        └───────────┬──────────────────┬───────────────┘
                                    │                  │
                     (performance   │                  │  (every API call:
                      data: metrics,│                  │   who / what / when /
                      logs, traces) │                  │   from where)
                                    ▼                  ▼
                        ┌───────────────────┐  ┌───────────────────┐
                        │  Amazon           │  │  AWS CloudTrail   │
                        │  CloudWatch       │◄─┤  (streams events  │
                        │                   │  │   to CW Logs)     │
                        │ Metrics · Logs ·  │  └─────────┬─────────┘
                        │ Alarms · X-Ray ·  │            │
                        │ Dashboards        │            ▼
                        └───────┬───────────┘  ┌───────────────────┐
                                │              │  S3 (immutable,   │
                                ▼              │  encrypted audit  │
                    ┌───────────────────────┐  │  archive)         │
                    │ ACT: SNS alerts,      │  └─────────┬─────────┘
                    │ Auto Scaling, Lambda  │            ▼
                    │ auto-remediation      │  ┌───────────────────┐
                    └───────────────────────┘  │ Athena SQL        │
                                               │ forensics         │
                                               └───────────────────┘
```

**The complete loop in one incident:**
1. An engineer runs `aws iam create-user --user-name Alice`.
2. **CloudTrail** captures the API call (identity, source IP, timestamp) → delivers to S3 + streams to CloudWatch Logs.
3. A **CloudWatch metric filter** spots the `CreateUser` event → **alarm** fires → **SNS** notifies the security team on Slack.
4. Meanwhile, **CloudWatch metrics** show a CPU spike on production; the on-call engineer uses **Logs Insights** + **X-Ray traces** to find root cause in minutes.

---

## Prerequisites

- An AWS account (Free Tier is sufficient for most labs)
- AWS CLI v2 installed and configured (`aws configure`)
- IAM permissions: `CloudWatchFullAccess`, `AWSCloudTrail_FullAccess`, plus S3/SNS/Lambda as needed per lab
- Basic familiarity with EC2, S3, Lambda, and IAM
- (Optional) `kubectl` + an EKS cluster for the EKS monitoring lab

---

## Learning Path (Recommended Order)

| Step | Doc | What you'll learn |
|---|---|---|
| 1 | [cloudwatch/README.md](cloudwatch/README.md) | The 4-stage pipeline: Collect → Monitor → Act → Analyze |
| 2 | [cloudwatch/hands-on-labs.md](cloudwatch/hands-on-labs.md) | First alarm → custom metrics → EMF → dashboards → X-Ray |
| 3 | [cloudtrail/README.md](cloudtrail/README.md) | Event types, multi-region trails, log hardening |
| 4 | [cloudtrail/hands-on-labs.md](cloudtrail/hands-on-labs.md) | First trail → integrity validation → Athena forensics |
| 5 | Both `troubleshooting.md` files | Production war stories and fixes |
| 6 | Both `commands-cheatsheet.md` files | Keep open in a second tab, always |

---

## License & Contributions

Free to use for learning and interview prep. PRs welcome — especially new labs, updated pricing notes, and additional troubleshooting scenarios.
