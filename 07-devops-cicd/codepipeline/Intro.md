# AWS CI/CD with the Code Suite — A Practical, End-to-End Guide

> A complete, hands-on learning reference for building continuous integration and continuous delivery (CI/CD) pipelines on AWS using **CodeCommit**, **CodeBuild**, **CodeDeploy**, and **CodePipeline** — explained in a simple **What → Why → How** format.

<p align="left">
  <img alt="AWS" src="https://img.shields.io/badge/AWS-Developer%20Tools-FF9900?logo=amazonaws&logoColor=white">
  <img alt="CI/CD" src="https://img.shields.io/badge/CI%2FCD-Pipeline-2088FF">
  <img alt="Level" src="https://img.shields.io/badge/Level-Beginner%20→%20Advanced-blueviolet">
  <img alt="IaC" src="https://img.shields.io/badge/Style-What%20%7C%20Why%20%7C%20How-success">
</p>

---

## 📚 Documentation Map

This guide is intentionally split into focused files so nothing becomes overwhelming:

| File | What's inside |
|------|---------------|
| **README.md** (you are here) | Big-picture concepts, architecture, service deep-dives, configuration, and where to use each service |
| **[commands-cheatsheet.md](./commands-cheatsheet.md)** | Every important AWS CLI command, grouped by service, ready to copy-paste |
| **[hands-on-labs.md](./hands-on-labs.md)** | Build a working pipeline from an empty account to a live deployment, step by step |
| **[troubleshooting.md](./troubleshooting.md)** | Real error messages, root causes, and fixes |

---

## Table of Contents

1. [What Is CI/CD? (The Foundation)](#1-what-is-cicd-the-foundation)
2. [Meet the Four Services](#2-meet-the-four-services)
3. [High-Level Architecture & Service Flow](#3-high-level-architecture--service-flow)
4. [Prerequisites](#4-prerequisites)
5. [Core Features & Deep-Dive (per service)](#5-core-features--deep-dive-per-service)
6. [Supporting Concepts You Must Know](#6-supporting-concepts-you-must-know)
7. [Step-by-Step Configuration & Implementation Guide](#7-step-by-step-configuration--implementation-guide)
8. [How to Use & Where to Use (Target Use Cases)](#8-how-to-use--where-to-use-target-use-cases)
9. [Cost, Security & Best Practices](#9-cost-security--best-practices)
10. [Important 2025/2026 Service Status Notes](#10-important-20252026-service-status-notes)
11. [Glossary](#11-glossary)

---

## 1. What Is CI/CD? (The Foundation)

**What.** CI/CD is a way of delivering software through automation instead of manual steps.

- **CI — Continuous Integration:** every code change is automatically built and tested the moment it lands in the repository.
- **CD — Continuous Delivery / Deployment:** every change that passes the tests is automatically packaged and released to an environment (staging or production).

**Why.** Manual builds and deployments are slow, inconsistent, and error-prone. Someone forgets a step, copies the wrong file, or deploys from the wrong branch. CI/CD replaces "hope and memory" with a **repeatable, auditable pipeline** so that shipping code becomes boring and safe.

**How (on AWS).** AWS provides four managed services that map cleanly onto the CI/CD stages. You wire them together once, and every future commit flows through automatically:

```
 Write code  →  Store it  →  Build & test it  →  Deploy it  →  Repeat
                (CodeCommit)   (CodeBuild)      (CodeDeploy)
                         orchestrated by  (CodePipeline)
```

---

## 2. Meet the Four Services

| Service | The one-line "What" | CI/CD stage it owns |
|---------|--------------------|---------------------|
| **AWS CodeCommit** | A fully managed, private Git repository (like a self-hosted GitHub inside AWS) | **Source** |
| **AWS CodeBuild** | A fully managed build server that compiles, tests, and packages your code | **Build / Test** |
| **AWS CodeDeploy** | An automated deployment engine that pushes your app to EC2, Lambda, ECS, or on-prem servers | **Deploy** |
| **AWS CodePipeline** | The orchestrator that connects Source → Build → Deploy into one automated workflow | **Orchestration (the pipeline itself)** |

**Analogy — a factory production line:**
- **CodeCommit** is the *warehouse* holding raw materials (your source code).
- **CodeBuild** is the *assembly machine* that turns raw materials into a finished product (a build artifact).
- **CodeDeploy** is the *delivery truck* that ships the finished product to customers (your servers).
- **CodePipeline** is the *conveyor belt and control room* that moves items between stations automatically and stops the line if something breaks.

---

## 3. High-Level Architecture & Service Flow

### 3.1 The end-to-end flow (text diagram)

```
                                   ┌─────────────────────────────────────────────┐
                                   │              DEVELOPER                        │
                                   │   git push  (or PR merge to main branch)     │
                                   └───────────────────┬─────────────────────────┘
                                                       │
                                                       ▼
        ┌──────────────────────────────────────────────────────────────────────────────┐
        │  ①  SOURCE STAGE                                                                │
        │  ┌────────────────────────┐                                                    │
        │  │   AWS CodeCommit       │   (or GitHub / Bitbucket / S3 as a source)         │
        │  │   Git repository       │   A change event triggers the pipeline via         │
        │  └────────────────────────┘   Amazon EventBridge.                              │
        └───────────────────────────────────┬──────────────────────────────────────────┘
                                             │  source artifact  (zipped code → S3)
                                             ▼
        ┌──────────────────────────────────────────────────────────────────────────────┐
        │  ②  BUILD STAGE                                                                 │
        │  ┌────────────────────────┐                                                    │
        │  │   AWS CodeBuild        │   Reads buildspec.yml                               │
        │  │   ephemeral container  │   → installs deps, runs tests, compiles,           │
        │  └────────────────────────┘   → produces a build artifact                      │
        └───────────────────────────────────┬──────────────────────────────────────────┘
                                             │  build artifact  (jar/zip/image → S3/ECR)
                                             ▼
        ┌──────────────────────────────────────────────────────────────────────────────┐
        │  ③  (OPTIONAL) MANUAL APPROVAL                                                  │
        │      A human clicks "Approve" before production. Sends SNS notification.        │
        └───────────────────────────────────┬──────────────────────────────────────────┘
                                             ▼
        ┌──────────────────────────────────────────────────────────────────────────────┐
        │  ④  DEPLOY STAGE                                                                │
        │  ┌────────────────────────┐   Reads appspec.yml                                 │
        │  │   AWS CodeDeploy       │   → in-place OR blue/green deployment               │
        │  │                        │   → runs lifecycle hooks, health checks             │
        │  └───────────┬────────────┘   → auto-rollback on failure                        │
        │              ▼                                                                  │
        │   EC2 / Auto Scaling Group  •  AWS Lambda  •  Amazon ECS  •  On-prem servers    │
        └──────────────────────────────────────────────────────────────────────────────┘

        ══════════════════════════════════════════════════════════════════════════════
          AWS CodePipeline wraps ALL of the above stages and moves artifacts between them.
          Artifacts are stored in an S3 "artifact store" bucket. IAM roles grant each
          service exactly the permissions it needs. CloudWatch/SNS provide observability.
        ══════════════════════════════════════════════════════════════════════════════
```

### 3.2 Who talks to whom

```
CodePipeline ──(reads source)──▶ CodeCommit
CodePipeline ──(starts build)──▶ CodeBuild ──(reads)──▶ buildspec.yml
CodePipeline ──(starts deploy)─▶ CodeDeploy ──(reads)─▶ appspec.yml
     │
     ├──(stores artifacts)──▶ Amazon S3 (artifact store)
     ├──(assumes)───────────▶ IAM service roles
     ├──(triggered by)──────▶ Amazon EventBridge (on commit)
     └──(notifies)──────────▶ Amazon SNS / CloudWatch
```

### 3.3 Mental model in one sentence

> **CodePipeline** is the brain; **CodeCommit**, **CodeBuild**, and **CodeDeploy** are the hands; **S3**, **IAM**, **EventBridge**, and **CloudWatch** are the nervous system that lets them work together safely.

---

## 4. Prerequisites

**What you need before starting the labs:**

| Requirement | Details |
|-------------|---------|
| **AWS account** | With billing enabled. Most of this fits in the AWS Free Tier for learning. |
| **IAM user/role** | Admin or a scoped role that can create CodeCommit, CodeBuild, CodeDeploy, CodePipeline, IAM roles, S3 buckets, and EC2. |
| **AWS CLI v2** | Installed and configured (`aws configure`). See the [cheat sheet](./commands-cheatsheet.md#0-setup--identity). |
| **Git** | Installed locally (`git --version`). |
| **Git credential helper** | For CodeCommit over HTTPS (the AWS CLI provides one — covered in the labs). |
| **Basic knowledge** | Comfortable with Git basics (clone, commit, push) and a terminal. |
| **A sample app** | The labs provide a tiny Node.js / static web app; any language works. |

**Region note:** Pick one Region (e.g., `us-east-1`) and stay in it for the whole tutorial to avoid cross-region confusion.

---

## 5. Core Features & Deep-Dive (per service)

### 5.1 AWS CodeCommit — Source Control

**What.** A fully managed, secure, private Git repository hosted in your AWS account. It speaks standard Git — you clone, commit, branch, and push exactly as with GitHub.

**Why use it.**
- **No servers to manage** — AWS runs and scales it for you.
- **Native IAM integration** — access is controlled by AWS IAM policies, not a separate user system.
- **Encryption everywhere** — data is encrypted in transit (HTTPS/SSH) and at rest (AWS KMS) automatically.
- **VPC/private access** — you can keep code entirely inside your private network.
- **Tight AWS integration** — triggers CodePipeline directly, no third-party webhooks needed.

**Key features / concepts:**
- **Repositories** — the Git repos themselves.
- **Branches, commits, tags** — standard Git objects.
- **Pull requests & approval rules** — code review inside AWS, with approval-rule templates.
- **Triggers & notifications** — fire on push/PR events to Lambda, SNS, or EventBridge.
- **Two access methods:** HTTPS (Git credential helper or Git credentials) and SSH (key pairs).

**Good to know:** CodeCommit is Git — everything you know about Git still applies. It just lives in AWS.

---

### 5.2 AWS CodeBuild — Build & Test

**What.** A fully managed build service that runs your build commands inside a fresh, disposable container, then hands back the output (an *artifact*).

**Why use it.**
- **No build servers** — no Jenkins box to patch, no idle EC2 costs. You pay per build-minute.
- **Elastic & parallel** — scales to many concurrent builds automatically.
- **Reproducible** — every build runs in a clean environment defined by an image, so "works on my machine" disappears.
- **Pluggable** — works standalone or as a stage inside CodePipeline.

**Key features / concepts:**
- **Build project** — the configuration: source, environment, and what to run.
- **`buildspec.yml`** — a YAML file (in your repo root, or inline) that defines the build in phases:
  - `install` → set up runtimes/tools
  - `pre_build` → login to registries, prep
  - `build` → compile, run tests
  - `post_build` → package, push images
  - `artifacts` → which files to keep
  - `cache` / `reports` / `env` → caching, test reports, variables
- **Environment** — the compute type (small/medium/large), OS, and Docker image (managed images for Node, Python, Java, Go, Docker, etc., or a custom image).
- **Environment variables** — plaintext, or secure via SSM Parameter Store / Secrets Manager.
- **Artifacts** — build outputs sent to S3 (or passed to the next pipeline stage).
- **Caching** — speed up builds by caching dependencies (local or S3).
- **Reports** — test and code-coverage reports viewable in the console.
- **Local builds** — you can run CodeBuild locally with the `codebuild_build.sh` agent for debugging.

**A minimal `buildspec.yml`:**

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - echo "Installing dependencies..."
      - npm ci
  pre_build:
    commands:
      - echo "Running linters..."
  build:
    commands:
      - echo "Running tests and build..."
      - npm test
      - npm run build
  post_build:
    commands:
      - echo "Build complete on `date`"

artifacts:
  files:
    - '**/*'
  base-directory: dist        # only ship the built output
```

---

### 5.3 AWS CodeDeploy — Deployment Automation

**What.** A service that automates *how* your application gets onto its target (EC2, Auto Scaling groups, Lambda, ECS, or on-prem), including health checks and automatic rollback.

**Why use it.**
- **Consistent releases** — the same deployment steps run every time.
- **Minimize downtime** — blue/green and rolling strategies keep the app available during release.
- **Safety nets** — automatic rollback on a failed health check or alarm.
- **Multi-platform** — one service handles servers, serverless, and containers.

**Key features / concepts:**
- **Application** — a logical container for your deployments, tied to a *compute platform*:
  - **EC2/On-Premises** — deploy to servers running the **CodeDeploy agent**.
  - **AWS Lambda** — shift traffic between function versions.
  - **Amazon ECS** — shift traffic between task sets.
- **Deployment group** — *where* to deploy (which EC2 tags, ASG, ECS service, etc.) plus alarms and rollback settings.
- **Deployment configuration** — *how fast* to deploy:
  - `AllAtOnce`, `HalfAtATime`, `OneAtATime` (EC2)
  - `Canary` (a % first, then the rest), `Linear` (equal % at intervals), `AllAtOnce` (Lambda/ECS)
- **`appspec.yml` / `appspec.yaml`** — tells CodeDeploy which files go where and which **lifecycle hooks** to run.
- **Lifecycle event hooks** (EC2/On-Prem): `ApplicationStop → DownloadBundle → BeforeInstall → Install → AfterInstall → ApplicationStart → ValidateService`. You attach scripts to these.
- **In-place vs. Blue/Green:**
  - **In-place** — updates the *existing* instances one batch at a time.
  - **Blue/Green** — spins up a *new* fleet (green), tests it, then flips traffic from old (blue) to green; instant rollback by flipping back.
- **CodeDeploy agent** — a small daemon on EC2/on-prem instances that pulls and runs deployments.

**A minimal `appspec.yml` (EC2/On-Premises):**

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html
hooks:
  BeforeInstall:
    - location: scripts/stop_server.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 300
      runas: root
  ValidateService:
    - location: scripts/health_check.sh
      timeout: 300
      runas: root
```

---

### 5.4 AWS CodePipeline — Orchestration

**What.** A fully managed workflow service that models your release process as **stages** and **actions**, and moves your code through them automatically.

**Why use it.**
- **One place to see everything** — a visual graph of Source → Build → Approve → Deploy.
- **Automatic triggering** — a commit kicks off the whole flow.
- **Fan-out & gates** — run actions in parallel, add manual approvals, split by environment.
- **Extensible** — integrates with CodeBuild, CodeDeploy, CloudFormation, ECS, Lambda, third-party actions (Jenkins, GitHub), and custom actions.

**Key features / concepts:**
- **Pipeline** — the whole workflow.
- **Stage** — a logical phase (Source, Build, Deploy). Stages run in order.
- **Action** — a task within a stage (e.g., "pull from CodeCommit", "run CodeBuild"). Actions in a stage can run in **sequence** (via run-order) or **parallel**.
- **Action categories** — `Source`, `Build`, `Test`, `Deploy`, `Approval`, `Invoke`.
- **Transitions** — the gates between stages; you can disable a transition to pause flow.
- **Artifacts** — inputs/outputs passed between actions, stored in the **S3 artifact store**.
- **Manual approval action** — pauses the pipeline for human sign-off (with optional SNS notification).
- **Pipeline types** — **V1** and **V2** (V2 adds parameters, richer triggers, and stage-level conditions).
- **Execution modes** — `SUPERSEDED` (default), `QUEUED`, `PARALLEL`.

---

## 6. Supporting Concepts You Must Know

These are the "glue" concepts that make the four services actually work together. Skipping them is the #1 cause of failed first pipelines.

### 6.1 IAM Roles (the most important supporting concept)

**What.** Every Code service assumes an **IAM service role** to act on your behalf.

- **CodePipeline service role** — lets the pipeline read the source, start builds, and start deployments.
- **CodeBuild service role** — lets the build read source, write logs to CloudWatch, and push artifacts to S3/ECR.
- **CodeDeploy service role** — lets CodeDeploy read tags, talk to Auto Scaling, and register/deregister instances.
- **EC2 instance profile** — lets target EC2 instances pull the deployment bundle from S3.

**Why it matters.** ~80% of "it doesn't work" moments are missing or too-narrow permissions. Get the roles right first.

**Trust vs. permissions:** a role has (1) a **trust policy** — *who* can assume it (e.g., `codebuild.amazonaws.com`) — and (2) **permission policies** — *what* it can do.

### 6.2 Artifacts & the Artifact Store

**What.** An **artifact** is a zipped bundle passed between stages (source code → build output → deployable package). CodePipeline stores them in an **S3 bucket** it owns per Region.

**Why.** Stages are decoupled — each reads its input artifact and writes an output artifact, so stages don't need to know about each other's internals.

### 6.3 buildspec.yml vs. appspec.yml (don't mix them up)

| File | Belongs to | Answers the question |
|------|-----------|----------------------|
| `buildspec.yml` | **CodeBuild** | "*How do I build and test?*" |
| `appspec.yml` | **CodeDeploy** | "*Where do the files go and what scripts run during deploy?*" |

### 6.4 Triggers (how a commit starts everything)

Modern pipelines are triggered by **Amazon EventBridge** rules watching your source repo. When you push, EventBridge fires an event that starts the pipeline. (Older pipelines used periodic polling — slower and not recommended.)

### 6.5 Notifications & Observability

- **Amazon SNS** — email/SMS/chat notifications on approvals, successes, failures.
- **CloudWatch Logs** — build logs and deployment logs.
- **CloudWatch Alarms** — can automatically roll back a CodeDeploy deployment.
- **AWS CloudTrail** — audit trail of every API call.

### 6.6 Encryption

Artifacts in S3 and repos in CodeCommit are encrypted with **AWS KMS**. You can use the AWS-managed key or your own customer-managed key (CMK) for tighter control and cross-account sharing.

### 6.7 Adjacent services worth knowing

- **AWS CodeArtifact** — managed package repository (npm, Maven, PyPI, NuGet). Great for private dependencies.
- **Amazon ECR** — private Docker image registry (where CodeBuild often pushes container images).
- **AWS CloudFormation / CDK** — Infrastructure as Code; frequently a *deploy* action inside CodePipeline.
- **Amazon CodeGuru** — automated code reviews and profiling.

---

## 7. Step-by-Step Configuration & Implementation Guide

> This is the high-level checklist. For fully worked, copy-paste steps see **[hands-on-labs.md](./hands-on-labs.md)**.

**Phase 1 — Prepare identity & source**
1. Configure the AWS CLI (`aws configure`).
2. Create IAM service roles for CodePipeline, CodeBuild, and CodeDeploy (or let the console create them).
3. Create a CodeCommit repository and push your sample app + `buildspec.yml` + `appspec.yml`.

**Phase 2 — Build**
4. Create a CodeBuild project pointing at the repo, choosing a managed image and the `buildspec.yml`.
5. Run a standalone build to confirm it's green **before** adding it to a pipeline.

**Phase 3 — Deploy target**
6. Launch an EC2 instance (with an instance profile) and install the **CodeDeploy agent**, OR set up a Lambda/ECS target.
7. Create a CodeDeploy **application** and **deployment group** (choose in-place or blue/green).

**Phase 4 — Orchestrate**
8. Create a CodePipeline: Source (CodeCommit) → Build (CodeBuild) → (optional Approval) → Deploy (CodeDeploy).
9. Push a commit and watch the pipeline run end-to-end.

**Phase 5 — Harden**
10. Add SNS notifications, CloudWatch alarms with auto-rollback, manual approval before prod, and least-privilege IAM.

---

## 8. How to Use & Where to Use (Target Use Cases)

### When each service shines

| Scenario | Best fit |
|----------|----------|
| You want private Git hosted inside AWS with IAM-based access | **CodeCommit** |
| You need to compile/test but hate managing Jenkins | **CodeBuild** |
| You must deploy to EC2/ASG with zero-downtime and rollback | **CodeDeploy** (blue/green) |
| You want the whole release automated and visualized | **CodePipeline** |
| You want serverless releases (Lambda) with canary traffic shifting | **CodeDeploy** (Lambda platform) + **CodePipeline** |
| You run containers on ECS and want safe rollouts | **CodeDeploy** (ECS blue/green) + **CodePipeline** |

### Real-world patterns

- **Classic 3-tier web app on EC2:** CodeCommit → CodeBuild → CodeDeploy (in-place or blue/green) via CodePipeline.
- **Serverless API:** CodeCommit → CodeBuild (SAM/CDK package) → CodeDeploy (Lambda canary) via CodePipeline.
- **Containerized microservice:** CodeCommit → CodeBuild (build & push image to ECR) → CodeDeploy (ECS blue/green) via CodePipeline.
- **Infra as Code:** CodeCommit → CodeBuild (validate/lint templates) → CloudFormation deploy action via CodePipeline.
- **Multi-account / multi-region:** central pipeline account deploys into separate dev/stage/prod accounts using cross-account roles.

### When *not* to use the Code suite

- Your org is standardized on **GitHub Actions / GitLab CI** and wants to stay there — you can still plug those into AWS, or use them end-to-end. (CodePipeline can use **GitHub as a source**.)
- You want an all-in-one IDE + project experience — note the CodeCatalyst status in [Section 10](#10-important-20252026-service-status-notes).

---

## 9. Cost, Security & Best Practices

### Cost (learn the pricing model, always check current rates)
- **CodeCommit** — priced per active user / storage / Git requests above a free tier.
- **CodeBuild** — pay per build-minute, by compute size. Small builds are cheap; caching reduces minutes.
- **CodeDeploy** — free to EC2/Lambda/ECS in your account; you pay only for the underlying resources (on-prem instances have a per-update charge).
- **CodePipeline** — priced per active pipeline per month (V1); V2 has an action-execution-based model. There's a free tier.
- **Hidden costs:** S3 artifact storage, KMS, CloudWatch logs, and the EC2/compute you deploy to.

> 💡 Always verify current pricing in the AWS Pricing pages — rates and models change.

### Security best practices
- **Least privilege IAM** — start narrow; add permissions only when a run fails for lack of them.
- **Separate roles per service** — don't reuse one giant role.
- **Encrypt with a CMK** — especially for cross-account artifacts.
- **Manual approval before production** — a human gate for high-risk stages.
- **Secrets in Secrets Manager / SSM Parameter Store** — never hard-code credentials in `buildspec.yml`.
- **Branch protection & PR approval rules** in CodeCommit.

### Reliability best practices
- Test CodeBuild standalone before wiring it into a pipeline.
- Enable **automatic rollback** on CodeDeploy via CloudWatch alarms.
- Prefer **blue/green** for production; keep **in-place** for lower environments.
- Cache dependencies in CodeBuild for faster, cheaper builds.
- Tag everything (env, owner, cost-center).

---

## 10. Important 2025/2026 Service Status Notes

> These are moving targets — always confirm on the official AWS pages before making architecture decisions.

- **AWS CodeCommit** was **closed to new customers on July 25, 2024**, then **reversed** and **returned to full General Availability on November 24, 2025**. New AWS accounts can again create CodeCommit repositories via the console, CLI, and API. If you ever hit a "cannot create repository" error, it's tied to this history — see [troubleshooting.md](./troubleshooting.md).
- **Amazon CodeCatalyst** (the all-in-one dev service) **closed to new customers on November 7, 2025**. AWS points new users toward **CodeBuild, CodePipeline, CodeDeploy, and CodeArtifact** instead — which is exactly this guide's stack.
- **CodeBuild, CodeDeploy, and CodePipeline** remain fully supported and actively developed.
- **Alternatives to CodeCommit** if you prefer: GitHub, GitLab, or Bitbucket — all can serve as a CodePipeline source.

---

## 11. Glossary

| Term | Meaning |
|------|---------|
| **Artifact** | A zipped bundle of files passed between pipeline stages (stored in S3). |
| **buildspec.yml** | CodeBuild's instruction file (phases + artifacts). |
| **appspec.yml** | CodeDeploy's instruction file (file mappings + lifecycle hooks). |
| **Deployment group** | The set of targets (EC2 tags/ASG/ECS/Lambda) a deployment goes to. |
| **Deployment configuration** | The speed/strategy of a deployment (canary, linear, all-at-once, etc.). |
| **In-place deployment** | Updates existing instances in batches. |
| **Blue/green deployment** | Deploys to a new fleet, then shifts traffic; instant rollback. |
| **Lifecycle hook** | A named point during deploy where you can run a script. |
| **Service role** | An IAM role a service assumes to act on your behalf. |
| **Instance profile** | The IAM role attached to an EC2 instance. |
| **Stage / Action** | A phase / a task within a phase in CodePipeline. |
| **Transition** | The gate between two pipeline stages. |
| **CodeDeploy agent** | A daemon on EC2/on-prem that executes deployments. |
| **EventBridge trigger** | The event rule that starts a pipeline on a commit. |
| **Artifact store** | The S3 bucket CodePipeline uses to hold artifacts. |

---

### ⭐ Next steps

1. Skim this README to build the mental model.
2. Keep **[commands-cheatsheet.md](./commands-cheatsheet.md)** open in a second tab.
3. Follow **[hands-on-labs.md](./hands-on-labs.md)** to build a real pipeline.
4. Bookmark **[troubleshooting.md](./troubleshooting.md)** — you'll need it.

> If this guide helped you, consider giving the repo a ⭐ on GitHub.

---

*Author's note: Built as a practical, portfolio-quality learning reference. Contributions and corrections welcome via pull request.*
