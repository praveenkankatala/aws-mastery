# AWS EKS End-to-End Learning Guide

> A complete, hands-on learning repository for **Amazon Elastic Kubernetes Service (EKS)** — from control-plane fundamentals to production-grade Terraform deployments, Karpenter autoscaling, and day-two operations.

[![AWS](https://img.shields.io/badge/AWS-EKS-orange)](https://aws.amazon.com/eks/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.30+-blue)](https://kubernetes.io/)
[![Terraform](https://img.shields.io/badge/Terraform-1.5+-purple)](https://www.terraform.io/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## Table of Contents

1. [About This Repository](#about-this-repository)
2. [What You Will Learn](#what-you-will-learn)
3. [Repository Structure](#repository-structure)
4. [Prerequisites](#prerequisites)
5. [EKS Architecture Overview](#eks-architecture-overview)
6. [Core Concepts at a Glance](#core-concepts-at-a-glance)
7. [Compute Options Compared](#compute-options-compared)
8. [Networking Deep Dive](#networking-deep-dive)
9. [Security & Identity Model](#security--identity-model)
10. [Advanced Features](#advanced-features)
11. [Autoscaling Landscape](#autoscaling-landscape)
12. [Learning Path](#learning-path)
13. [Reference Links](#reference-links)

---

## About This Repository

This repository is a structured, practical study guide that covers Amazon EKS end-to-end — the same content a DevOps or platform engineer would need to design, deploy, secure, and operate a production Kubernetes cluster on AWS. It combines conceptual explanations, working Terraform blueprints, CLI cheat sheets, hands-on labs, and a troubleshooting playbook.

The material is organized so you can either read it linearly (as a tutorial) or dip into individual files as reference material during real work.

---

## What You Will Learn

By working through this repository, you will be able to:

- Explain the split responsibilities between the AWS-managed control plane and your worker data plane.
- Choose the right compute model (Managed Node Groups, Self-Managed Nodes, Fargate, or EKS Auto Mode) for a given workload.
- Design a VPC that supports pod-level IP allocation via the Amazon VPC CNI without exhausting IP space.
- Configure authentication using EKS Access Entries (the modern replacement for `aws-auth` ConfigMap) and authorization via Kubernetes RBAC.
- Give pods scoped AWS permissions using **IRSA** (OIDC-based) or the newer **EKS Pod Identity**.
- Deploy the AWS Load Balancer Controller so that Kubernetes Ingress objects create real ALBs.
- Deploy and tune **Karpenter** for just-in-time, cost-optimized node provisioning.
- Upgrade cluster versions safely using EKS Upgrade Insights and the 7-day rollback window.
- Diagnose and fix the most common EKS failures — pending pods, IP exhaustion, IAM auth errors, ALB provisioning issues, and more.

---

## Repository Structure

```
eks-learning/
├── README.md                  # You are here — overview, architecture, learning path
├── commands-cheatsheet.md     # AWS CLI, eksctl, kubectl, Helm, Terraform commands
├── hands-on-labs.md           # 12 progressive labs from cluster creation to Karpenter
└── troubleshooting.md         # Common errors, root causes, and fixes
```

---

## Prerequisites

### Knowledge

- **Kubernetes basics** — pods, deployments, services, namespaces, kubectl.
- **AWS fundamentals** — VPC, subnets, IAM roles/policies, EC2, IAM instance profiles.
- **Linux CLI comfort** — bash, environment variables, SSH.
- **Networking basics** — CIDR blocks, routing, security groups, NAT gateways.

### Tooling (install these first)

| Tool | Version | Purpose |
|------|---------|---------|
| AWS CLI | v2.x | Authenticate and interact with AWS services |
| kubectl | Matches your EKS minor version (± 1) | Talk to the Kubernetes API |
| eksctl | Latest stable | Simplest CLI to create/manage EKS clusters |
| Helm | 3.x | Install controllers and add-ons |
| Terraform | 1.5+ | Infrastructure as Code deployments |
| jq | Any | Parse JSON output from CLIs |

### AWS Account Setup

- An AWS account with programmatic access (Access Key ID / Secret, or SSO).
- IAM permissions to create EKS clusters, VPCs, EC2 instances, IAM roles, and Load Balancers. For labs, an admin-level role is easiest; for production, scope down using least privilege.
- A default region set (`aws configure` → e.g., `us-east-1` or `ap-south-1`).
- Sufficient service quotas for EIPs, VPCs, and EC2 instances in your chosen region.

---

## EKS Architecture Overview

The following diagram shows how AWS-managed components, your VPC, worker nodes, and traffic flow all fit together.

```
                                  ┌────────────────────────────────────────────┐
                                  │        AWS-MANAGED CONTROL PLANE VPC       │
                                  │  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
                                  │  │ API srv  │ │ scheduler│ │controller│    │
                                  │  │  AZ-a    │ │  AZ-b    │ │  AZ-c    │    │
                                  │  └────┬─────┘ └────┬─────┘ └────┬─────┘    │
                                  │       └───────────┴──etcd──────┘           │
                                  └───────────────────┬────────────────────────┘
                                                      │  (ENIs projected into
                                                      │   your VPC for kubelet
                                                      │   ↔ apiserver traffic)
   External User                                      │
        │                                             │
        │ HTTPS                                       │
        ▼                                             ▼
 ┌───────────────┐         ┌──────────────────────────────────────────────────┐
 │ Route 53 / DNS│────────►│                  YOUR VPC (10.0.0.0/16)          │
 └───────────────┘         │                                                  │
                           │  ┌────────── Public Subnets (3 AZs) ─────────┐   │
                           │  │  ALB / NLB   │ NAT GW-a │ NAT GW-b │ ...  │   │
                           │  └───────┬──────────────────────────────────┘    │
                           │          │                                       │
                           │  ┌───────▼────── Private Subnets (3 AZs) ────┐   │
                           │  │                                            │  │
                           │  │  ┌────Node──┐  ┌────Node──┐  ┌────Node──┐  │  │
                           │  │  │ Pod A    │  │ Pod C    │  │ Pod E    │  │  │
                           │  │  │ Pod B    │  │ Pod D    │  │ Pod F    │  │  │
                           │  │  │  (IPs    │  │  from    │  │  VPC     │  │  │
                           │  │  │   CIDR)  │  │          │  │          │  │  │
                           │  │  └──────────┘  └──────────┘  └──────────┘  │  │
                           │  │                                            │  │
                           │  └────────────────────────────────────────────┘  │
                           └──────────────────────────────────────────────────┘
```

### End-to-end request path

1. External user hits `app.example.com` → Route 53 resolves it to an ALB DNS name.
2. The ALB (public subnets) was dynamically provisioned by the **AWS Load Balancer Controller** running inside the cluster, based on an `Ingress` object.
3. The ALB forwards traffic to pod IPs directly (IP target type), routing into private subnets.
4. The pod handles the request. If it needs AWS access (S3, DynamoDB, Secrets Manager), it uses temporary credentials from an **IRSA** or **Pod Identity** association — never the node's IAM role.
5. Response flows back out through the ALB.

---

## Core Concepts at a Glance

### 1. The Control Plane (AWS-Managed)

The control plane is the "brain" of your Kubernetes cluster. On EKS, AWS runs and scales it for you.

- **Single-tenant, HA by default** — EKS provisions the control plane in an AWS-owned VPC across **at least three Availability Zones**.
- **Components managed by AWS**:
  - `kube-apiserver` — the entry point for every kubectl and controller call, scaled horizontally by load.
  - `etcd` — the distributed key-value store holding all cluster state. AWS handles scaling, snapshots, and backup.
  - `kube-scheduler` — decides which node each pod runs on.
  - `kube-controller-manager` — reconciles cluster state (deployments, replicasets, etc.).
- You never SSH into these. You interact only via the API endpoint.

### 2. The Data Plane (Where Your Workloads Run)

Worker nodes host your pods. EKS offers three compute models — see the [table below](#compute-options-compared).

### 3. Networking (VPC + CNI)

- EKS uses the **Amazon VPC CNI plugin** by default.
- Every pod gets a **real, routable private IP** from your VPC subnet range (no overlay network).
- This eliminates the encapsulation overhead of Flannel/Calico overlays and gives native AWS network performance.
- Trade-off: pods consume VPC IPs directly, so subnet sizing matters (see [Networking Deep Dive](#networking-deep-dive)).

### 4. Security & Identity

- **Authentication (who you are)** → AWS IAM.
- **Authorization (what you can do)** → Kubernetes RBAC.
- The mapping from IAM identity to Kubernetes permissions is done via **EKS Access Entries** (modern) or the legacy `aws-auth` ConfigMap.
- Pod-level AWS credentials come from **IRSA** or **EKS Pod Identity** — never from the node's instance profile in production.

---

## Compute Options Compared

| Feature | Managed Node Groups (MNG) | Self-Managed Nodes | AWS Fargate | EKS Auto Mode |
|---------|---------------------------|--------------------|-------------|---------------|
| **Who manages EC2?** | AWS provisions & lifecycles, you set instance types | You (custom AMIs, bootstrap scripts) | Serverless — no EC2 at all | AWS picks instance types automatically |
| **OS patching** | AWS-driven with drain | You do it | N/A | Fully automated |
| **Instance flexibility** | Fixed types in the group | Full control | No choice — Fargate profile only | AWS picks optimal size per workload |
| **Scaling backend** | EC2 Auto Scaling Group | You wire it up | Per-pod | Karpenter under the hood |
| **Pricing model** | EC2 pricing | EC2 pricing | Per-pod vCPU & memory | EC2 pricing + AutoMode fee |
| **Best for** | Standard workloads, mixed instance types | Custom AMIs, GPU nodes, unusual OS | Isolated multi-tenant, spiky jobs | Teams that want minimal ops overhead |

---

## Networking Deep Dive

### VPC CIDR planning

A recommended baseline for a production cluster:

| Subnet Type | Count | Recommended Size | Purpose |
|-------------|-------|------------------|---------|
| Public | 3 (one per AZ) | `/20` or `/22` | ALB/NLB, NAT Gateways, bastions |
| Private | 3 (one per AZ) | `/20` minimum | Worker nodes AND pod IPs |

**Why `/20` for private subnets?** Because the VPC CNI hands each pod its own VPC IP, a busy node with 30+ pods can burn through IPs quickly. A `/20` gives you ~4,091 usable IPs per AZ, which is a safer floor.

### Cluster endpoint access modes

You control how the `kube-apiserver` is reachable:

| Mode | Description | Use case |
|------|-------------|----------|
| **Public only** | API accessible from the internet, optionally restricted by CIDR whitelist | Simple dev clusters |
| **Public + Private** | Nodes talk over private endpoint; kubectl from outside uses public endpoint | Most common production setup |
| **Private only** | API only reachable from within the VPC (or via VPN/Direct Connect) | Hardened environments |

### Add-ons every cluster needs

- **VPC CNI** — pod networking (installed by default).
- **CoreDNS** — in-cluster DNS.
- **kube-proxy** — service networking.
- **EBS CSI driver** — persistent volumes backed by EBS.
- **AWS Load Balancer Controller** — provisions ALBs/NLBs from `Ingress` and `Service` objects.

---

## Security & Identity Model

### Authentication vs. Authorization

```
      kubectl call
          │
          ▼
   ┌──────────────┐        ┌────────────────────┐
   │  AWS IAM     │        │   Kubernetes RBAC  │
   │ (Who are you)│───►    │ (What can you do)  │
   └──────────────┘        └────────────────────┘
          │                          │
          └─── Access Entries map ───┘
```

### EKS Access Entries (modern approach)

Replaces the legacy `aws-auth` ConfigMap. Maps an IAM principal ARN directly to Kubernetes permissions via the EKS API:

```yaml
accessConfig:
  authenticationMode: API             # or API_AND_CONFIG_MAP during migration
  accessEntries:
    - principalArn: arn:aws:iam::123456789012:role/DevOpsAdminRole
      type: STANDARD
      accessPolicies:
        - policyArn: arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy
          accessScope:
            type: cluster
```

### Pod-level AWS access

Two ways to give a pod scoped AWS credentials:

| Method | Mechanism | When to use |
|--------|-----------|-------------|
| **IRSA (IAM Roles for Service Accounts)** | Cluster OIDC provider + IAM role trust policy tied to a ServiceAccount | Existing clusters, cross-account patterns |
| **EKS Pod Identity** | Native EKS association object, no OIDC juggling | New clusters — simpler and preferred |

**Never** give the node IAM role broad AWS permissions. Every pod on that node would inherit them.

---

## Advanced Features

### EKS Auto Mode

A managed compute mode that automates node provisioning, patching, storage, networking add-ons, and core Kubernetes upgrades — with a single toggle. Internally it uses Karpenter to pick the cheapest instance that fits your pods.

### Upgrade Insights & Rollback

- **Upgrade Insights** — a pre-flight report on API deprecations and version compatibility issues before you upgrade.
- **Rollback window** — you can roll back a minor version upgrade for up to **7 days** if something breaks in production.

### Control-plane logging

You can stream `api`, `audit`, `authenticator`, `controllerManager`, and `scheduler` logs to CloudWatch Logs — critical for troubleshooting auth and scheduling issues.

---

## Autoscaling Landscape

EKS clusters typically use **multiple** autoscalers together:

| Scaler | Layer | Scales what |
|--------|-------|-------------|
| **HPA** (Horizontal Pod Autoscaler) | Kubernetes native | Number of pod replicas based on CPU/mem/custom metrics |
| **VPA** (Vertical Pod Autoscaler) | Kubernetes native | Right-sizes a pod's CPU/mem requests |
| **Cluster Autoscaler (CA)** | Node level | Scales existing EC2 Auto Scaling Groups |
| **Karpenter** | Node level | Provisions EC2 instances directly via the EC2 Fleet API, bypassing ASGs |

### Karpenter in one sentence

Karpenter watches unschedulable pods, calculates the exact best EC2 instance to fit them, calls the EC2 API directly, and continuously consolidates underutilized nodes to save money.

**Karpenter vs. Cluster Autoscaler**

| Feature | Cluster Autoscaler | Karpenter |
|---------|--------------------|-----------|
| Abstraction | Bound to ASGs | Direct EC2 Fleet API |
| Scale-up speed | Minutes | Seconds |
| Instance selection | Only what's in the ASG template | Any instance from the entire AWS catalog |
| Downscaling | Basic idle threshold | Simulated rescheduling and consolidation |
| Spot support | Manual | First-class, mixed with on-demand |

---

## Learning Path

Follow the files in this order for a structured journey:

1. **README.md** (this file) — build a mental model.
2. **[commands-cheatsheet.md](commands-cheatsheet.md)** — memorize the essential CLI muscle memory.
3. **[hands-on-labs.md](hands-on-labs.md)** — build a real cluster, deploy apps, install Karpenter, do an upgrade.
4. **[troubleshooting.md](troubleshooting.md)** — reference during and after the labs when things break.

### Suggested weekly plan

| Week | Focus | Deliverable |
|------|-------|-------------|
| 1 | Concepts + CLI setup | eksctl-created cluster with sample app |
| 2 | Networking + Ingress | ALB Controller + working `Ingress` |
| 3 | Identity + Storage | IRSA/Pod Identity + EBS-backed StatefulSet |
| 4 | Autoscaling | Karpenter + HPA driving real load |
| 5 | Terraform IaC | Full cluster reproduced from code |
| 6 | Upgrade + Observability | Cluster upgraded with Insights, CloudWatch logs on |

---

## Reference Links

- [Amazon EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
- [eksctl Documentation](https://eksctl.io/)
- [Karpenter Documentation](https://karpenter.sh/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Terraform AWS EKS Module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)

---

## License

MIT — free to use, share, and adapt.

## Contributing

Issues and PRs welcome. If you find outdated commands or better patterns, please open a PR.
