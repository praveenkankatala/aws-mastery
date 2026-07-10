# AWS EKS — Commands Cheat Sheet

> Every command you need for daily EKS operations, grouped by tool. Copy-paste-ready with brief explanations.

## Table of Contents

1. [AWS CLI — EKS](#1-aws-cli--eks)
2. [eksctl](#2-eksctl)
3. [kubectl (EKS-focused)](#3-kubectl-eks-focused)
4. [Helm](#4-helm)
5. [Terraform](#5-terraform)
6. [Karpenter](#6-karpenter)
7. [AWS Load Balancer Controller](#7-aws-load-balancer-controller)
8. [Networking & VPC CNI](#8-networking--vpc-cni)
9. [IAM, IRSA & Pod Identity](#9-iam-irsa--pod-identity)
10. [Storage (EBS / EFS CSI)](#10-storage-ebs--efs-csi)
11. [Observability](#11-observability)
12. [Troubleshooting quick commands](#12-troubleshooting-quick-commands)

---

## 1. AWS CLI — EKS

### Cluster lifecycle

```bash
# List clusters in the current region
aws eks list-clusters --region us-east-1

# Describe a cluster (endpoint, version, VPC, status)
aws eks describe-cluster --name my-cluster --region us-east-1

# Only get the cluster version
aws eks describe-cluster --name my-cluster --query "cluster.version" --output text

# Update your kubeconfig so kubectl can talk to the cluster
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Update kubeconfig with a specific IAM role
aws eks update-kubeconfig --name my-cluster --role-arn arn:aws:iam::123456789012:role/EKSAdminRole

# Delete a cluster (nodes/addons must be gone first)
aws eks delete-cluster --name my-cluster
```

### Node groups

```bash
# List managed node groups
aws eks list-nodegroups --cluster-name my-cluster

# Describe a node group
aws eks describe-nodegroup --cluster-name my-cluster --nodegroup-name core-nodes

# Update a node group's desired size
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name core-nodes \
  --scaling-config minSize=3,maxSize=10,desiredSize=5

# Delete a node group
aws eks delete-nodegroup --cluster-name my-cluster --nodegroup-name core-nodes
```

### Add-ons

```bash
# List add-ons available for a Kubernetes version
aws eks describe-addon-versions --kubernetes-version 1.30

# List installed add-ons
aws eks list-addons --cluster-name my-cluster

# Install an add-on (e.g., VPC CNI)
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.18.0-eksbuild.1

# Update an add-on
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name coredns \
  --addon-version v1.11.1-eksbuild.4

# Delete an add-on
aws eks delete-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver
```

### Access Entries (modern IAM ↔ RBAC mapping)

```bash
# List access entries
aws eks list-access-entries --cluster-name my-cluster

# Create an access entry for an IAM role
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/DevOpsRole \
  --type STANDARD

# Attach a managed access policy to that principal
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/DevOpsRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster

# Delete an access entry
aws eks delete-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/DevOpsRole
```

### EKS Pod Identity

```bash
# Create the Pod Identity Agent add-on (must be running before associations work)
aws eks create-addon --cluster-name my-cluster --addon-name eks-pod-identity-agent

# Associate a Kubernetes service account with an IAM role
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace default \
  --service-account my-app-sa \
  --role-arn arn:aws:iam::123456789012:role/MyAppRole

# List associations
aws eks list-pod-identity-associations --cluster-name my-cluster
```

### Cluster upgrades

```bash
# Update the control plane version
aws eks update-cluster-version --name my-cluster --kubernetes-version 1.30

# Update a managed node group (uses launch-template rolling update)
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name core-nodes \
  --kubernetes-version 1.30

# Track an in-progress update
aws eks describe-update --name my-cluster --update-id <update-id>
```

---

## 2. eksctl

### Create clusters (the fast path)

```bash
# Minimal cluster (uses default VPC — fine for demos, not production)
eksctl create cluster --name demo-cluster --region us-east-1

# With specific version, node group, and AZs
eksctl create cluster \
  --name prod-cluster \
  --region us-east-1 \
  --version 1.30 \
  --zones us-east-1a,us-east-1b,us-east-1c \
  --nodegroup-name workers \
  --node-type m6i.large \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 6 \
  --managed

# From a config file (recommended for reproducibility)
eksctl create cluster -f cluster-config.yaml

# Fargate-only cluster (no EC2 nodes)
eksctl create cluster --name fargate-cluster --fargate
```

### Sample cluster config

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: prod-cluster
  region: us-east-1
  version: "1.30"

vpc:
  cidr: 10.0.0.0/16
  clusterEndpoints:
    publicAccess: true
    privateAccess: true

managedNodeGroups:
  - name: core
    instanceType: m6i.large
    minSize: 3
    maxSize: 10
    desiredCapacity: 3
    privateNetworking: true
    volumeSize: 50

addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
  - name: aws-ebs-csi-driver

iam:
  withOIDC: true
```

### Node group management

```bash
# Add a new node group
eksctl create nodegroup -f nodegroup.yaml

# Scale a node group
eksctl scale nodegroup --cluster=my-cluster --name=workers --nodes=5

# Drain and delete a node group (safe migration path)
eksctl delete nodegroup --cluster=my-cluster --name=old-workers --drain
```

### Other useful eksctl commands

```bash
# List clusters
eksctl get cluster

# List node groups
eksctl get nodegroup --cluster my-cluster

# Enable OIDC provider (needed for IRSA)
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# Create a service account with IRSA
eksctl create iamserviceaccount \
  --name my-app-sa \
  --namespace default \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# Upgrade the control plane
eksctl upgrade cluster --name my-cluster --version 1.30 --approve

# Delete a cluster (cleans up VPC if eksctl created it)
eksctl delete cluster --name my-cluster
```

---

## 3. kubectl (EKS-focused)

### Context and cluster info

```bash
# Show current context
kubectl config current-context

# Switch context
kubectl config use-context arn:aws:eks:us-east-1:123456789012:cluster/my-cluster

# Cluster info
kubectl cluster-info

# API server version
kubectl version --short
```

### Nodes

```bash
# List nodes with their instance type and AZ
kubectl get nodes -o wide

# Show labels (useful for verifying Karpenter/nodegroup labels)
kubectl get nodes --show-labels

# Describe a node (capacity, allocatable, conditions, events)
kubectl describe node <node-name>

# Cordon (mark unschedulable)
kubectl cordon <node-name>

# Drain (evict pods and cordon)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

### Pods and workloads

```bash
# List pods across all namespaces
kubectl get pods -A

# Watch pod status live
kubectl get pods -n default -w

# Describe (events at the bottom are gold)
kubectl describe pod <pod-name> -n default

# Logs (follow with -f, last 100 lines with --tail=100)
kubectl logs <pod-name> -f --tail=100

# Logs from a previous crashed container
kubectl logs <pod-name> --previous

# Shell into a running pod
kubectl exec -it <pod-name> -- /bin/sh

# Apply a manifest
kubectl apply -f manifest.yaml

# Delete a resource
kubectl delete -f manifest.yaml
```

### Services and Ingress

```bash
# List services (LoadBalancer type will show the AWS ELB hostname)
kubectl get svc -A

# List ingress
kubectl get ingress -A

# Describe an ingress (see events for ALB provisioning progress)
kubectl describe ingress my-app -n default
```

### Deployments and scaling

```bash
# Scale a deployment
kubectl scale deployment my-app --replicas=5

# Rollout status
kubectl rollout status deployment/my-app

# Rollout history
kubectl rollout history deployment/my-app

# Rollback
kubectl rollout undo deployment/my-app
```

### HPA (Horizontal Pod Autoscaler)

```bash
# Create an HPA imperatively
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=70

# List HPAs and current metrics
kubectl get hpa

# Describe (shows current vs target metrics)
kubectl describe hpa my-app
```

### Resource usage (needs metrics-server)

```bash
kubectl top nodes
kubectl top pods -A
```

---

## 4. Helm

```bash
# Add a repo
helm repo add eks https://aws.github.io/eks-charts
helm repo add karpenter oci://public.ecr.aws/karpenter/karpenter
helm repo update

# Search a repo
helm search repo eks

# Install a chart
helm install aws-lb-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller

# List releases
helm list -A

# Upgrade a release
helm upgrade aws-lb-controller eks/aws-load-balancer-controller -n kube-system --reuse-values

# Roll back
helm rollback aws-lb-controller 1 -n kube-system

# Uninstall
helm uninstall aws-lb-controller -n kube-system

# Show computed values
helm get values aws-lb-controller -n kube-system
```

---

## 5. Terraform

```bash
# Init providers and modules
terraform init

# Format code
terraform fmt -recursive

# Validate syntax and references
terraform validate

# Preview changes
terraform plan -out tfplan

# Apply
terraform apply tfplan

# Show current state
terraform show

# List resources in state
terraform state list

# Import an existing resource
terraform import module.eks.aws_eks_cluster.this my-cluster

# Destroy everything (careful)
terraform destroy
```

### Common EKS-specific outputs to define

```hcl
output "cluster_name"     { value = module.eks.cluster_name }
output "cluster_endpoint" { value = module.eks.cluster_endpoint }
output "cluster_oidc_arn" { value = module.eks.oidc_provider_arn }
output "kubeconfig_cmd"   { value = "aws eks update-kubeconfig --name ${module.eks.cluster_name} --region ${var.region}" }
```

---

## 6. Karpenter

```bash
# Install via Helm (v1.x uses OCI registry)
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "1.0.0" \
  --namespace kube-system \
  --set "settings.clusterName=my-cluster" \
  --set "settings.interruptionQueue=my-cluster" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi

# Watch Karpenter events
kubectl logs -f -n kube-system -l app.kubernetes.io/name=karpenter

# List NodePools
kubectl get nodepool

# List EC2NodeClasses
kubectl get ec2nodeclass

# Nodes claimed by Karpenter
kubectl get nodeclaim

# Describe a claim to see instance type reasoning
kubectl describe nodeclaim <name>

# Trigger a scale-up by creating a pending workload
kubectl create deployment inflate --image=public.ecr.aws/eks-distro/kubernetes/pause:3.7
kubectl scale deployment inflate --replicas=20

# Force Karpenter to consolidate (remove workload)
kubectl scale deployment inflate --replicas=0
```

### Minimal NodePool + EC2NodeClass

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
      nodeClassRef:
        name: default
  limits:
    cpu: "1000"
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  role: KarpenterNodeRole-my-cluster
```

---

## 7. AWS Load Balancer Controller

```bash
# Download the IAM policy JSON
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

# Create the IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Create the IRSA service account
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::123456789012:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install the controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify it's running
kubectl get deployment -n kube-system aws-load-balancer-controller

# Watch controller logs while creating an Ingress
kubectl logs -f -n kube-system deploy/aws-load-balancer-controller
```

### Sample Ingress (ALB)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

---

## 8. Networking & VPC CNI

```bash
# Check VPC CNI version
kubectl describe daemonset aws-node -n kube-system | grep Image

# Enable prefix delegation (packs more pods per node)
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true

# Show ENI IP usage on a node
kubectl exec -n kube-system <aws-node-pod> -- /app/grpc-health-probe -addr=:50051

# Get pod-to-node IP mapping
kubectl get pods -A -o wide

# Show ENIs attached to a node's instance
aws ec2 describe-network-interfaces \
  --filters "Name=attachment.instance-id,Values=<instance-id>"
```

### Common environment variables (on `aws-node` DaemonSet)

| Variable | Effect |
|----------|--------|
| `WARM_IP_TARGET` | How many free IPs to keep warm per node |
| `MINIMUM_IP_TARGET` | Minimum IPs allocated at startup |
| `ENABLE_PREFIX_DELEGATION` | Assign `/28` prefixes instead of individual IPs |
| `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG` | Use custom `ENIConfig` for pods |

---

## 9. IAM, IRSA & Pod Identity

### IRSA (OIDC-based, classic)

```bash
# Get the cluster OIDC issuer URL
aws eks describe-cluster --name my-cluster --query "cluster.identity.oidc.issuer" --output text

# Create/verify the OIDC provider
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# Create an IAM role that trusts the cluster OIDC for a specific SA
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace default \
  --name s3-reader \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# Verify the service account has the role annotation
kubectl get sa s3-reader -n default -o yaml
```

### EKS Pod Identity (modern, preferred)

```bash
# Ensure the agent addon is installed
aws eks create-addon --cluster-name my-cluster --addon-name eks-pod-identity-agent

# Create the IAM role with the pods.eks.amazonaws.com trust
aws iam create-role \
  --role-name my-app-pod-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "pods.eks.amazonaws.com"},
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }]
  }'

aws iam attach-role-policy \
  --role-name my-app-pod-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Associate the role with a namespace + service account
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace default \
  --service-account my-app-sa \
  --role-arn arn:aws:iam::123456789012:role/my-app-pod-role
```

---

## 10. Storage (EBS / EFS CSI)

```bash
# Install the EBS CSI driver as a managed add-on
aws eks create-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver

# Create the default gp3 StorageClass
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
EOF

# Verify
kubectl get storageclass
kubectl get pv,pvc -A
```

### EFS CSI (for shared, multi-AZ storage)

```bash
# Install
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver -n kube-system

# Create the EFS file system in AWS first, then reference it in a PV:
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  accessModes: [ReadWriteMany]
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-0123456789abcdef0
```

---

## 11. Observability

### Control plane logs to CloudWatch

```bash
# Enable all control plane log types
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

### CloudWatch Container Insights

```bash
# Quick install (managed add-on)
aws eks create-addon --cluster-name my-cluster --addon-name amazon-cloudwatch-observability
```

### Metrics Server (needed by HPA and `kubectl top`)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

---

## 12. Troubleshooting Quick Commands

```bash
# All events in a namespace, newest last
kubectl get events -n default --sort-by=.lastTimestamp

# Only warning events cluster-wide
kubectl get events -A --field-selector type=Warning

# Why is a pod pending? (usually scheduling / IP / resource reasons)
kubectl describe pod <pod> -n <ns> | tail -30

# System pods health
kubectl get pods -n kube-system

# CoreDNS logs (name resolution issues)
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100

# VPC CNI logs (pod IP allocation issues)
kubectl logs -n kube-system -l k8s-app=aws-node --tail=100

# kubelet on a node (via SSM Session Manager, no SSH key needed)
aws ssm start-session --target <instance-id>
sudo journalctl -u kubelet -n 200 --no-pager

# ALB controller logs (Ingress not provisioning)
kubectl logs -n kube-system deploy/aws-load-balancer-controller --tail=100

# Get the exact IAM identity kubectl is using
aws sts get-caller-identity
```

---

## Cluster Deletion Checklist (cost hygiene)

When you're done experimenting, delete resources in this order to avoid orphans:

```bash
# 1. Delete workloads that provisioned AWS resources (LB, PVCs)
kubectl delete ingress --all -A
kubectl delete svc --all -A --field-selector spec.type=LoadBalancer
kubectl delete pvc --all -A

# 2. Uninstall Helm releases
helm list -A --short | xargs -I {} helm uninstall {} -n kube-system

# 3. Delete node groups
eksctl delete nodegroup --cluster my-cluster --name workers --drain

# 4. Delete the cluster (this also removes eksctl-created VPC)
eksctl delete cluster --name my-cluster

# 5. Verify no orphans
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerArn"
aws ec2 describe-volumes --filters "Name=status,Values=available"
aws ec2 describe-addresses --query "Addresses[?AssociationId==null]"
```
