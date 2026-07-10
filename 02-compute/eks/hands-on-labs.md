# AWS EKS — Hands-On Labs

> Twelve progressive labs that take you from an empty AWS account to a production-grade EKS cluster with Karpenter, IRSA, persistent storage, monitoring, and a Terraform IaC deployment.

## Prerequisites Check

Before starting Lab 1, run these to confirm your local environment is ready:

```bash
aws --version         # aws-cli/2.x
kubectl version --client
eksctl version
helm version
terraform --version   # v1.5+
aws sts get-caller-identity   # confirms you're authenticated
```

Export common variables you'll reuse:

```bash
export AWS_REGION=us-east-1
export CLUSTER_NAME=eks-lab
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

---

## Table of Labs

| # | Lab | Time | Skills |
|---|-----|------|--------|
| 1 | [Create a cluster with eksctl](#lab-1-create-a-cluster-with-eksctl) | 20 min | eksctl, kubectl basics |
| 2 | [Deploy your first application](#lab-2-deploy-your-first-application) | 15 min | Deployments, Services |
| 3 | [Expose an app with LoadBalancer](#lab-3-expose-an-app-with-loadbalancer) | 15 min | NLB, Service type |
| 4 | [Install the AWS Load Balancer Controller](#lab-4-install-the-aws-load-balancer-controller) | 30 min | IRSA, Helm, Ingress → ALB |
| 5 | [Pod-level AWS access with EKS Pod Identity](#lab-5-pod-level-aws-access-with-eks-pod-identity) | 25 min | Pod Identity, IAM |
| 6 | [Persistent storage with EBS CSI](#lab-6-persistent-storage-with-ebs-csi) | 20 min | StorageClass, PVC, StatefulSet |
| 7 | [Horizontal Pod Autoscaling](#lab-7-horizontal-pod-autoscaling) | 20 min | Metrics Server, HPA |
| 8 | [Install and use Karpenter](#lab-8-install-and-use-karpenter) | 45 min | NodePool, EC2NodeClass, spot |
| 9 | [Observability: logging and metrics](#lab-9-observability-logging-and-metrics) | 30 min | Container Insights, control-plane logs |
| 10 | [Cluster upgrade with rollback safety](#lab-10-cluster-upgrade-with-rollback-safety) | 45 min | Upgrade Insights, node draining |
| 11 | [Deploy everything with Terraform](#lab-11-deploy-everything-with-terraform) | 60 min | Terraform, EKS module, VPC module |
| 12 | [Clean up all resources](#lab-12-clean-up-all-resources) | 10 min | Cost hygiene |

---

## Lab 1: Create a cluster with eksctl

**Goal:** Get a working EKS cluster in ~20 minutes with sane production-lean defaults.

### Step 1 — Write the cluster config

Save as `cluster.yaml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eks-lab
  region: us-east-1
  version: "1.30"

vpc:
  cidr: 10.10.0.0/16
  clusterEndpoints:
    publicAccess: true
    privateAccess: true

iam:
  withOIDC: true    # required for IRSA in later labs

managedNodeGroups:
  - name: core
    instanceType: m6i.large
    minSize: 2
    maxSize: 4
    desiredCapacity: 2
    volumeSize: 50
    privateNetworking: true
    labels:
      workload: general
    tags:
      Environment: lab

addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
  - name: aws-ebs-csi-driver
```

### Step 2 — Create the cluster

```bash
eksctl create cluster -f cluster.yaml
```

This takes 15-20 minutes. eksctl builds a CloudFormation stack for the VPC, then another for the cluster, then the node group.

### Step 3 — Verify

```bash
kubectl get nodes
kubectl get pods -A
```

You should see 2 nodes in `Ready` state and system pods running in `kube-system`.

### Step 4 — Inspect what was created

```bash
aws eks describe-cluster --name eks-lab --region us-east-1 \
  --query "cluster.{version:version,status:status,endpoint:endpoint,oidc:identity.oidc.issuer}"
```

**Common pitfall:** if `kubectl` returns `Unable to connect to the server`, run:
```bash
aws eks update-kubeconfig --name eks-lab --region us-east-1
```

---

## Lab 2: Deploy your first application

**Goal:** Understand Deployments, Services, and namespaces on EKS.

### Step 1 — Create a namespace

```bash
kubectl create namespace webapp
```

### Step 2 — Deploy a stateless app

Save as `nginx-deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: webapp
spec:
  replicas: 3
  selector:
    matchLabels: { app: nginx }
  template:
    metadata:
      labels: { app: nginx }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
```

Apply:

```bash
kubectl apply -f nginx-deploy.yaml
kubectl get pods -n webapp -w
```

### Step 3 — Expose it internally

```yaml
# nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: webapp
spec:
  type: ClusterIP
  selector: { app: nginx }
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f nginx-svc.yaml
```

### Step 4 — Test from inside the cluster

```bash
kubectl run -it --rm curl --image=curlimages/curl --restart=Never -- \
  curl -s nginx.webapp.svc.cluster.local
```

You should see the nginx welcome HTML.

---

## Lab 3: Expose an app with LoadBalancer

**Goal:** Get an internet-reachable endpoint using the classic `Service` type `LoadBalancer` (which provisions a Classic ELB or NLB, depending on annotations).

### Step 1 — Change the service type

```yaml
# nginx-nlb.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-public
  namespace: webapp
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector: { app: nginx }
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f nginx-nlb.yaml
kubectl get svc -n webapp nginx-public
```

Wait until `EXTERNAL-IP` shows a full `elb.amazonaws.com` hostname (2-3 min).

### Step 2 — Test

```bash
NLB=$(kubectl get svc -n webapp nginx-public -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -s http://$NLB
```

> **Note:** the "external" annotation requires the AWS Load Balancer Controller from Lab 4. If it isn't installed yet, EKS falls back to the legacy in-tree controller (which creates a Classic ELB) — that also works but is deprecated.

---

## Lab 4: Install the AWS Load Balancer Controller

**Goal:** Enable `Ingress` objects that produce real Application Load Balancers.

### Step 1 — Create the IAM policy

```bash
curl -o iam_policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### Step 2 — Create the IRSA-linked service account

```bash
eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### Step 3 — Install the controller with Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Step 4 — Verify

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl logs -n kube-system deploy/aws-load-balancer-controller | tail
```

### Step 5 — Create an Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-alb
  namespace: webapp
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress -n webapp -w
```

When the `ADDRESS` column populates with an `alb.amazonaws.com` hostname, hit it in your browser.

---

## Lab 5: Pod-level AWS access with EKS Pod Identity

**Goal:** Give a pod scoped S3 read access without touching the node's IAM role.

### Step 1 — Install the Pod Identity Agent add-on

```bash
aws eks create-addon --cluster-name $CLUSTER_NAME --addon-name eks-pod-identity-agent
kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent
```

### Step 2 — Create an IAM role that pods can assume

```bash
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "pods.eks.amazonaws.com"},
    "Action": ["sts:AssumeRole", "sts:TagSession"]
  }]
}
EOF

aws iam create-role \
  --role-name pod-s3-reader \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name pod-s3-reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### Step 3 — Associate the role with a service account

```bash
kubectl create serviceaccount s3-reader -n webapp

aws eks create-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --namespace webapp \
  --service-account s3-reader \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/pod-s3-reader
```

### Step 4 — Run a pod that uses the SA

```yaml
# aws-cli-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli
  namespace: webapp
spec:
  serviceAccountName: s3-reader
  containers:
    - name: cli
      image: amazon/aws-cli:latest
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f aws-cli-pod.yaml
kubectl exec -n webapp aws-cli -- aws sts get-caller-identity
kubectl exec -n webapp aws-cli -- aws s3 ls
```

The identity should be `pod-s3-reader`, not the node's role.

---

## Lab 6: Persistent storage with EBS CSI

**Goal:** Attach a persistent EBS volume to a stateful workload.

### Step 1 — Confirm the EBS CSI driver is running

```bash
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl get storageclass
```

### Step 2 — Create a gp3 StorageClass as default

```yaml
# gp3-sc.yaml
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
```

```bash
kubectl apply -f gp3-sc.yaml
```

### Step 3 — Deploy a StatefulSet using a PVC

```yaml
# postgres-ss.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: pg
  namespace: webapp
spec:
  serviceName: pg
  replicas: 1
  selector:
    matchLabels: { app: pg }
  template:
    metadata:
      labels: { app: pg }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: POSTGRES_PASSWORD
              value: labpass
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp3
        resources:
          requests:
            storage: 10Gi
```

```bash
kubectl apply -f postgres-ss.yaml
kubectl get pvc,pv -n webapp
```

### Step 4 — Persistence test

```bash
kubectl exec -n webapp pg-0 -- psql -U postgres -c "CREATE TABLE t (id serial); INSERT INTO t VALUES (default);"
kubectl delete pod -n webapp pg-0
kubectl exec -n webapp pg-0 -- psql -U postgres -c "SELECT * FROM t;"
```

The data survives the pod restart because the EBS volume is reattached.

---

## Lab 7: Horizontal Pod Autoscaling

**Goal:** Automatically scale pod replicas based on CPU load.

### Step 1 — Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

### Step 2 — Create an HPA on the nginx deployment

```bash
kubectl autoscale deployment nginx -n webapp --min=2 --max=10 --cpu-percent=50
kubectl get hpa -n webapp
```

### Step 3 — Generate load

```bash
kubectl run -n webapp load --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://nginx.webapp.svc.cluster.local; done"
```

Watch it scale:

```bash
kubectl get hpa -n webapp -w
kubectl get pods -n webapp -w
```

### Step 4 — Stop the load

```bash
kubectl delete pod -n webapp load
```

HPA scales back down after ~5 min (default cool-down).

---

## Lab 8: Install and use Karpenter

**Goal:** Replace fixed node group scaling with just-in-time provisioning.

### Step 1 — Tag your subnets and security groups

Karpenter discovers where to launch nodes via tags. Tag your existing private subnets and cluster SG:

```bash
CLUSTER_SG=$(aws eks describe-cluster --name $CLUSTER_NAME \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)

# Get private subnets that eksctl created
SUBNETS=$(aws eks describe-cluster --name $CLUSTER_NAME \
  --query "cluster.resourcesVpcConfig.subnetIds[]" --output text)

for s in $SUBNETS; do
  aws ec2 create-tags --resources $s --tags Key=karpenter.sh/discovery,Value=$CLUSTER_NAME
done

aws ec2 create-tags --resources $CLUSTER_SG \
  --tags Key=karpenter.sh/discovery,Value=$CLUSTER_NAME
```

### Step 2 — Create IAM role for Karpenter nodes

```bash
cat > karpenter-node-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role --role-name KarpenterNodeRole-$CLUSTER_NAME \
  --assume-role-policy-document file://karpenter-node-trust.json

for p in AmazonEKSWorkerNodePolicy AmazonEKS_CNI_Policy \
         AmazonEC2ContainerRegistryReadOnly AmazonSSMManagedInstanceCore; do
  aws iam attach-role-policy --role-name KarpenterNodeRole-$CLUSTER_NAME \
    --policy-arn arn:aws:iam::aws:policy/$p
done

aws iam create-instance-profile --instance-profile-name KarpenterNodeRole-$CLUSTER_NAME
aws iam add-role-to-instance-profile \
  --instance-profile-name KarpenterNodeRole-$CLUSTER_NAME \
  --role-name KarpenterNodeRole-$CLUSTER_NAME
```

### Step 3 — Grant Karpenter node role access to the cluster

```bash
aws eks create-access-entry \
  --cluster-name $CLUSTER_NAME \
  --principal-arn arn:aws:iam::${ACCOUNT_ID}:role/KarpenterNodeRole-$CLUSTER_NAME \
  --type EC2_LINUX
```

### Step 4 — Create the Karpenter controller role (Pod Identity based)

```bash
# Simplified — see Karpenter docs for the full policy JSON
aws iam create-role --role-name KarpenterControllerRole-$CLUSTER_NAME \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "pods.eks.amazonaws.com"},
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }]
  }'

# Attach the Karpenter controller managed policy or your own custom one
# (see https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/ for the JSON)
```

### Step 5 — Install Karpenter with Helm

```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "1.0.0" \
  --namespace kube-system \
  --set "settings.clusterName=$CLUSTER_NAME" \
  --set "settings.interruptionQueue=$CLUSTER_NAME" \
  --wait
```

Associate the controller SA with the IAM role:

```bash
aws eks create-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --namespace kube-system \
  --service-account karpenter \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/KarpenterControllerRole-$CLUSTER_NAME
```

### Step 6 — Deploy a NodePool and EC2NodeClass

```yaml
# karpenter-config.yaml
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
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["5"]
      nodeClassRef:
        name: default
  limits:
    cpu: "100"
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
  role: KarpenterNodeRole-eks-lab
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: eks-lab
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: eks-lab
```

```bash
kubectl apply -f karpenter-config.yaml
```

### Step 7 — Provoke a scale-up

```bash
kubectl create deployment inflate \
  --image=public.ecr.aws/eks-distro/kubernetes/pause:3.7 \
  --replicas=0

kubectl set resources deployment/inflate \
  --requests=cpu=1

kubectl scale deployment/inflate --replicas=30

# Watch new nodes appear
kubectl get nodes -w
kubectl logs -f -n kube-system -l app.kubernetes.io/name=karpenter
```

### Step 8 — Trigger consolidation

```bash
kubectl scale deployment/inflate --replicas=0

# Within 30-60 seconds, Karpenter should terminate the empty nodes
kubectl get nodeclaim
```

---

## Lab 9: Observability: logging and metrics

**Goal:** Turn on control-plane logs and CloudWatch Container Insights.

### Step 1 — Enable control-plane logs

```bash
aws eks update-cluster-config \
  --name $CLUSTER_NAME \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

### Step 2 — Install Container Insights

```bash
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name amazon-cloudwatch-observability
```

### Step 3 — View logs and metrics

Open **CloudWatch → Log groups** and look for `/aws/eks/eks-lab/cluster`.
Open **CloudWatch → Container Insights** to see per-pod CPU/memory/network.

### Step 4 — Query with CloudWatch Logs Insights

Example query — audit-log failed authentication attempts in the last hour:

```
fields @timestamp, user.username, verb, objectRef.resource, responseStatus.code
| filter responseStatus.code >= 400
| sort @timestamp desc
| limit 50
```

---

## Lab 10: Cluster upgrade with rollback safety

**Goal:** Practice the full upgrade cycle — insights, control plane, add-ons, node group.

### Step 1 — Check upgrade readiness (Upgrade Insights)

Open the EKS console → your cluster → **Upgrade Insights** tab. Fix any flagged deprecated APIs before proceeding.

Alternatively via CLI:

```bash
aws eks list-insights --cluster-name $CLUSTER_NAME
aws eks describe-insight --cluster-name $CLUSTER_NAME --id <insight-id>
```

### Step 2 — Upgrade the control plane

```bash
CURRENT=$(aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.version" --output text)
echo "Currently on: $CURRENT"

aws eks update-cluster-version \
  --name $CLUSTER_NAME \
  --kubernetes-version 1.31   # replace with next minor
```

Wait until status returns to `ACTIVE`. This takes ~10 min.

### Step 3 — Upgrade add-ons

```bash
for addon in vpc-cni coredns kube-proxy aws-ebs-csi-driver; do
  LATEST=$(aws eks describe-addon-versions \
    --kubernetes-version 1.31 --addon-name $addon \
    --query "addons[0].addonVersions[0].addonVersion" --output text)
  aws eks update-addon --cluster-name $CLUSTER_NAME --addon-name $addon --addon-version $LATEST
done
```

### Step 4 — Upgrade the managed node group

```bash
aws eks update-nodegroup-version \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name core \
  --kubernetes-version 1.31
```

Node group upgrades roll one instance at a time by default, cordoning and draining each node.

### Step 5 — Verify

```bash
kubectl get nodes    # all should show the new version
kubectl version --short
```

### Step 6 — (Only if something broke) roll back the control plane

Within **7 days** of the upgrade:

```bash
aws eks update-cluster-version \
  --name $CLUSTER_NAME \
  --kubernetes-version 1.30 \
  --force
```

---

## Lab 11: Deploy everything with Terraform

**Goal:** Reproduce a full EKS cluster from code — VPC, EKS control plane, node group, ALB controller, and Pod Identity.

### Project structure

```
terraform-eks/
├── main.tf
├── vpc.tf
├── eks.tf
├── alb-controller.tf
├── variables.tf
├── outputs.tf
└── versions.tf
```

### `versions.tf`

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws        = { source = "hashicorp/aws",       version = "~> 5.0" }
    kubernetes = { source = "hashicorp/kubernetes", version = "~> 2.30" }
    helm       = { source = "hashicorp/helm",       version = "~> 2.13" }
  }
}
```

### `variables.tf`

```hcl
variable "region"       { default = "us-east-1" }
variable "cluster_name" { default = "eks-tf" }
variable "cluster_version" { default = "1.30" }
```

### `main.tf`

```hcl
provider "aws" { region = var.region }

data "aws_availability_zones" "available" {}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
  }
}

provider "helm" {
  kubernetes {
    host                   = module.eks.cluster_endpoint
    cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
    }
  }
}
```

### `vpc.tf`

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.cluster_name}-vpc"
  cidr = "10.0.0.0/16"
  azs  = slice(data.aws_availability_zones.available.names, 0, 3)

  private_subnets = ["10.0.1.0/20",  "10.0.17.0/20", "10.0.33.0/20"]
  public_subnets  = ["10.0.49.0/20", "10.0.65.0/20", "10.0.81.0/20"]

  enable_nat_gateway   = true
  single_nat_gateway   = true   # switch to false in prod
  enable_dns_hostnames = true
  enable_dns_support   = true

  public_subnet_tags = {
    "kubernetes.io/role/elb"           = "1"
    "karpenter.sh/discovery"           = var.cluster_name
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb"  = "1"
    "karpenter.sh/discovery"           = var.cluster_name
  }
}
```

### `eks.tf`

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # Modern API-driven auth (no aws-auth ConfigMap)
  authentication_mode                       = "API"
  enable_cluster_creator_admin_permissions  = true

  cluster_endpoint_public_access = true

  cluster_addons = {
    vpc-cni                = { most_recent = true }
    coredns                = { most_recent = true }
    kube-proxy             = { most_recent = true }
    aws-ebs-csi-driver     = { most_recent = true }
    eks-pod-identity-agent = { most_recent = true }
  }

  eks_managed_node_groups = {
    core = {
      min_size       = 2
      max_size       = 6
      desired_size   = 2
      instance_types = ["m6i.large"]
      labels = {
        Environment = "lab"
        Workload    = "general"
      }
    }
  }
}
```

### `alb-controller.tf`

```hcl
# IAM policy for the controller
data "http" "alb_policy" {
  url = "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json"
}

resource "aws_iam_policy" "alb" {
  name   = "${var.cluster_name}-alb-controller"
  policy = data.http.alb_policy.response_body
}

# Role assumed via Pod Identity
resource "aws_iam_role" "alb" {
  name = "${var.cluster_name}-alb-controller-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "pods.eks.amazonaws.com" }
      Action = ["sts:AssumeRole", "sts:TagSession"]
    }]
  })
}

resource "aws_iam_role_policy_attachment" "alb" {
  role       = aws_iam_role.alb.name
  policy_arn = aws_iam_policy.alb.arn
}

resource "aws_eks_pod_identity_association" "alb" {
  cluster_name    = module.eks.cluster_name
  namespace       = "kube-system"
  service_account = "aws-load-balancer-controller"
  role_arn        = aws_iam_role.alb.arn
}

resource "helm_release" "alb" {
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace  = "kube-system"
  version    = "1.8.1"

  set {
    name  = "clusterName"
    value = module.eks.cluster_name
  }
  set {
    name  = "serviceAccount.create"
    value = "true"
  }
  set {
    name  = "serviceAccount.name"
    value = "aws-load-balancer-controller"
  }

  depends_on = [aws_eks_pod_identity_association.alb]
}
```

### `outputs.tf`

```hcl
output "cluster_name"     { value = module.eks.cluster_name }
output "cluster_endpoint" { value = module.eks.cluster_endpoint }
output "kubeconfig_cmd"   {
  value = "aws eks update-kubeconfig --name ${module.eks.cluster_name} --region ${var.region}"
}
```

### Run it

```bash
terraform init
terraform plan -out tfplan
terraform apply tfplan

# Configure kubectl
aws eks update-kubeconfig --name eks-tf --region us-east-1

# Verify
kubectl get nodes
kubectl get pods -n kube-system
```

### Destroy it

```bash
# Delete anything the cluster provisioned outside Terraform's knowledge
kubectl delete ingress --all -A
kubectl delete svc --all -A --field-selector spec.type=LoadBalancer

terraform destroy
```

---

## Lab 12: Clean up all resources

**Goal:** Kill everything you created so you don't pay overnight.

### Step 1 — Delete workloads that provision AWS assets

```bash
kubectl delete ingress --all -A
kubectl delete svc --all -A --field-selector spec.type=LoadBalancer
kubectl delete pvc --all -A
```

Wait ~2 min so the controller can delete the ALBs/EBS volumes.

### Step 2 — Uninstall Helm releases

```bash
helm list -A
helm uninstall aws-load-balancer-controller -n kube-system
helm uninstall karpenter -n kube-system   # if installed
```

### Step 3 — Delete IAM roles/policies you created

```bash
aws iam delete-role-policy --role-name pod-s3-reader --policy-name AmazonS3ReadOnlyAccess 2>/dev/null
aws iam detach-role-policy --role-name pod-s3-reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name pod-s3-reader

aws iam delete-policy --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy
```

### Step 4 — Delete the cluster

```bash
eksctl delete cluster --name $CLUSTER_NAME --region $AWS_REGION
```

### Step 5 — Verify no orphans

```bash
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerArn"
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[].VolumeId"
aws ec2 describe-addresses --query "Addresses[?AssociationId==null].AllocationId"
aws eks list-clusters
```

If any of the above return items, delete them individually.

---

## What's Next

- Extend Lab 11 to a **multi-environment** setup (dev / staging / prod) using Terraform workspaces or Terragrunt.
- Add **GitOps** (ArgoCD or Flux) to sync manifests from a Git repo.
- Add a **service mesh** (Istio, Linkerd, or AWS App Mesh) for mTLS and traffic shaping.
- Implement **Network Policies** using Calico or Cilium.
- Add **Secrets Management** with External Secrets Operator + AWS Secrets Manager.
- Practice **disaster recovery** with Velero for cluster backups.

Return to the **[README.md](README.md)** or dig into **[troubleshooting.md](troubleshooting.md)** whenever things break.
