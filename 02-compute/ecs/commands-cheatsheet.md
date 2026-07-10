# 🧰 AWS ECS — CLI Commands Cheat Sheet

> A complete, organized reference of every AWS CLI command you'll need for ECS, ECR, ALB, IAM, CodeDeploy, and Auto Scaling. Copy-paste ready.

> **Prerequisites:** AWS CLI v2 installed and configured (`aws configure`).

---

## 📋 Table of Contents

1. [Environment Variables Setup](#-environment-variables-setup)
2. [ECR — Registry Commands](#-ecr--registry-commands)
3. [ECS Cluster Commands](#-ecs-cluster-commands)
4. [Task Definition Commands](#-task-definition-commands)
5. [ECS Service Commands](#-ecs-service-commands)
6. [ECS Task Commands](#-ecs-task-commands)
7. [Container Instance Commands (EC2 Launch Type)](#-container-instance-commands-ec2-launch-type)
8. [Load Balancer (ALB) Commands](#-load-balancer-alb-commands)
9. [Auto Scaling Commands](#-auto-scaling-commands)
10. [CloudWatch Logs & Metrics](#-cloudwatch-logs--metrics)
11. [IAM Role Commands](#-iam-role-commands)
12. [Secrets Manager & Parameter Store](#-secrets-manager--parameter-store)
13. [CodeDeploy — Blue/Green Commands](#-codedeploy--bluegreen-commands)
14. [ECS Exec (Debug Live Containers)](#-ecs-exec-debug-live-containers)
15. [Capacity Providers](#-capacity-providers)
16. [Docker Local Commands](#-docker-local-commands)
17. [One-Liners & Power Commands](#-one-liners--power-commands)

---

## 🌍 Environment Variables Setup

Set these once at the top of your terminal session to make later commands cleaner.

```bash
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME="my-cluster"
export SERVICE_NAME="my-service"
export TASK_FAMILY="my-app-task"
export ECR_REPO="my-app"
export ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
```

---

## 🐳 ECR — Registry Commands

### Create a Repository

```bash
aws ecr create-repository \
  --repository-name my-app \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region $AWS_REGION
```

### List Repositories

```bash
aws ecr describe-repositories
```

### Authenticate Docker with ECR

```bash
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin $ECR_URI
```

### List Images in a Repository

```bash
aws ecr list-images --repository-name $ECR_REPO
```

### Describe Image Details (size, pushed date, scan results)

```bash
aws ecr describe-images --repository-name $ECR_REPO
```

### Get Image Vulnerability Scan Results

```bash
aws ecr describe-image-scan-findings \
  --repository-name $ECR_REPO \
  --image-id imageTag=v1.0.0
```

### Delete an Image

```bash
aws ecr batch-delete-image \
  --repository-name $ECR_REPO \
  --image-ids imageTag=v1.0.0
```

### Delete a Repository

```bash
aws ecr delete-repository --repository-name $ECR_REPO --force
```

### Set Lifecycle Policy (auto-clean old images)

```bash
aws ecr put-lifecycle-policy \
  --repository-name $ECR_REPO \
  --lifecycle-policy-text '{
    "rules": [{
      "rulePriority": 1,
      "description": "Keep last 10 images",
      "selection": { "tagStatus": "any", "countType": "imageCountMoreThan", "countNumber": 10 },
      "action": { "type": "expire" }
    }]
  }'
```

---

## 🧱 ECS Cluster Commands

### Create a Cluster (Fargate-ready)

```bash
aws ecs create-cluster \
  --cluster-name $CLUSTER_NAME \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1 \
  --settings name=containerInsights,value=enabled
```

### List All Clusters

```bash
aws ecs list-clusters
```

### Describe a Cluster

```bash
aws ecs describe-clusters --clusters $CLUSTER_NAME
```

### Delete a Cluster

```bash
aws ecs delete-cluster --cluster $CLUSTER_NAME
```

### Update Cluster Settings (Enable Container Insights)

```bash
aws ecs update-cluster-settings \
  --cluster $CLUSTER_NAME \
  --settings name=containerInsights,value=enabled
```

---

## 📄 Task Definition Commands

### Register a Task Definition (from JSON file)

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

### List Task Definition Families

```bash
aws ecs list-task-definition-families
```

### List All Revisions of a Family

```bash
aws ecs list-task-definitions --family-prefix $TASK_FAMILY
```

### Describe a Specific Revision

```bash
aws ecs describe-task-definition --task-definition ${TASK_FAMILY}:5
```

### Download Current Task Definition as JSON (for editing)

```bash
aws ecs describe-task-definition \
  --task-definition $TASK_FAMILY \
  --query 'taskDefinition' > task-definition.json
```

### Deregister an Old Revision

```bash
aws ecs deregister-task-definition --task-definition ${TASK_FAMILY}:3
```

### Get the Latest Revision Number

```bash
aws ecs describe-task-definition \
  --task-definition $TASK_FAMILY \
  --query 'taskDefinition.revision'
```

---

## 🛡 ECS Service Commands

### Create a Service (Fargate + ALB)

```bash
aws ecs create-service \
  --cluster $CLUSTER_NAME \
  --service-name $SERVICE_NAME \
  --task-definition $TASK_FAMILY \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-abc123,subnet-def456],
    securityGroups=[sg-0abc123],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=my-container,containerPort=8080" \
  --health-check-grace-period-seconds 60 \
  --deployment-configuration "deploymentCircuitBreaker={enable=true,rollback=true},minimumHealthyPercent=100,maximumPercent=200"
```

### List Services in a Cluster

```bash
aws ecs list-services --cluster $CLUSTER_NAME
```

### Describe a Service

```bash
aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services $SERVICE_NAME
```

### Update a Service to a New Task Definition Revision

```bash
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service $SERVICE_NAME \
  --task-definition ${TASK_FAMILY}:5 \
  --force-new-deployment
```

### Scale a Service Up/Down Manually

```bash
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service $SERVICE_NAME \
  --desired-count 5
```

### Wait for Service to Stabilize (useful in CI/CD)

```bash
aws ecs wait services-stable \
  --cluster $CLUSTER_NAME \
  --services $SERVICE_NAME
```

### Delete a Service (must scale to 0 first)

```bash
aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME --desired-count 0
aws ecs delete-service --cluster $CLUSTER_NAME --service $SERVICE_NAME
```

---

## ⚡ ECS Task Commands

### Run a One-Off Task (batch job)

```bash
aws ecs run-task \
  --cluster $CLUSTER_NAME \
  --task-definition $TASK_FAMILY \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-abc123],
    securityGroups=[sg-0abc123],
    assignPublicIp=ENABLED
  }"
```

### List Running Tasks

```bash
aws ecs list-tasks --cluster $CLUSTER_NAME --service-name $SERVICE_NAME
```

### List Tasks by Status

```bash
aws ecs list-tasks --cluster $CLUSTER_NAME --desired-status RUNNING
aws ecs list-tasks --cluster $CLUSTER_NAME --desired-status STOPPED
```

### Describe a Task (get exit code, stop reason, IPs)

```bash
aws ecs describe-tasks \
  --cluster $CLUSTER_NAME \
  --tasks <task-id-or-arn>
```

### Stop a Task

```bash
aws ecs stop-task \
  --cluster $CLUSTER_NAME \
  --task <task-id-or-arn> \
  --reason "Manual stop for debugging"
```

---

## 🖥 Container Instance Commands (EC2 Launch Type)

### List Container Instances in a Cluster

```bash
aws ecs list-container-instances --cluster $CLUSTER_NAME
```

### Describe Container Instance Resources

```bash
aws ecs describe-container-instances \
  --cluster $CLUSTER_NAME \
  --container-instances <instance-arn>
```

### Drain an Instance (before termination — moves tasks off)

```bash
aws ecs update-container-instances-state \
  --cluster $CLUSTER_NAME \
  --container-instances <instance-arn> \
  --status DRAINING
```

### Re-activate a Drained Instance

```bash
aws ecs update-container-instances-state \
  --cluster $CLUSTER_NAME \
  --container-instances <instance-arn> \
  --status ACTIVE
```

---

## 🌐 Load Balancer (ALB) Commands

### Create an Application Load Balancer

```bash
aws elbv2 create-load-balancer \
  --name my-alb \
  --subnets subnet-abc123 subnet-def456 \
  --security-groups sg-0abc123 \
  --scheme internet-facing \
  --type application
```

### Create a Target Group (IP-based for Fargate)

```bash
aws elbv2 create-target-group \
  --name my-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-abc123 \
  --target-type ip \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3
```

### Create a Listener

```bash
aws elbv2 create-listener \
  --load-balancer-arn <alb-arn> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=<tg-arn>
```

### List Targets in a Target Group

```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

### Get ALB DNS Name

```bash
aws elbv2 describe-load-balancers \
  --names my-alb \
  --query 'LoadBalancers[0].DNSName' --output text
```

---

## 📈 Auto Scaling Commands

### Register ECS Service as a Scalable Target

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/${SERVICE_NAME} \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10
```

### Create a Target Tracking Scaling Policy (CPU-based)

```bash
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/${SERVICE_NAME} \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": { "PredefinedMetricType": "ECSServiceAverageCPUUtilization" },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

### Create a Target Tracking Policy (ALB Request Count)

```bash
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/${SERVICE_NAME} \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name request-count-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 1000.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "app/my-alb/xxxxx/targetgroup/my-tg/yyyyy"
    }
  }'
```

### List Scaling Policies

```bash
aws application-autoscaling describe-scaling-policies \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/${SERVICE_NAME}
```

### Delete a Scaling Policy

```bash
aws application-autoscaling delete-scaling-policy \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/${SERVICE_NAME} \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-target-tracking
```

---

## 📊 CloudWatch Logs & Metrics

### Tail Logs from a Specific Task

```bash
aws logs tail /ecs/my-app --follow
```

### Get Recent Log Events

```bash
aws logs get-log-events \
  --log-group-name /ecs/my-app \
  --log-stream-name ecs/my-container/<task-id>
```

### List Log Streams

```bash
aws logs describe-log-streams --log-group-name /ecs/my-app
```

### Get ECS Service Metrics

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ClusterName,Value=$CLUSTER_NAME Name=ServiceName,Value=$SERVICE_NAME \
  --statistics Average \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60
```

### Create a CloudWatch Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=ClusterName,Value=$CLUSTER_NAME Name=ServiceName,Value=$SERVICE_NAME \
  --alarm-actions arn:aws:sns:region:acct:my-topic
```

---

## 🔐 IAM Role Commands

### Create the ECS Task Execution Role

```bash
# Trust policy
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

### Create a Task Role for App Permissions (S3 example)

```bash
aws iam create-role \
  --role-name myAppTaskRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name myAppTaskRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

---

## 🔒 Secrets Manager & Parameter Store

### Create a Secret

```bash
aws secretsmanager create-secret \
  --name prod/db/password \
  --secret-string "supersecret123"
```

### Retrieve a Secret

```bash
aws secretsmanager get-secret-value --secret-id prod/db/password
```

### Store a Value in SSM Parameter Store

```bash
aws ssm put-parameter \
  --name /prod/db/password \
  --value "supersecret123" \
  --type SecureString
```

### Retrieve an SSM Parameter

```bash
aws ssm get-parameter --name /prod/db/password --with-decryption
```

---

## 🔵🟢 CodeDeploy — Blue/Green Commands

### List Applications

```bash
aws deploy list-applications
```

### Create a Deployment Application

```bash
aws deploy create-application \
  --application-name my-ecs-app \
  --compute-platform ECS
```

### List Deployment Groups

```bash
aws deploy list-deployment-groups --application-name my-ecs-app
```

### List Recent Deployments

```bash
aws deploy list-deployments --application-name my-ecs-app
```

### Get Deployment Details

```bash
aws deploy get-deployment --deployment-id d-XXXXXXXXX
```

### Stop a Deployment (manual rollback)

```bash
aws deploy stop-deployment \
  --deployment-id d-XXXXXXXXX \
  --auto-rollback-enabled
```

### Create a Deployment (Blue/Green)

```bash
aws deploy create-deployment \
  --application-name my-ecs-app \
  --deployment-group-name my-dg \
  --revision '{
    "revisionType": "AppSpecContent",
    "appSpecContent": {
      "content": "<contents of appspec.yaml as JSON string>"
    }
  }'
```

---

## 🐚 ECS Exec (Debug Live Containers)

### Enable ECS Exec on the Service

```bash
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service $SERVICE_NAME \
  --enable-execute-command \
  --force-new-deployment
```

### Get a Shell in a Running Container

```bash
aws ecs execute-command \
  --cluster $CLUSTER_NAME \
  --task <task-id> \
  --container my-container \
  --interactive \
  --command "/bin/bash"
```

### Run a Single Command

```bash
aws ecs execute-command \
  --cluster $CLUSTER_NAME \
  --task <task-id> \
  --container my-container \
  --command "cat /app/config.json" \
  --interactive
```

> **Requires:** Session Manager plugin installed locally + Task Role with `ssmmessages:*` permissions.

---

## 🎛 Capacity Providers

### Create a Capacity Provider (EC2 Auto Scaling)

```bash
aws ecs create-capacity-provider \
  --name my-ec2-cp \
  --auto-scaling-group-provider "autoScalingGroupArn=<asg-arn>,managedScaling={
    status=ENABLED,
    targetCapacity=100,
    minimumScalingStepSize=1,
    maximumScalingStepSize=10
  },managedTerminationProtection=ENABLED"
```

### Associate Capacity Providers with a Cluster

```bash
aws ecs put-cluster-capacity-providers \
  --cluster $CLUSTER_NAME \
  --capacity-providers FARGATE FARGATE_SPOT my-ec2-cp \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=2 \
    capacityProvider=FARGATE_SPOT,weight=4
```

---

## 🐋 Docker Local Commands

### Build an Image

```bash
docker build -t my-app:$(git rev-parse --short HEAD) .
```

### Tag for ECR

```bash
docker tag my-app:$(git rev-parse --short HEAD) $ECR_URI:$(git rev-parse --short HEAD)
```

### Push to ECR

```bash
docker push $ECR_URI:$(git rev-parse --short HEAD)
```

### Run Locally to Test

```bash
docker run -p 8080:8080 --env-file .env my-app:latest
```

### Inspect Image Metadata

```bash
docker inspect $ECR_URI:v1.0.0
```

### Prune Unused Images

```bash
docker image prune -a
```

---

## ⚡ One-Liners & Power Commands

### Get the Latest Task Definition Revision Number

```bash
aws ecs describe-task-definition \
  --task-definition $TASK_FAMILY \
  --query 'taskDefinition.revision' --output text
```

### Get the Current Running Image URI

```bash
aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services $SERVICE_NAME \
  --query 'services[0].taskDefinition' --output text
```

### Redeploy the Same Service (force pull latest image)

```bash
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service $SERVICE_NAME \
  --force-new-deployment
```

### Kill All Running Tasks in a Service (Restart)

```bash
aws ecs list-tasks --cluster $CLUSTER_NAME --service-name $SERVICE_NAME --query 'taskArns[]' --output text | \
  xargs -n1 -I {} aws ecs stop-task --cluster $CLUSTER_NAME --task {}
```

### Get IPs of All Running Tasks

```bash
TASK_ARNS=$(aws ecs list-tasks --cluster $CLUSTER_NAME --service-name $SERVICE_NAME --query 'taskArns[]' --output text)
aws ecs describe-tasks --cluster $CLUSTER_NAME --tasks $TASK_ARNS \
  --query 'tasks[].attachments[].details[?name==`privateIPv4Address`].value' --output text
```

### Watch a Service Deployment in Real Time

```bash
watch -n 2 "aws ecs describe-services --cluster $CLUSTER_NAME --services $SERVICE_NAME \
  --query 'services[0].deployments' --output table"
```

### Get the ECS Cluster's Total Task Count

```bash
aws ecs describe-clusters --clusters $CLUSTER_NAME \
  --query 'clusters[0].{Running:runningTasksCount,Pending:pendingTasksCount}'
```

### Full Pipeline: Build, Push, and Deploy

```bash
IMAGE_TAG=$(git rev-parse --short HEAD)

# Build & Push
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI
docker build -t $ECR_URI:$IMAGE_TAG .
docker push $ECR_URI:$IMAGE_TAG

# Update Task Def
aws ecs describe-task-definition --task-definition $TASK_FAMILY --query taskDefinition > td.json
jq --arg IMG "$ECR_URI:$IMAGE_TAG" \
   '.containerDefinitions[0].image = $IMG |
    del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)' \
   td.json > new-td.json

NEW_REV=$(aws ecs register-task-definition --cli-input-json file://new-td.json \
  --query 'taskDefinition.taskDefinitionArn' --output text)

# Deploy
aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME --task-definition $NEW_REV
aws ecs wait services-stable --cluster $CLUSTER_NAME --services $SERVICE_NAME

echo "✅ Deployed $IMAGE_TAG successfully"
```

---

## 🔗 Related Files

- **[README.md](README.md)** — Architecture, concepts, features
- **[what-why-how.md](what-why-how.md)** — Conceptual explanations
- **[hands-on-labs.md](hands-on-labs.md)** — Step-by-step tutorials
- **[troubleshooting.md](troubleshooting.md)** — Common issues & fixes
