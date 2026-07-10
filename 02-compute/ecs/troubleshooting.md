# 🩺 AWS ECS — Troubleshooting Guide

> Common problems, symptoms, root causes, and battle-tested fixes for AWS ECS in production.

Every issue below follows the same structure:

- **🔍 Symptom** — What you observe
- **🎯 Likely Cause** — Why it's happening
- **✅ Fix** — Step-by-step resolution
- **🛡 Prevention** — How to stop it from recurring

---

## 📋 Table of Contents

1. [Task Stuck in PENDING State](#-1-task-stuck-in-pending-state)
2. [Task Keeps Crashing / CrashLoopBackOff](#-2-task-keeps-crashing--crashloopbackoff)
3. [Image Pull Failure (CannotPullContainerError)](#-3-image-pull-failure-cannotpullcontainererror)
4. [ALB Health Checks Failing](#-4-alb-health-checks-failing)
5. [Service Stuck in "In-Progress" Deployment](#-5-service-stuck-in-in-progress-deployment)
6. [Deployment Circuit Breaker Rollback](#-6-deployment-circuit-breaker-rollback)
7. [Task Stopped with Exit Code 137 (OOMKilled)](#-7-task-stopped-with-exit-code-137-oomkilled)
8. [Exit Code 139 (Segfault) or 1 (App Error)](#-8-exit-code-139-segfault-or-1-app-error)
9. [Cannot Reach the Application via ALB](#-9-cannot-reach-the-application-via-alb)
10. [Auto Scaling Not Triggering](#-10-auto-scaling-not-triggering)
11. [Secrets Not Injected into Container](#-11-secrets-not-injected-into-container)
12. [Fargate Task Cannot Access Internet / ECR](#-12-fargate-task-cannot-access-internet--ecr)
13. [Blue/Green Deployment Stuck or Failing](#-13-bluegreen-deployment-stuck-or-failing)
14. [ECS Exec (Shell Access) Not Working](#-14-ecs-exec-shell-access-not-working)
15. [High CPU or Memory Metrics — Unclear Which Task](#-15-high-cpu-or-memory-metrics--unclear-which-task)
16. [EFS Mount Failure](#-16-efs-mount-failure)
17. [Insufficient Capacity Errors](#-17-insufficient-capacity-errors)
18. [Fargate Spot Task Interrupted](#-18-fargate-spot-task-interrupted)
19. [CloudWatch Logs Not Appearing](#-19-cloudwatch-logs-not-appearing)
20. [Tasks Not Registering with Target Group](#-20-tasks-not-registering-with-target-group)

---

## 🔴 1. Task Stuck in PENDING State

### 🔍 Symptom
`aws ecs describe-tasks` shows `lastStatus: PENDING` for minutes, never transitioning to `RUNNING`.

### 🎯 Likely Cause
- **EC2 launch type:** No cluster capacity — no EC2 instance has enough free CPU/memory.
- **Fargate:** Task is waiting for an ENI to be provisioned in a subnet.
- **Networking:** Subnet is in a private VPC with no NAT/VPC Endpoint for ECR image pull.
- **IAM:** Task Execution Role lacks permission to pull the image.

### ✅ Fix

**Step 1:** Check the events tab of the service:
```bash
aws ecs describe-services --cluster $CLUSTER_NAME --services $SERVICE_NAME \
  --query 'services[0].events[0:10]'
```

**Step 2:** Look for messages like:
- `unable to place a task because no container instance met all of its requirements` → EC2 cluster is full → scale the ASG.
- `RESOURCE:MEMORY` or `RESOURCE:CPU` → insufficient available resources.

**Step 3 (Fargate):** Check subnet has an available IP:
```bash
aws ec2 describe-subnets --subnet-ids <subnet-id> \
  --query 'Subnets[0].AvailableIpAddressCount'
```

If low, either free IPs or use a bigger subnet.

**Step 4:** Verify Task Execution Role has:
- `ecr:GetAuthorizationToken`
- `ecr:BatchGetImage`
- `ecr:GetDownloadUrlForLayer`

### 🛡 Prevention
- Monitor EC2 cluster reservation metrics.
- Right-size subnets (`/24` or larger).
- Always attach the AWS-managed `AmazonECSTaskExecutionRolePolicy`.

---

## 🔴 2. Task Keeps Crashing / CrashLoopBackOff

### 🔍 Symptom
New tasks launch, run for a few seconds, then stop. Repeats indefinitely.

### 🎯 Likely Cause
- Application throwing an unhandled exception at startup.
- Missing environment variable or secret.
- Health check firing too early (before app is ready).
- Port mismatch (app listens on 3000, but Task Def says 8080).

### ✅ Fix

**Step 1:** Get the stop reason:
```bash
aws ecs describe-tasks --cluster $CLUSTER_NAME --tasks <task-id> \
  --query 'tasks[0].{Reason:stoppedReason,Containers:containers[].reason}'
```

**Step 2:** Check CloudWatch logs for the crash trace:
```bash
aws logs tail /ecs/my-app --follow
```

**Step 3:** Common findings:
- `ECONNREFUSED to DB` → DB security group / connection string wrong.
- `Cannot find module 'X'` → Docker build missed dependencies (check `.dockerignore`).
- `Address already in use` → Multiple containers competing for the same port.

**Step 4:** If the app is slow to boot, increase `startPeriod` in the container health check:
```json
"healthCheck": {
  "startPeriod": 60,
  "interval": 30,
  "timeout": 10,
  "retries": 3
}
```

### 🛡 Prevention
- Test the Docker image locally with `docker run` before pushing.
- Enable the **Deployment Circuit Breaker** with `rollback: true`.
- Log to stdout, not files.
- Set generous `startPeriod` for JVM/Rails-style slow-starting apps.

---

## 🔴 3. Image Pull Failure (CannotPullContainerError)

### 🔍 Symptom
Task shows `CannotPullContainerError: pull access denied` or `manifest not found`.

### 🎯 Likely Cause
- Task Execution Role lacks ECR permissions.
- Image tag typo (case-sensitive!).
- Fargate in private subnet without NAT/VPC Endpoint to reach ECR.
- Cross-region pull without ECR replication.

### ✅ Fix

**Step 1:** Verify image exists:
```bash
aws ecr describe-images --repository-name my-app \
  --image-ids imageTag=v1.0.0
```

**Step 2:** Confirm Task Execution Role has the ECR policy:
```bash
aws iam list-attached-role-policies --role-name ecsTaskExecutionRole
```

Must include `AmazonECSTaskExecutionRolePolicy`.

**Step 3:** For private subnets, ensure either:
- A **NAT Gateway** in a public subnet + route from private subnet, **or**
- **VPC Endpoints** for `com.amazonaws.<region>.ecr.api`, `com.amazonaws.<region>.ecr.dkr`, `com.amazonaws.<region>.s3` (for image layers), and `com.amazonaws.<region>.logs`.

**Step 4:** Verify image URI format:
```
<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>
```

### 🛡 Prevention
- Use **immutable tags** to prevent overwrites.
- Enable **image scanning on push** to catch bad images early.
- Standardize on VPC Endpoints in all production VPCs.

---

## 🔴 4. ALB Health Checks Failing

### 🔍 Symptom
Target Group shows targets as `unhealthy`. Application is unreachable via ALB DNS.

### 🎯 Likely Cause
- Health check path returns non-2xx (e.g., `/health` returns 404).
- Wrong port in Target Group vs Task Definition.
- Security group blocks ALB → task traffic.
- App takes longer to start than the health check grace period.

### ✅ Fix

**Step 1:** Check target health:
```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

Look at the `Reason` field:
- `Target.Timeout` → Task not reachable (SG issue).
- `Target.FailedHealthChecks` → App returning wrong status code.
- `Target.RegistrationInProgress` → Wait; grace period may need extending.

**Step 2:** Verify security groups:
- Task SG must allow inbound on the container port from the **ALB's security group** (not from `0.0.0.0/0`).

**Step 3:** Test health endpoint directly from another task in the same VPC:
```bash
curl http://<task-private-ip>:8080/health
```

**Step 4:** Increase `HealthCheckGracePeriodSeconds` in the service:
```bash
aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME \
  --health-check-grace-period-seconds 120
```

### 🛡 Prevention
- Always implement a real `/health` endpoint that verifies critical dependencies (DB, cache).
- Use `HealthyThresholdCount: 2`, `UnhealthyThresholdCount: 3` (not 1 — too jittery).
- Set container-level health check *and* ALB health check as defense in depth.

---

## 🔴 5. Service Stuck in "In-Progress" Deployment

### 🔍 Symptom
`aws ecs describe-services` shows `deployments` array with old + new deployments, and it never converges.

### 🎯 Likely Cause
- New tasks failing health checks; ECS retries indefinitely.
- Insufficient cluster capacity.
- ALB rejecting new tasks (health check misconfigured).

### ✅ Fix

**Step 1:** Watch deployment state:
```bash
watch -n 5 "aws ecs describe-services --cluster $CLUSTER_NAME --services $SERVICE_NAME \
  --query 'services[0].deployments'"
```

**Step 2:** Diagnose the failing tasks (see [issue #2](#-2-task-keeps-crashing--crashloopbackoff) and [#4](#-4-alb-health-checks-failing)).

**Step 3:** Force stop the deployment and revert:
```bash
# Roll back to the previous task def revision
aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME \
  --task-definition ${TASK_FAMILY}:PREVIOUS_REVISION \
  --force-new-deployment
```

### 🛡 Prevention
- **Always enable the Deployment Circuit Breaker** with rollback:
  ```json
  "deploymentCircuitBreaker": { "enable": true, "rollback": true }
  ```
- Test images in a staging cluster before pushing to prod.

---

## 🔴 6. Deployment Circuit Breaker Rollback

### 🔍 Symptom
Service events say `Circuit breaker rolled back`. Your new revision was reverted automatically.

### 🎯 Likely Cause
Your new Task Definition or Docker image is broken. ECS detected N consecutive failed task launches and reverted.

### ✅ Fix

**Step 1:** Look at the stopped tasks from the failed deployment:
```bash
aws ecs list-tasks --cluster $CLUSTER_NAME --desired-status STOPPED \
  --started-by "ecs-svc/deployment-id"
```

**Step 2:** Describe them for exit codes and reasons:
```bash
aws ecs describe-tasks --cluster $CLUSTER_NAME --tasks <task-arn>
```

**Step 3:** Fix the underlying issue (env var, image, config), test locally, and redeploy.

### 🛡 Prevention
- The Circuit Breaker doing its job **is a good outcome** — you just avoided prod downtime.
- Add pre-deployment smoke tests in your CI (build → run container → hit `/health`).

---

## 🔴 7. Task Stopped with Exit Code 137 (OOMKilled)

### 🔍 Symptom
Task stops abruptly. Container's `exitCode: 137` with reason "OutOfMemoryError: Container killed due to memory usage."

### 🎯 Likely Cause
Container tried to allocate more memory than its Task Definition limit → the OS killed it (`SIGKILL`).

### ✅ Fix

**Step 1:** Confirm memory limit:
```bash
aws ecs describe-task-definition --task-definition $TASK_FAMILY \
  --query 'taskDefinition.containerDefinitions[0].memory'
```

**Step 2:** Check CloudWatch Container Insights for peak memory usage.

**Step 3:** Increase the memory in the Task Definition:
```json
{
  "cpu": "512",
  "memory": "2048"
}
```

Register a new revision and deploy.

### 🛡 Prevention
- Profile your app under load *before* production.
- Add CloudWatch alarms on `MemoryUtilization > 80%` for early warning.
- For Node.js: set `--max-old-space-size` to match container limits.
- For Java: set `-XX:MaxRAMPercentage=75.0`.

---

## 🔴 8. Exit Code 139 (Segfault) or 1 (App Error)

### 🔍 Symptom
Container exits shortly after starting with exit code `1` (generic app error) or `139` (segmentation fault).

### 🎯 Likely Cause
- **Code 1:** Application-level exception at startup — missing env var, invalid config, DB unreachable.
- **Code 139:** Native code crash (usually in a binary dependency like sharp, node-sass, or a compiled Python module).
- Architecture mismatch (ARM64 image on x86_64 Fargate or vice versa).

### ✅ Fix

**Step 1:** Check application logs:
```bash
aws logs tail /ecs/my-app --follow
```

**Step 2 (segfault):** Verify architecture match:
```bash
docker inspect $ECR_URI:v1.0.0 --format '{{.Architecture}}'
```

Ensure it matches your Task Definition's `runtimePlatform`:
```json
"runtimePlatform": { "cpuArchitecture": "X86_64", "operatingSystemFamily": "LINUX" }
```

**Step 3:** Test the image locally with the same env vars:
```bash
docker run --rm -e KEY=value $ECR_URI:v1.0.0
```

### 🛡 Prevention
- Standardize on one architecture (ARM64 recommended for cost/perf).
- Add integration tests to CI that boot the container and check `/health`.

---

## 🔴 9. Cannot Reach the Application via ALB

### 🔍 Symptom
`curl http://alb-dns.amazonaws.com` returns `Connection timeout` or `502 Bad Gateway`.

### 🎯 Likely Cause
- ALB security group doesn't allow inbound from your IP.
- No tasks registered / healthy in the Target Group.
- ALB is in a different VPC than the tasks.
- Listener rules don't match the incoming path/host.

### ✅ Fix

**Step 1:** Check ALB security group inbound rules:
```bash
aws ec2 describe-security-groups --group-ids $ALB_SG_ID \
  --query 'SecurityGroups[0].IpPermissions'
```

Must allow port 80/443 from `0.0.0.0/0` (or your specific IPs).

**Step 2:** Check target health:
```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

Zero healthy targets → see [issue #4](#-4-alb-health-checks-failing).

**Step 3:** Verify ALB is in the same VPC as your tasks:
```bash
aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].VpcId'
```

**Step 4:** For 502 errors specifically: often means the target is unreachable at the network layer or is returning invalid HTTP.

### 🛡 Prevention
- Automate infrastructure with Terraform/CloudFormation to enforce consistency.
- Regularly audit security group rules.

---

## 🔴 10. Auto Scaling Not Triggering

### 🔍 Symptom
CPU is hitting 90%, but no new tasks are launching.

### 🎯 Likely Cause
- Scalable target not registered.
- Scaling policy not attached.
- CloudWatch alarm not firing.
- Max capacity already reached.

### ✅ Fix

**Step 1:** Verify scalable target:
```bash
aws application-autoscaling describe-scalable-targets \
  --service-namespace ecs \
  --resource-ids service/${CLUSTER_NAME}/${SERVICE_NAME}
```

**Step 2:** Verify policy:
```bash
aws application-autoscaling describe-scaling-policies \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/${SERVICE_NAME}
```

**Step 3:** Check CloudWatch alarm state:
```bash
aws cloudwatch describe-alarms --alarm-name-prefix "TargetTracking-service/"
```

State should be `ALARM` if metric exceeds threshold.

**Step 4:** Check activities/scaling history:
```bash
aws application-autoscaling describe-scaling-activities \
  --service-namespace ecs \
  --resource-id service/${CLUSTER_NAME}/${SERVICE_NAME}
```

### 🛡 Prevention
- After creating any scaling policy, load-test to verify it triggers.
- Set `max-capacity` generously — you can always scale down, but you can't scale beyond max.

---

## 🔴 11. Secrets Not Injected into Container

### 🔍 Symptom
App crashes on startup: `DB_PASSWORD is undefined` even though it's in Task Def's `secrets`.

### 🎯 Likely Cause
- Task Execution Role lacks `secretsmanager:GetSecretValue` or `ssm:GetParameters`.
- Wrong ARN in `valueFrom`.
- Secret is in a different region than the ECS cluster.
- KMS key access denied.

### ✅ Fix

**Step 1:** Test the secret manually:
```bash
aws secretsmanager get-secret-value --secret-id prod/db/password
```

**Step 2:** Attach permissions to Task Execution Role:
```bash
aws iam put-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-name SecretsAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue", "ssm:GetParameters", "kms:Decrypt"],
      "Resource": "*"
    }]
  }'
```

**Step 3:** Verify ARN format in Task Definition:
```json
"secrets": [{
  "name": "DB_PASSWORD",
  "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db/password-abcXYZ"
}]
```

Note: The ARN includes a random 6-char suffix (`-abcXYZ`). Copy the full ARN.

### 🛡 Prevention
- Use SSM Parameter Store for simple secrets (free tier).
- Use Secrets Manager for rotatable secrets ($0.40/secret/month).

---

## 🔴 12. Fargate Task Cannot Access Internet / ECR

### 🔍 Symptom
Task fails with `CannotPullContainerError: dial tcp: lookup ...ecr.amazonaws.com: no such host` or similar network timeout.

### 🎯 Likely Cause
- Task is in a **private subnet** with no NAT Gateway.
- No VPC Endpoints for ECR.
- Route table missing the internet route.
- Security group blocks outbound.

### ✅ Fix

**Option A (public subnet):** Set `assignPublicIp: ENABLED` in the network config. Only for demos — not production.

**Option B (recommended):** Deploy a **NAT Gateway** in a public subnet, and route the private subnet's `0.0.0.0/0` through it.

**Option C (best for cost & security):** Create **VPC Endpoints**:

```bash
# Interface endpoints (per-hour + data cost, but no NAT needed)
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --subnet-ids $SUBNET_IDS \
  --security-group-ids $ENDPOINT_SG_ID

aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ecr.api \
  --subnet-ids $SUBNET_IDS

aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.logs \
  --subnet-ids $SUBNET_IDS

# Gateway endpoint for S3 (free! required for ECR image layers)
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
  --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids $PRIVATE_RT_ID
```

### 🛡 Prevention
- Bake VPC Endpoints into your Terraform/CloudFormation VPC modules.
- Use S3 Gateway Endpoint (free) for cost savings on ECR pulls.

---

## 🔴 13. Blue/Green Deployment Stuck or Failing

### 🔍 Symptom
CodeDeploy deployment status: `Failed` or hangs on `Install` / `Ready`.

### 🎯 Likely Cause
- CodeDeploy service role lacks permissions.
- Green tasks fail health checks.
- Test listener misconfigured.
- Both target groups pointing at the wrong port.

### ✅ Fix

**Step 1:** Read the deployment lifecycle events:
```bash
aws deploy get-deployment --deployment-id d-XXXXXXXXX \
  --query 'deploymentInfo.errorInformation'
```

Common errors:
- `Target group with ARN ... not found` → wrong TG name.
- `Task Set failed to launch tasks` → check the underlying ECS service events.
- `LifecycleEvent hook 'AfterAllowTraffic' failed` → your Lambda test hook script returned failure.

**Step 2:** Stop the failing deployment:
```bash
aws deploy stop-deployment --deployment-id d-XXXXXXXXX --auto-rollback-enabled
```

**Step 3:** Verify Blue and Green target groups point at the same container port.

### 🛡 Prevention
- Always start with `All-at-Once` deploys in staging, then move to Canary/Linear in prod.
- Test Lambda validation hooks separately before wiring them into deployments.

---

## 🔴 14. ECS Exec (Shell Access) Not Working

### 🔍 Symptom
```
An error occurred (TargetNotConnectedException) when calling the ExecuteCommand operation
```

### 🎯 Likely Cause
- Service not started with `enableExecuteCommand: true`.
- Task Role lacks SSM permissions.
- SSM plugin not installed on your local machine.
- Task launched *before* Exec was enabled.

### ✅ Fix

**Step 1:** Install the Session Manager plugin locally:
- macOS: `brew install --cask session-manager-plugin`
- Linux: https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html

**Step 2:** Add SSM permissions to Task Role:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ssmmessages:CreateControlChannel",
      "ssmmessages:CreateDataChannel",
      "ssmmessages:OpenControlChannel",
      "ssmmessages:OpenDataChannel"
    ],
    "Resource": "*"
  }]
}
```

**Step 3:** Enable on the service:
```bash
aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME \
  --enable-execute-command --force-new-deployment
```

**Step 4:** Wait for new tasks to launch (existing tasks won't get Exec retroactively).

### 🛡 Prevention
- Enable ECS Exec by default in staging clusters for easy debugging.
- Never enable it in prod without strict IAM guardrails and audit logging.

---

## 🔴 15. High CPU or Memory Metrics — Unclear Which Task

### 🔍 Symptom
Service-level CPU is 85%, but you can't tell which specific task is the culprit.

### ✅ Fix

**Step 1:** Enable **Container Insights** on the cluster:
```bash
aws ecs update-cluster-settings --cluster $CLUSTER_NAME \
  --settings name=containerInsights,value=enabled
```

**Step 2:** In CloudWatch → Container Insights → ECS Clusters → your cluster, drill into per-task metrics.

**Step 3:** For runtime profiling, use ECS Exec to attach:
```bash
aws ecs execute-command --cluster $CLUSTER_NAME --task <task-id> \
  --container my-container --interactive --command "top"
```

### 🛡 Prevention
- Always enable Container Insights (small extra cost, huge visibility).
- Add APM (Datadog, New Relic, AWS X-Ray) for deep observability.

---

## 🔴 16. EFS Mount Failure

### 🔍 Symptom
Task fails with `ResourceInitializationError: failed to invoke EFS utils commands...` or `mount failed`.

### 🎯 Likely Cause
- EFS security group doesn't allow port 2049 from Task security group.
- No EFS mount target in the same AZ as the task.
- Wrong `fileSystemId` in Task Def.
- Missing `elasticfilesystem:ClientMount` in Task Role.

### ✅ Fix

**Step 1:** Verify mount targets exist in the task's subnets:
```bash
aws efs describe-mount-targets --file-system-id fs-XXXXXXXX
```

**Step 2:** Check EFS security group allows NFS (port 2049) from Task SG:
```bash
aws ec2 describe-security-groups --group-ids $EFS_SG_ID
```

**Step 3:** For access points, ensure the Task Role has:
```json
{
  "Effect": "Allow",
  "Action": ["elasticfilesystem:ClientMount", "elasticfilesystem:ClientWrite"],
  "Resource": "arn:aws:elasticfilesystem:region:acct:file-system/fs-XXXXXXXX"
}
```

### 🛡 Prevention
- Create mount targets in **every** subnet where tasks might run.
- Use EFS Access Points to enforce POSIX permissions and root directory isolation.

---

## 🔴 17. Insufficient Capacity Errors

### 🔍 Symptom
Task launch fails with `RESOURCE:CPU` or `RESOURCE:MEMORY` errors (EC2 launch type).

### 🎯 Likely Cause
Cluster's EC2 instances don't have enough free resources to place the new task.

### ✅ Fix

**Step 1:** Check current capacity:
```bash
aws ecs describe-clusters --clusters $CLUSTER_NAME \
  --include ATTACHMENTS \
  --query 'clusters[0].registeredContainerInstancesCount'
```

**Step 2:** Increase the Auto Scaling Group max size:
```bash
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name $ASG_NAME \
  --max-size 20 --desired-capacity 10
```

**Step 3 (permanent fix):** Switch to a Capacity Provider with `managedScaling` so the cluster auto-scales.

### 🛡 Prevention
- Use **Capacity Providers** with managed scaling — the cluster adds instances automatically.
- Or move to **Fargate** — no capacity planning at all.

---

## 🔴 18. Fargate Spot Task Interrupted

### 🔍 Symptom
Task stops with reason `SpotInterruption` or `Service was unable to place a task`.

### 🎯 Likely Cause
AWS reclaimed the Spot capacity (2-minute warning was sent).

### ✅ Fix

**Step 1:** Adjust your Capacity Provider strategy — increase the ratio of on-demand Fargate:
```
FARGATE:      base=4, weight=1
FARGATE_SPOT: weight=3
```

**Step 2:** Handle `SIGTERM` in your app to drain gracefully:

```javascript
process.on('SIGTERM', async () => {
  console.log('Received SIGTERM, draining connections...');
  server.close(() => process.exit(0));
});
```

**Step 3:** For workloads that cannot tolerate interruption, use standard Fargate only.

### 🛡 Prevention
- Only use Spot for **stateless, retry-safe** workloads (batch, workers).
- Maintain a baseline of on-demand tasks for critical services.

---

## 🔴 19. CloudWatch Logs Not Appearing

### 🔍 Symptom
No logs appear in CloudWatch `/ecs/my-app`, but the task is running.

### 🎯 Likely Cause
- Log group doesn't exist (and `awslogs-create-group` isn't set).
- Task Execution Role lacks `logs:CreateLogStream` / `logs:PutLogEvents`.
- App is logging to files instead of stdout/stderr.
- Wrong region in `awslogs-region`.

### ✅ Fix

**Step 1:** Create the log group manually:
```bash
aws logs create-log-group --log-group-name /ecs/my-app
```

**Or** enable auto-create in Task Def:
```json
"logConfiguration": {
  "logDriver": "awslogs",
  "options": {
    "awslogs-create-group": "true",
    "awslogs-group": "/ecs/my-app",
    "awslogs-region": "us-east-1",
    "awslogs-stream-prefix": "app"
  }
}
```

**Step 2:** Ensure app writes to **stdout/stderr**, not files. Docker captures those; files inside the container are wiped when the container stops.

For Node.js: `console.log()` ✅ Not `fs.writeFile('app.log', ...)` ❌
For Python: `print(..., flush=True)` ✅

### 🛡 Prevention
- Use structured logging (JSON) with a library like Winston, Pino, or Python's `structlog`.
- Set log retention to control CloudWatch costs:
  ```bash
  aws logs put-retention-policy --log-group-name /ecs/my-app --retention-in-days 30
  ```

---

## 🔴 20. Tasks Not Registering with Target Group

### 🔍 Symptom
Tasks are `RUNNING` but the Target Group shows zero targets.

### 🎯 Likely Cause
- Service was created without `loadBalancers` configuration.
- Container name in `loadBalancers` doesn't match container name in Task Def.
- Wrong container port.
- Task not launched by the Service (e.g., a `run-task` standalone).

### ✅ Fix

**Step 1:** Check the Service configuration:
```bash
aws ecs describe-services --cluster $CLUSTER_NAME --services $SERVICE_NAME \
  --query 'services[0].loadBalancers'
```

**Step 2:** Container name must match exactly:
```json
// Task Definition
"containerDefinitions": [{ "name": "hello-container", ... }]

// Service loadBalancers
{ "containerName": "hello-container", "containerPort": 8080, ... }
```

**Step 3:** If the service was created without ALB, you must delete and recreate it (you cannot add load balancers to an existing service).

### 🛡 Prevention
- Configure ALB attachment at service creation time.
- Use IaC (Terraform, CloudFormation) so this is versioned and reproducible.

---

## 🔧 General Debugging Toolkit

### The 5-Command Diagnostic Sweep

Run these when you have no idea what's wrong:

```bash
# 1. Service state
aws ecs describe-services --cluster $CLUSTER_NAME --services $SERVICE_NAME \
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount,Events:events[0:5]}'

# 2. Task health
aws ecs list-tasks --cluster $CLUSTER_NAME --service-name $SERVICE_NAME \
  --desired-status STOPPED --query 'taskArns[0]' --output text | \
  xargs -I {} aws ecs describe-tasks --cluster $CLUSTER_NAME --tasks {} \
  --query 'tasks[0].{Reason:stoppedReason,Containers:containers[].{Name:name,Reason:reason,Code:exitCode}}'

# 3. Target health
aws elbv2 describe-target-health --target-group-arn <tg-arn>

# 4. Recent logs
aws logs tail /ecs/my-app --since 10m

# 5. Recent CloudWatch alarms
aws cloudwatch describe-alarms --state-value ALARM
```

### Where to Look First

| Symptom | First Check |
|---|---|
| Task never starts | Service events → task placement errors |
| Task starts then dies | CloudWatch logs → app crash trace |
| App running but unreachable | Target Group health + Security Groups |
| Slow deployments | Health check grace period + circuit breaker |
| Auto-scaling silent | Scalable target registered? Alarm firing? |
| Weird behavior | ECS Exec into the container and inspect |

---

## 🔗 Related Files

- **[README.md](README.md)** — Architecture & concepts
- **[what-why-how.md](what-why-how.md)** — Conceptual "why" for every feature
- **[commands-cheatsheet.md](commands-cheatsheet.md)** — Full CLI reference
- **[hands-on-labs.md](hands-on-labs.md)** — Step-by-step deployment walkthroughs

---

> 💡 **Pro Tip:** Whenever ECS misbehaves, look at the **Service Events** first — they're the concise, human-readable log of *what ECS is trying to do* and *why it's failing*.
