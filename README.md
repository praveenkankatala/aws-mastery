# AWS DevOps Learning Repository

A structured, hands-on record of learning AWS services and DevOps practices —
each service documented consistently: architecture, deep-dive, configuration
guide, and real-world use cases.

## How this repo is organized

- Numbered top-level folders define the learning path (`01-foundations` → `09-real-world-projects`)
- Every individual service folder contains the same five files:
  `README.md`, `hands-on-labs.md`, `commands-cheatsheet.md`, `troubleshooting.md`, plus `configs/` and `diagrams/`
- `templates/service-notes-template.md` is the master template — copy it when adding a new service

## Progress Tracker

| Module | Topic | Status |
|---|---|---|
| 00 | Getting Started | 🚧 |
| 01 | Foundations (Global Infra, IAM, Billing) | 🚧 |
| 02 | Compute (EC2, Lambda, ECS, EKS, Beanstalk) | 🚧 |
| 03 | Storage (S3, EBS, EFS) | 🚧 |
| 04 | Networking (VPC, Route53, CloudFront, ELB) | 🚧 |
| 05 | Databases (RDS, DynamoDB, ElastiCache) | 🚧 |
| 06 | Security & Identity (IAM, KMS, Secrets Manager) | 🚧 |
| 07 | DevOps & CI/CD (CodePipeline, Terraform, CloudFormation) | 🚧 |
| 08 | Monitoring & Observability (CloudWatch, CloudTrail) | 🚧 |
| 09 | Real-World Projects | 🚧 |

Legend: 🚧 In Progress · ✅ Complete · ⏳ Not Started

## How to add a new service

```bash
./scaffold.sh                     # re-run any time — safe, won't overwrite existing files
```
Then edit `scaffold_service` calls in `scaffold.sh`, or manually copy
`templates/service-notes-template.md` into a new folder.
