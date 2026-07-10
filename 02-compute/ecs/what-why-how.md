# 🧭 What / Why / How — The AWS ECS Concept Handbook

> Every core ECS concept explained in three questions:
> **What** it is · **Why** it exists · **How** it works in practice.
>
> Use this document to build a rock-solid mental model before diving into hands-on labs.

---

## 📋 Table of Contents

1. [Amazon ECS](#1-amazon-ecs)
2. [Task Definition](#2-task-definition)
3. [Task](#3-task)
4. [Service](#4-service)
5. [Cluster](#5-cluster)
6. [AWS Fargate](#6-aws-fargate)
7. [EC2 Launch Type](#7-ec2-launch-type)
8. [ECS Anywhere](#8-ecs-anywhere)
9. [ECR (Elastic Container Registry)](#9-ecr-elastic-container-registry)
10. [Application Load Balancer + Target Group](#10-application-load-balancer--target-group)
11. [Rolling Update Deployment](#11-rolling-update-deployment)
12. [Blue/Green Deployment](#12-bluegreen-deployment)
13. [ECS Service Auto Scaling](#13-ecs-service-auto-scaling)
14. [CloudWatch Alarms for ECS](#14-cloudwatch-alarms-for-ecs)
15. [Task Execution Role vs Task Role](#15-task-execution-role-vs-task-role)
16. [Docker Image Versioning](#16-docker-image-versioning)
17. [ECR Tag Immutability](#17-ecr-tag-immutability)
18. [awsvpc Networking Mode](#18-awsvpc-networking-mode)
19. [Service Connect](#19-service-connect)
20. [Deployment Circuit Breaker](#20-deployment-circuit-breaker)
21. [Fargate Spot](#21-fargate-spot)
22. [Capacity Providers](#22-capacity-providers)
23. [Ephemeral Storage vs EFS vs EBS](#23-ephemeral-storage-vs-efs-vs-ebs)
24. [Secrets Injection](#24-secrets-injection)
25. [CI/CD Pipeline for ECS](#25-cicd-pipeline-for-ecs)
26. [ECS vs EKS](#26-ecs-vs-eks)
27. [ECS vs Fargate](#27-ecs-vs-fargate)

---

## 1. Amazon ECS

**What:** A fully-managed container orchestration service from AWS that runs, scales, and manages Docker containers across a cluster of compute resources.

**Why:** Running Docker in production requires scheduling (where to place containers), health checking, scaling, load-balancer integration, and rolling updates. Without orchestration, you'd have to build all of this yourself — a massive engineering burden. ECS handles it natively on AWS with no cluster-management fee and deep integration with IAM, CloudWatch, ALB, and ECR.

**How:**
- You package your app as a Docker image and push it to ECR.
- You write a **Task Definition** (JSON blueprint) describing how to run it.
- You create a **Service** that maintains N healthy Tasks.
- ECS places them on **Fargate** (serverless) or **EC2** (self-managed) compute.
- The **Application Load Balancer** distributes traffic to healthy Tasks.
- **CloudWatch Alarms** trigger auto-scaling on load spikes.

---

## 2. Task Definition

**What:** A JSON document that specifies **how** to run one or more containers as a unit — image URI, CPU, memory, network, ports, environment variables, IAM roles, storage, logging.

**Why:** ECS needs a repeatable, immutable blueprint to launch containers consistently. Without it, every deployment would be ad-hoc. Task Definitions provide version history (revisions), enabling instant rollbacks and audit trails.

**How:**
- You define it via JSON, Terraform, CloudFormation, or the AWS Console.
- Each save creates a **new revision** (`my-app:1`, `my-app:2`, `my-app:3`).
- Revisions are **immutable** — you never edit an old one; you register a new one.
- A running Task references a specific revision, so you can compare/rollback easily.

Example fields:
```json
{
  "family": "my-app",
  "cpu": "512",
  "memory": "1024",
  "networkMode": "awsvpc",
  "containerDefinitions": [ /* image, ports, env, secrets */ ]
}
```

---

## 3. Task

**What:** The **live, running instantiation** of a Task Definition. If the Task Definition says *"run one Nginx container"*, the Task is the Nginx container currently executing.

**Why:** ECS needs an atomic unit to schedule, monitor, restart, and destroy. Tasks are the smallest deployable unit — you can run a one-off Task for a batch job, or many identical Tasks managed by a Service.

**How:**
- ECS reads the Task Definition and picks a placement target (Fargate slot or EC2 instance).
- It downloads the Docker image from ECR.
- It provisions network (ENI + private IP), storage, IAM credentials.
- It starts the container(s), monitors health, and reports status to the ECS control plane.
- If the Task crashes or the host fails, the parent Service replaces it.

---

## 4. Service

**What:** A long-running controller that maintains a **desired count** of healthy Tasks and integrates them with a load balancer.

**Why:** In production, you need self-healing (dead tasks → replaced automatically), consistent capacity, integrated load balancing, and coordinated rolling updates. A raw Task doesn't do any of this — a Service does.

**How:**
- You specify `desiredCount: 3` (for example).
- The Service watches all its Tasks constantly.
- If a Task dies → new one launched.
- If you update the Task Definition → Service rolls out new revision (rolling / Blue-Green).
- The Service auto-registers new Task IPs with the ALB Target Group and deregisters dying ones.

---

## 5. Cluster

**What:** A logical grouping of resources (compute + services + tasks) that acts as a namespace boundary.

**Why:** You need isolation between environments (dev/staging/prod), teams, or workload types. Clusters provide that boundary and are the root context for IAM policies, monitoring, and billing views.

**How:**
- You create clusters like `prod-cluster`, `staging-cluster`, `data-processing-cluster`.
- A cluster can host Fargate tasks, EC2 tasks, or both (via Capacity Providers).
- Clusters don't cost anything on their own — you only pay for the compute inside them.

---

## 6. AWS Fargate

**What:** A serverless compute engine that runs containers **without you managing any servers**. You specify CPU + memory, and AWS provisions an isolated micro-VM to run your container.

**Why:** Managing EC2 instances (patching, autoscaling groups, cluster utilization) is significant operational overhead. Fargate eliminates it — you pay per-second for exactly what you use, and AWS handles the underlying infrastructure entirely.

**How:**
- You set `launchType: FARGATE` in your Service.
- ECS forwards the Task Definition to Fargate.
- Fargate provisions a dedicated micro-VM (hypervisor-isolated, single-tenant).
- Your container runs; you get billed per second.
- When the Task stops, the micro-VM is destroyed — billing stops.

---

## 7. EC2 Launch Type

**What:** The original ECS model where you run your own fleet of EC2 instances, install the ECS agent on them, and ECS schedules containers onto them.

**Why:** You get complete control — custom OS, SSH access, specialized hardware (GPUs, high-memory instances), and lower per-resource cost at high utilization. Ideal for large predictable workloads or ML.

**How:**
- Launch EC2 instances from the **ECS-optimized AMI** (has the agent preinstalled).
- Put them in an **Auto Scaling Group** so the cluster grows/shrinks with demand.
- Register them to a cluster.
- ECS places Tasks onto instances based on available CPU/memory.
- You own patching, OS updates, and instance-level monitoring.

---

## 8. ECS Anywhere

**What:** A feature that lets you run ECS Tasks on your own hardware, VMs, or other clouds — completely outside AWS data centers.

**Why:** Some workloads must stay on-premises for data-residency, low-latency, or hybrid-cloud reasons. ECS Anywhere gives you a single control plane for containers whether they run in AWS or your own basement server room.

**How:**
- Install the open-source **AWS SSM agent + ECS agent** on your server.
- Register the server as an **external instance** to an ECS cluster.
- ECS treats it like a regular container instance — you can deploy Tasks to it.
- The control plane (scheduling, monitoring) lives in AWS; the compute runs on your hardware.

---

## 9. ECR (Elastic Container Registry)

**What:** AWS's managed Docker container registry — a private repository where you store, tag, and pull Docker images.

**Why:** Docker Hub has rate limits and is public by default. Enterprise apps need private, secure, IAM-integrated image storage. ECR is deeply integrated with ECS/EKS, supports image scanning, and enforces tag immutability.

**How:**
- Create an ECR repository (`aws ecr create-repository`).
- Authenticate Docker with ECR (`aws ecr get-login-password`).
- Build & push (`docker push <account>.dkr.ecr.<region>.amazonaws.com/my-app:tag`).
- ECS Tasks pull from ECR using the **Task Execution Role** permissions.

---

## 10. Application Load Balancer + Target Group

**What:** An **ALB** distributes HTTP/HTTPS traffic across healthy backend targets. A **Target Group** is the list of backend IPs (Tasks) the ALB routes to.

**Why:** You need to distribute traffic across multiple containers for scalability and high availability, plus a single stable DNS endpoint that abstracts away the ever-changing task IPs.

**How:**
- Create the ALB with a public DNS name.
- Configure a **Listener** on port 80/443.
- Attach a **Target Group** that will hold your Task IPs.
- Configure **Health Checks** (e.g., HTTP GET `/health` expecting 200).
- ECS Service auto-registers new Task IPs into the Target Group as tasks launch and deregisters them on scale-in.

---

## 11. Rolling Update Deployment

**What:** The default ECS deployment strategy — gradually replaces old tasks with new tasks while keeping the service available.

**Why:** You need zero-downtime deployments. Rolling updates ensure at least some healthy tasks are always serving traffic while new revisions come online.

**How:**
- Controlled by `minimumHealthyPercent` (e.g., 100% — never drop below current capacity) and `maximumPercent` (e.g., 200% — can double capacity during the swap).
- ECS launches new tasks alongside old ones.
- ALB health-checks new tasks; they receive traffic once healthy.
- Old tasks are drained and stopped one by one.
- No downtime; rollback requires a redeploy of the previous revision.

---

## 12. Blue/Green Deployment

**What:** A deployment strategy powered by AWS CodeDeploy that stands up a completely new task pool (Green) alongside the existing pool (Blue), then shifts traffic once Green is verified.

**Why:** Rolling updates are safe but slow to roll back — you'd have to redeploy the old image. Blue/Green lets you shift 100% traffic back to Blue instantly if something goes wrong, because Blue is still running (during the bake period).

**How:**
- Set the service's `deploymentController` to `CODE_DEPLOY`.
- CodeDeploy provisions Green tasks in a second Target Group.
- Health checks + optional Lambda test scripts verify Green.
- Traffic shifts (Linear / Canary / All-at-Once) via ALB listener rewrites.
- After a **bake period**, Blue tasks are terminated. If any CloudWatch alarm trips → instant automatic rollback to Blue.

---

## 13. ECS Service Auto Scaling

**What:** A feature that automatically adjusts the number of Tasks in a Service based on metrics like CPU, memory, or request count.

**Why:** Traffic isn't constant. Manually scaling is slow and error-prone. Auto Scaling ensures you have exactly the capacity you need — no more (cost), no less (performance).

**How:**
- Register the ECS Service as a **scalable target** in Application Auto Scaling.
- Attach a scaling policy: **Target Tracking** (recommended), **Step Scaling**, or **Scheduled**.
- The policy references a **CloudWatch metric** (e.g., `ECSServiceAverageCPUUtilization`).
- When the metric crosses the threshold, ECS adjusts `desiredCount` up or down.
- New tasks are registered with the ALB automatically; old ones are drained.

---

## 14. CloudWatch Alarms for ECS

**What:** Automated triggers that fire when a metric crosses a threshold (e.g., CPU > 70% for 3 minutes).

**Why:** They're the nervous system of auto-scaling and Blue/Green rollback. Without alarms, ECS wouldn't know when to scale out, scale in, or roll back a bad deployment.

**How:**
- Metrics come from CloudWatch (CPU, memory), ALB (request count, 5XX errors, latency), or your custom app.
- Alarms have three states: `OK`, `ALARM`, `INSUFFICIENT_DATA`.
- Alarms trigger actions: Auto Scaling policies, SNS notifications, CodeDeploy rollbacks, Lambda functions.

---

## 15. Task Execution Role vs Task Role

**What:**
- **Task Execution Role** = permissions the **ECS agent** needs to *start* the container.
- **Task Role** = permissions your **application code** needs *while running*.

**Why:** Separation of concerns. The agent's needs (pull image, write logs) are static and cluster-wide. Your app's needs (S3 access, DynamoDB queries) are workload-specific. Mixing them would violate least-privilege.

**How:**
- **Task Execution Role** permissions:
  - `ecr:*` (pull image)
  - `logs:*` (send logs to CloudWatch)
  - `secretsmanager:GetSecretValue` (fetch secrets for injection)
- **Task Role** permissions:
  - Whatever your app needs — `s3:GetObject`, `dynamodb:PutItem`, `sqs:SendMessage`, etc.
- Both are specified in the Task Definition (`executionRoleArn` and `taskRoleArn`).

---

## 16. Docker Image Versioning

**What:** The practice of assigning a unique, meaningful tag to every Docker image built by your CI pipeline.

**Why:** Without proper versioning, you cannot trace which code is running, cannot roll back reliably, and cannot audit deployments. Using `:latest` is a rollback nightmare.

**How:**

**Option 1 (recommended): Git Commit SHA**
```bash
IMAGE_TAG=$(git rev-parse --short HEAD)
docker build -t my-app:$IMAGE_TAG .
```

**Option 2: Semantic Versioning via Git Tags**
```bash
git tag v1.4.0
docker build -t my-app:v1.4.0 .
```

Both create a permanent 1-to-1 mapping between the running container and the exact code that produced it.

---

## 17. ECR Tag Immutability

**What:** An ECR setting that prevents anyone from overwriting an existing image tag.

**Why:** Prevents accidental (or malicious) overwriting of a released image. Guarantees that `my-app:v1.0.0` today is *bit-for-bit identical* to `my-app:v1.0.0` a year from now.

**How:**
- Enable in the ECR repo settings: `Image Tag Immutability: IMMUTABLE`.
- Any `docker push` attempting to reuse an existing tag will be rejected with an error.

---

## 18. `awsvpc` Networking Mode

**What:** The ECS networking mode where each Task gets its own **Elastic Network Interface (ENI)** and a private VPC IP address — behaving like a mini EC2 instance.

**Why:** Provides true network isolation, first-class VPC integration, and per-task security groups. All Fargate Tasks are required to use `awsvpc`.

**How:**
- Set `networkMode: awsvpc` in the Task Definition.
- Specify VPC subnet(s) and security group(s) in the Service's `networkConfiguration`.
- ECS creates an ENI in your VPC per Task at launch.
- The Task can be reached at its private IP just like any EC2 instance.
- If in a private subnet, needs a **NAT Gateway** or **VPC Endpoints** for outbound (ECR, logs, secrets).

---

## 19. Service Connect

**What:** A modern ECS feature that simplifies **service-to-service communication** via automatic sidecar proxies.

**Why:** Microservices need to call each other. Traditionally you'd use ALBs (expensive, latency) or Cloud Map (complex DNS setup). Service Connect makes it as easy as calling `http://backend-service` with automatic TLS, retries, and load balancing.

**How:**
- Enable Service Connect on the ECS Service and specify a namespace.
- ECS injects an **Envoy proxy sidecar** into each Task.
- Services register a friendly DNS name in the namespace.
- Other services in the namespace call `http://<service-name>:<port>` — the sidecar routes automatically.
- Get automatic TLS, retries, timeouts, and CloudWatch traffic metrics.

---

## 20. Deployment Circuit Breaker

**What:** A safety feature that automatically detects failing deployments and rolls back to the last known-good Task Definition revision.

**Why:** Without it, a bad deployment can loop endlessly (launch → crash → relaunch → crash), consuming resources and blocking future deployments. Circuit Breaker stops the bleeding.

**How:**
- Enable in the Service configuration:
```json
"deploymentConfiguration": {
  "deploymentCircuitBreaker": { "enable": true, "rollback": true }
}
```
- If N consecutive Task launches fail health checks, ECS **stops the deployment**.
- If `rollback: true`, ECS automatically redeploys the previous revision.

---

## 21. Fargate Spot

**What:** A discounted (up to 70% off) Fargate compute option that uses AWS's excess capacity.

**Why:** Massive cost savings on interruption-tolerant workloads. If your app is stateless and retry-safe (batch jobs, dev environments, queue workers), you can slash bills dramatically.

**How:**
- Set your Capacity Provider strategy to include `FARGATE_SPOT`.
- Tasks run at the same performance as regular Fargate.
- AWS may reclaim capacity with a **2-minute warning** signal (SIGTERM to the container).
- Design your app to shut down gracefully or hand off work when it sees the signal.

---

## 22. Capacity Providers

**What:** An ECS abstraction for **how compute is provisioned** for your Services.

**Why:** In real production, you rarely want 100% Fargate or 100% Spot — you want a mix. Capacity Providers let you declaratively split workloads (e.g., 30% Fargate + 70% Fargate Spot) for cost + reliability balance.

**How:**
- Define a strategy like:
```
FARGATE:      weight=1, base=2   (always at least 2 tasks on reliable Fargate)
FARGATE_SPOT: weight=4           (rest of the fleet uses cheap Spot)
```
- ECS respects the base + weight rules when placing Tasks.
- Also supports mixing EC2 instance types for advanced strategies.

---

## 23. Ephemeral Storage vs EFS vs EBS

**What:** The three main storage options for ECS Tasks.

**Why:** Containers are stateless by default. If your workload needs state — user uploads, config files, or high-IOPS databases — you need explicit storage options.

**How:**

| Type | Persistence | Sharing | Best For |
|---|---|---|---|
| **Ephemeral** | Wiped on task stop | Not shared | Temp files, downloads, scratch space (20 GB free) |
| **EFS** | Durable | Multi-task | WordPress uploads, shared configs, ML datasets |
| **EBS** | Durable | Single-task | Databases, high-IOPS block storage |

---

## 24. Secrets Injection

**What:** The pattern of pulling sensitive values (DB passwords, API keys) into containers at runtime, without hardcoding them.

**Why:** Never bake secrets into Docker images (they'd be in ECR forever, visible to anyone with pull access) or in Git (obvious leak). Secrets Manager / SSM Parameter Store provide encrypted, rotatable storage.

**How:**
- Store the secret in **AWS Secrets Manager** or **SSM Parameter Store**.
- Reference its ARN in the Task Definition:
```json
"secrets": [
  { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:region:acct:secret:name" }
]
```
- The **Task Execution Role** must have `secretsmanager:GetSecretValue` permission.
- At Task startup, ECS fetches the secret and injects it as an environment variable — never logged, never in Git.

---

## 25. CI/CD Pipeline for ECS

**What:** An automated workflow that goes from `git push` → new Docker image → deployed to ECS with zero manual steps.

**Why:** Manual deployments are slow, error-prone, and don't scale. A proper pipeline gives you audit trails, rollback capability, and confidence to deploy multiple times a day.

**How (5 phases):**

1. **Trigger** — Push to `main` or Git tag creation.
2. **Build** — CI runner builds Docker image using `Dockerfile`.
3. **Push** — Image tagged with Git SHA and pushed to ECR.
4. **Render** — CI updates Task Definition JSON with new image URI.
5. **Deploy** — Registers new Task Def revision, updates Service, waits for stability.

Tools: **GitHub Actions**, **GitLab CI**, **AWS CodePipeline + CodeBuild**, **Jenkins**.

---

## 26. ECS vs EKS

**What:** Both are AWS container orchestrators — but ECS is AWS-native and EKS is managed Kubernetes.

**Why the choice matters:**

| Criterion | ECS wins if... | EKS wins if... |
|---|---|---|
| **Simplicity** | Small team, AWS-only, want low ops | Have Kubernetes experts |
| **Portability** | 100% AWS forever | Multi-cloud strategy needed |
| **Ecosystem** | AWS-native tools are enough | Need Helm, ArgoCD, Istio, Prometheus |
| **Cost** | No cluster fee | $73/mo cluster fee is acceptable |

**How they differ mechanically:**
- ECS uses **Task Definitions** (JSON); EKS uses **Manifests** (YAML).
- ECS is a proprietary API; EKS speaks Kubernetes API.
- ECS integrates natively with AWS IAM; EKS requires **IRSA + OIDC provider**.

---

## 27. ECS vs Fargate

**What:** This is the most common misconception. **They are not alternatives.** ECS is the *orchestrator* (control plane); Fargate is the *compute* (data plane).

**Why the confusion:** AWS marketing sometimes treats them separately, and people ask *"should I use ECS or Fargate?"* — but the correct question is *"should I run ECS **on** Fargate or **on** EC2?"*

**How they relate:**

```
┌─────────────────────────┐
│  Amazon ECS             │   ← Orchestrator (decides WHAT runs, WHEN)
│  (control plane)        │
└───────────┬─────────────┘
            │ launches tasks on
            ▼
┌─────────────────────────┐
│  Compute Backend        │   ← Provides CPU + RAM
│  • Fargate (serverless) │
│  • EC2 (self-managed)   │
│  • ECS Anywhere (hybrid)│
└─────────────────────────┘
```

**Analogy:** ECS is the project manager. Fargate is the on-demand worker. Neither works alone.

---

## 🎓 Final Mental Model

If you can answer these questions confidently, you understand ECS:

1. **What runs my container?** — A Task (an instance of a Task Definition).
2. **What keeps it healthy?** — A Service (monitors + replaces dead Tasks).
3. **Where does it physically run?** — On Fargate (serverless) or EC2 (self-managed).
4. **How does traffic reach it?** — An ALB routes via a Target Group of Task IPs.
5. **How do updates deploy?** — Rolling (default) or Blue/Green (CodeDeploy).
6. **How does it scale?** — CloudWatch Alarms → Application Auto Scaling → `desiredCount` up/down.
7. **How does it stay secure?** — Task Role (app permissions) + Task Execution Role (agent permissions) + secrets from Secrets Manager.
8. **How does CI/CD work?** — Build → push to ECR → new Task Def revision → update Service.

---

**Next steps:** Move on to **[commands-cheatsheet.md](commands-cheatsheet.md)** for CLI mastery, then **[hands-on-labs.md](hands-on-labs.md)** to build it yourself, and **[troubleshooting.md](troubleshooting.md)** for when things break.
