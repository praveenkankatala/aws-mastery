# 🧪 Hands-On Labs — AWS ECS from Zero to Production

> Nine end-to-end labs that take you from a blank AWS account to a fully-automated, auto-scaling, Blue/Green-deployed containerized application on ECS Fargate.

---

## 📋 Lab Index

| # | Lab | Time | Goal |
|---|---|---|---|
| **1** | [Prepare a Sample App & Dockerfile](#-lab-1-prepare-a-sample-app--dockerfile) | 15 min | Have a working Docker image locally |
| **2** | [Create an ECR Repository & Push Image](#-lab-2-create-an-ecr-repository--push-image) | 10 min | Store image in ECR |
| **3** | [Create an ECS Cluster (Fargate)](#-lab-3-create-an-ecs-cluster-fargate) | 5 min | Cluster ready for tasks |
| **4** | [Create Task Definition & IAM Roles](#-lab-4-create-task-definition--iam-roles) | 20 min | Blueprint registered |
| **5** | [Set Up ALB + Target Group + Service](#-lab-5-set-up-alb--target-group--service) | 25 min | Public endpoint serving traffic |
| **6** | [Configure Auto Scaling](#-lab-6-configure-auto-scaling) | 15 min | Service scales on CPU load |
| **7** | [Build a GitHub Actions CI/CD Pipeline](#-lab-7-build-a-github-actions-cicd-pipeline) | 30 min | Push code → auto-deploy |
| **8** | [Implement Blue/Green with CodeDeploy](#-lab-8-implement-bluegreen-with-codedeploy) | 40 min | Zero-risk deployments |
| **9** | [Attach Persistent Storage with EFS](#-lab-9-attach-persistent-storage-with-efs) | 20 min | Stateful workloads |

---

## 🔧 Prerequisites

- AWS CLI v2 installed and configured: `aws configure`
- Docker Desktop installed
- Git installed
- A GitHub account (for Lab 7)
- Basic bash comfort

Set your environment:

```bash
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REPO="hello-ecs"
export ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
export CLUSTER_NAME="learn-ecs-cluster"
export SERVICE_NAME="hello-service"
export TASK_FAMILY="hello-task"
```

---

## 🚀 Lab 1: Prepare a Sample App & Dockerfile

**Goal:** Build a minimal Node.js app and Docker image locally.

### Step 1.1 — Create a project folder

```bash
mkdir hello-ecs && cd hello-ecs
```

### Step 1.2 — Create `app.js`

```javascript
// app.js
const express = require('express');
const app = express();
const PORT = process.env.PORT || 8080;
const VERSION = process.env.APP_VERSION || '1.0.0';

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from ECS!',
    version: VERSION,
    hostname: require('os').hostname(),
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req, res) => {
  res.status(200).send('OK');
});

app.listen(PORT, () => {
  console.log(`Server listening on port ${PORT}`);
});
```

### Step 1.3 — Create `package.json`

```json
{
  "name": "hello-ecs",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": { "start": "node app.js" },
  "dependencies": { "express": "^4.18.2" }
}
```

### Step 1.4 — Create `Dockerfile`

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 8080
CMD ["node", "app.js"]
```

### Step 1.5 — Build and test locally

```bash
docker build -t hello-ecs:local .
docker run -p 8080:8080 hello-ecs:local
```

In another terminal:

```bash
curl http://localhost:8080
# → { "message": "Hello from ECS!", ... }

curl http://localhost:8080/health
# → OK
```

✅ **Lab 1 complete.** Kill the container with `Ctrl+C`.

---

## 📦 Lab 2: Create an ECR Repository & Push Image

**Goal:** Push your image to Amazon ECR so ECS can pull it.

### Step 2.1 — Create the repository

```bash
aws ecr create-repository \
  --repository-name $ECR_REPO \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region $AWS_REGION
```

### Step 2.2 — Authenticate Docker with ECR

```bash
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin $ECR_URI
```

### Step 2.3 — Tag and push

```bash
IMAGE_TAG="v1.0.0"
docker tag hello-ecs:local $ECR_URI:$IMAGE_TAG
docker push $ECR_URI:$IMAGE_TAG
```

### Step 2.4 — Verify

```bash
aws ecr list-images --repository-name $ECR_REPO
```

Expected output:

```json
{ "imageIds": [ { "imageDigest": "sha256:...", "imageTag": "v1.0.0" } ] }
```

✅ **Lab 2 complete.**

---

## 🏗 Lab 3: Create an ECS Cluster (Fargate)

**Goal:** Provision a cluster ready to run Fargate tasks.

### Step 3.1 — Create the cluster

```bash
aws ecs create-cluster \
  --cluster-name $CLUSTER_NAME \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1 \
  --settings name=containerInsights,value=enabled
```

### Step 3.2 — Verify

```bash
aws ecs describe-clusters --clusters $CLUSTER_NAME \
  --query 'clusters[0].{Name:clusterName,Status:status,CI:settings}'
```

✅ **Lab 3 complete.**

---

## 📄 Lab 4: Create Task Definition & IAM Roles

**Goal:** Register a Task Definition with proper IAM roles.

### Step 4.1 — Create the Task Execution Role

```bash
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ecs-tasks.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

### Step 4.2 — Create the Task Role (permissions for the app)

```bash
aws iam create-role \
  --role-name helloEcsTaskRole \
  --assume-role-policy-document file://trust-policy.json

# For this demo, no extra permissions needed. In real apps, attach policies here.
```

### Step 4.3 — Create the CloudWatch Log Group

```bash
aws logs create-log-group --log-group-name /ecs/hello-ecs
```

### Step 4.4 — Write the Task Definition

Create `task-definition.json`:

```json
{
  "family": "hello-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/helloEcsTaskRole",
  "containerDefinitions": [
    {
      "name": "hello-container",
      "image": "YOUR_ECR_URI:v1.0.0",
      "essential": true,
      "portMappings": [
        { "containerPort": 8080, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "APP_VERSION", "value": "1.0.0" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/hello-ecs",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "app"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "wget -qO- http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 30
      }
    }
  ]
}
```

Replace `YOUR_ACCOUNT_ID` and `YOUR_ECR_URI`:

```bash
sed -i "s/YOUR_ACCOUNT_ID/${AWS_ACCOUNT_ID}/g" task-definition.json
sed -i "s|YOUR_ECR_URI|${ECR_URI}|g" task-definition.json
```

### Step 4.5 — Register the Task Definition

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

Take note of the revision number in the output (should be `:1`).

✅ **Lab 4 complete.**

---

## ⚖ Lab 5: Set Up ALB + Target Group + Service

**Goal:** Public-facing Application Load Balancer routing to Fargate tasks.

### Step 5.1 — Get your default VPC and subnets

```bash
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query 'Vpcs[0].VpcId' --output text)

export SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[].SubnetId' --output text | tr '\t' ',')

echo "VPC: $VPC_ID"
echo "Subnets: $SUBNET_IDS"
```

### Step 5.2 — Create security groups

```bash
# Security group for the ALB (allows public HTTP)
ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name hello-alb-sg \
  --description "ALB security group" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0

# Security group for the ECS tasks (allows only ALB → task)
TASK_SG_ID=$(aws ec2 create-security-group \
  --group-name hello-task-sg \
  --description "Task security group" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $TASK_SG_ID --protocol tcp --port 8080 --source-group $ALB_SG_ID
```

### Step 5.3 — Create the ALB

```bash
FIRST_TWO_SUBNETS=$(echo $SUBNET_IDS | cut -d',' -f1,2 | tr ',' ' ')

ALB_ARN=$(aws elbv2 create-load-balancer \
  --name hello-alb \
  --subnets $FIRST_TWO_SUBNETS \
  --security-groups $ALB_SG_ID \
  --scheme internet-facing \
  --type application \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

echo "ALB: $ALB_ARN"
```

### Step 5.4 — Create the Target Group

```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name hello-tg \
  --protocol HTTP --port 8080 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

echo "Target Group: $TG_ARN"
```

### Step 5.5 — Create the Listener

```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

### Step 5.6 — Create the ECS Service

```bash
SUBNETS_JSON=$(echo $SUBNET_IDS | tr ',' '\n' | head -2 | jq -R . | jq -s -c .)

aws ecs create-service \
  --cluster $CLUSTER_NAME \
  --service-name $SERVICE_NAME \
  --task-definition $TASK_FAMILY \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=${SUBNETS_JSON},
    securityGroups=[${TASK_SG_ID}],
    assignPublicIp=ENABLED
  }" \
  --load-balancers "targetGroupArn=${TG_ARN},containerName=hello-container,containerPort=8080" \
  --health-check-grace-period-seconds 60 \
  --deployment-configuration "deploymentCircuitBreaker={enable=true,rollback=true},minimumHealthyPercent=100,maximumPercent=200"
```

> Note: We use `assignPublicIp=ENABLED` here because we're deploying to public subnets for simplicity. In production, put tasks in **private** subnets with NAT Gateway or VPC Endpoints.

### Step 5.7 — Wait for the service to stabilize

```bash
aws ecs wait services-stable --cluster $CLUSTER_NAME --services $SERVICE_NAME
```

### Step 5.8 — Get the ALB DNS and test

```bash
ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)

echo "Visit: http://${ALB_DNS}"
curl "http://${ALB_DNS}"
```

Expected: `{ "message": "Hello from ECS!", ... }`

✅ **Lab 5 complete.** Your app is live!

---

## 📈 Lab 6: Configure Auto Scaling

**Goal:** Automatically scale tasks based on CPU load.

### Step 6.1 — Register the Service as a Scalable Target

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/${SERVICE_NAME} \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10
```

### Step 6.2 — Create a Target Tracking Policy on CPU

```bash
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/${SERVICE_NAME} \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": { "PredefinedMetricType": "ECSServiceAverageCPUUtilization" },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

### Step 6.3 — Verify

```bash
aws application-autoscaling describe-scaling-policies \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/${SERVICE_NAME}
```

### Step 6.4 — Load test (optional)

Install `hey` (Go-based HTTP load generator):

```bash
# Mac: brew install hey
# Linux: install from https://github.com/rakyll/hey

hey -z 5m -c 50 "http://${ALB_DNS}"
```

In another terminal, watch tasks scale up:

```bash
watch -n 5 "aws ecs describe-services --cluster $CLUSTER_NAME --services $SERVICE_NAME \
  --query 'services[0].{Desired:desiredCount,Running:runningCount}'"
```

✅ **Lab 6 complete.**

---

## 🔄 Lab 7: Build a GitHub Actions CI/CD Pipeline

**Goal:** Push to `main` → new image builds → auto-deploys to ECS.

### Step 7.1 — Push your code to GitHub

```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin git@github.com:YOUR_USER/hello-ecs.git
git push -u origin main
```

### Step 7.2 — Add AWS credentials as GitHub Secrets

In your repo → **Settings → Secrets and variables → Actions → New repository secret**:

- `AWS_ACCESS_KEY_ID` — Your IAM user's access key
- `AWS_SECRET_ACCESS_KEY` — Your IAM user's secret key

> **Production recommendation:** Use **OIDC federation** instead of long-lived access keys.

### Step 7.3 — Create the workflow file

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Amazon ECS

on:
  push:
    branches: [ "main" ]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: hello-ecs
  ECS_SERVICE: hello-service
  ECS_CLUSTER: learn-ecs-cluster
  ECS_TASK_DEFINITION: task-definition.json
  CONTAINER_NAME: hello-container

jobs:
  deploy:
    name: Build & Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Download current task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition hello-task \
            --query taskDefinition > task-definition.json

      - name: Render new task definition with the new image
        id: render-task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ${{ steps.render-task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

### Step 7.4 — Test the pipeline

Make a change to `app.js`:

```javascript
// Change the message
res.json({ message: 'Hello from ECS — v2!', ... });
```

Commit and push:

```bash
git add . && git commit -m "Update message to v2" && git push
```

Watch the workflow in GitHub Actions. After ~3 minutes:

```bash
curl "http://${ALB_DNS}"
# → { "message": "Hello from ECS — v2!", ... }
```

✅ **Lab 7 complete. You have full CI/CD!**

---

## 🔵🟢 Lab 8: Implement Blue/Green with CodeDeploy

**Goal:** Zero-risk deployments with instant rollback.

### Step 8.1 — Create a second Target Group (Green)

```bash
TG_GREEN_ARN=$(aws elbv2 create-target-group \
  --name hello-tg-green \
  --protocol HTTP --port 8080 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-path /health \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
```

### Step 8.2 — Create CodeDeploy IAM Role

```bash
cat > cd-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "codedeploy.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name ecsCodeDeployRole \
  --assume-role-policy-document file://cd-trust.json

aws iam attach-role-policy \
  --role-name ecsCodeDeployRole \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeDeployRoleForECS
```

### Step 8.3 — Update the ECS Service to use CodeDeploy

You'll need to recreate the service with `deploymentController=CODE_DEPLOY`:

```bash
# First delete the existing service
aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME --desired-count 0
aws ecs wait services-stable --cluster $CLUSTER_NAME --services $SERVICE_NAME
aws ecs delete-service --cluster $CLUSTER_NAME --service $SERVICE_NAME

# Recreate with CODE_DEPLOY controller
aws ecs create-service \
  --cluster $CLUSTER_NAME \
  --service-name $SERVICE_NAME \
  --task-definition $TASK_FAMILY \
  --desired-count 2 \
  --launch-type FARGATE \
  --deployment-controller type=CODE_DEPLOY \
  --network-configuration "awsvpcConfiguration={
    subnets=${SUBNETS_JSON},
    securityGroups=[${TASK_SG_ID}],
    assignPublicIp=ENABLED
  }" \
  --load-balancers "targetGroupArn=${TG_ARN},containerName=hello-container,containerPort=8080" \
  --health-check-grace-period-seconds 60
```

### Step 8.4 — Create the CodeDeploy Application

```bash
aws deploy create-application \
  --application-name hello-ecs-app \
  --compute-platform ECS
```

### Step 8.5 — Create the Deployment Group

```bash
aws deploy create-deployment-group \
  --application-name hello-ecs-app \
  --deployment-group-name hello-dg \
  --service-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsCodeDeployRole \
  --deployment-config-name CodeDeployDefault.ECSLinear10PercentEvery1Minutes \
  --auto-rollback-configuration enabled=true,events=DEPLOYMENT_FAILURE \
  --deployment-style deploymentType=BLUE_GREEN,deploymentOption=WITH_TRAFFIC_CONTROL \
  --blue-green-deployment-configuration '{
    "terminateBlueInstancesOnDeploymentSuccess": {
      "action": "TERMINATE",
      "terminationWaitTimeInMinutes": 5
    },
    "deploymentReadyOption": { "actionOnTimeout": "CONTINUE_DEPLOYMENT" }
  }' \
  --load-balancer-info '{
    "targetGroupPairInfoList": [{
      "targetGroups": [
        { "name": "hello-tg" },
        { "name": "hello-tg-green" }
      ],
      "prodTrafficRoute": { "listenerArns": ["'$LISTENER_ARN'"] }
    }]
  }' \
  --ecs-services serviceName=$SERVICE_NAME,clusterName=$CLUSTER_NAME
```

### Step 8.6 — Create an `appspec.yaml`

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION_ARN>
        LoadBalancerInfo:
          ContainerName: "hello-container"
          ContainerPort: 8080
```

### Step 8.7 — Trigger a deployment

Update your task definition with a new image, then:

```bash
aws deploy create-deployment \
  --application-name hello-ecs-app \
  --deployment-group-name hello-dg \
  --revision '{
    "revisionType": "AppSpecContent",
    "appSpecContent": { "content": "<appspec.yaml as escaped JSON string>" }
  }'
```

Watch traffic shift in the AWS Console → CodeDeploy → Deployments.

✅ **Lab 8 complete.**

---

## 💾 Lab 9: Attach Persistent Storage with EFS

**Goal:** Give tasks a shared, persistent filesystem.

### Step 9.1 — Create the EFS filesystem

```bash
EFS_ID=$(aws efs create-file-system \
  --creation-token hello-efs-$(date +%s) \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --query 'FileSystemId' --output text)

echo "EFS: $EFS_ID"
```

### Step 9.2 — Create EFS mount targets (one per subnet)

```bash
# Allow NFS from tasks
EFS_SG_ID=$(aws ec2 create-security-group \
  --group-name hello-efs-sg \
  --description "EFS SG" --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $EFS_SG_ID --protocol tcp --port 2049 --source-group $TASK_SG_ID

# Create mount targets in each subnet
for SUBNET in $(echo $SUBNET_IDS | tr ',' ' '); do
  aws efs create-mount-target \
    --file-system-id $EFS_ID \
    --subnet-id $SUBNET \
    --security-groups $EFS_SG_ID
done
```

### Step 9.3 — Update Task Definition to mount EFS

Add to `task-definition.json`:

```json
"volumes": [{
  "name": "efs-data",
  "efsVolumeConfiguration": {
    "fileSystemId": "fs-XXXXXXXX",
    "rootDirectory": "/",
    "transitEncryption": "ENABLED"
  }
}],
"containerDefinitions": [{
  ...
  "mountPoints": [{
    "sourceVolume": "efs-data",
    "containerPath": "/data",
    "readOnly": false
  }]
}]
```

### Step 9.4 — Register the new revision and update the service

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME \
  --task-definition $TASK_FAMILY --force-new-deployment
```

Now any file written to `/data` inside your container persists across restarts and is visible to all running tasks.

✅ **Lab 9 complete.**

---

## 🧹 Cleanup

To avoid ongoing charges:

```bash
# Delete the service
aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME --desired-count 0
aws ecs wait services-stable --cluster $CLUSTER_NAME --services $SERVICE_NAME
aws ecs delete-service --cluster $CLUSTER_NAME --service $SERVICE_NAME

# Delete the cluster
aws ecs delete-cluster --cluster $CLUSTER_NAME

# Delete the ALB and target groups
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
aws elbv2 delete-target-group --target-group-arn $TG_ARN
aws elbv2 delete-target-group --target-group-arn $TG_GREEN_ARN

# Delete the ECR repo
aws ecr delete-repository --repository-name $ECR_REPO --force

# Delete IAM roles
aws iam detach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam delete-role --role-name ecsTaskExecutionRole
aws iam delete-role --role-name helloEcsTaskRole
aws iam detach-role-policy --role-name ecsCodeDeployRole \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeDeployRoleForECS
aws iam delete-role --role-name ecsCodeDeployRole

# Delete EFS
for MT in $(aws efs describe-mount-targets --file-system-id $EFS_ID --query 'MountTargets[].MountTargetId' --output text); do
  aws efs delete-mount-target --mount-target-id $MT
done
sleep 30
aws efs delete-file-system --file-system-id $EFS_ID

# Delete security groups
aws ec2 delete-security-group --group-id $ALB_SG_ID
aws ec2 delete-security-group --group-id $TASK_SG_ID
aws ec2 delete-security-group --group-id $EFS_SG_ID

# Delete log group
aws logs delete-log-group --log-group-name /ecs/hello-ecs
```

---

## 🎉 What You've Learned

By finishing these labs, you can:

- ✅ Build & push Docker images to ECR.
- ✅ Provision an ECS Fargate cluster from the CLI.
- ✅ Write Task Definitions with IAM, logging, and health checks.
- ✅ Configure ALBs with Target Groups and health checks.
- ✅ Set up auto-scaling policies driven by CloudWatch metrics.
- ✅ Automate deployments via GitHub Actions CI/CD.
- ✅ Implement Blue/Green deployments with CodeDeploy.
- ✅ Attach persistent EFS storage to Fargate tasks.

---

## 🔗 Next Steps

- **[troubleshooting.md](troubleshooting.md)** — When things go wrong, look here first.
- **[commands-cheatsheet.md](commands-cheatsheet.md)** — Every CLI command you need.
- **[what-why-how.md](what-why-how.md)** — Deepen your conceptual model.
- **[README.md](README.md)** — Architecture & feature overview.
