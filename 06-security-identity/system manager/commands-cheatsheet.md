# AWS SSM — Commands Cheatsheet

Quick-reference AWS CLI commands, grouped by the same pillars used in [README.md](./README.md). Replace placeholders like `<instance-id>`, `<region>`, `<name>` with your own values.

> Prerequisite for all commands: AWS CLI v2 installed and configured (`aws configure`), with an IAM identity that has SSM permissions.

---

## Table of Contents

1. [Prerequisites & Setup](#prerequisites--setup)
2. [Fleet / Inventory](#fleet--inventory)
3. [Parameter Store](#parameter-store)
4. [Session Manager](#session-manager)
5. [Run Command](#run-command)
6. [Patch Manager](#patch-manager)
7. [Automation](#automation)
8. [Change Manager](#change-manager)
9. [OpsCenter / Incident Manager](#opscenter--incident-manager)
10. [AppConfig](#appconfig)
11. [Secrets Manager (for comparison)](#secrets-manager-for-comparison)

---

## Prerequisites & Setup

```bash
# Attach the AWS-managed policy that lets an instance talk to SSM
aws iam attach-role-policy \
  --role-name <your-ec2-role-name> \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Check SSM Agent status on the instance itself (Linux)
sudo systemctl status amazon-ssm-agent

# Check SSM Agent status (Windows, PowerShell)
Get-Service AmazonSSMAgent

# Register an on-prem/hybrid server (creates an activation)
aws ssm create-activation \
  --default-instance-name "on-prem-server-01" \
  --iam-role <ssm-service-role> \
  --registration-limit 5

# Create a VPC endpoint for SSM (for isolated private subnets)
aws ec2 create-vpc-endpoint \
  --vpc-id <vpc-id> \
  --service-name com.amazonaws.<region>.ssm \
  --vpc-endpoint-type Interface \
  --subnet-ids <subnet-id> \
  --security-group-ids <sg-id>
# Repeat for com.amazonaws.<region>.ssmmessages and ec2messages
```

---

## Fleet / Inventory

```bash
# List all instances registered as SSM Managed Nodes
aws ssm describe-instance-information

# Check ping status of a specific managed node
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=<instance-id>"

# List inventory data (installed software, network config, etc.)
aws ssm list-inventory-entries \
  --instance-id <instance-id> \
  --type-name AWS:Application
```

---

## Parameter Store

```bash
# Create a plaintext parameter
aws ssm put-parameter \
  --name "/prod/app/log-level" \
  --value "INFO" \
  --type String

# Create an encrypted (SecureString) parameter
aws ssm put-parameter \
  --name "/prod/db/password" \
  --value "SuperSecret123!" \
  --type SecureString \
  --key-id alias/aws/ssm

# Get a single parameter (decrypted)
aws ssm get-parameter \
  --name "/prod/db/password" \
  --with-decryption

# Get all parameters under a path
aws ssm get-parameters-by-path \
  --path "/prod/" \
  --recursive \
  --with-decryption

# Update an existing parameter (overwrite)
aws ssm put-parameter \
  --name "/prod/app/log-level" \
  --value "DEBUG" \
  --type String \
  --overwrite

# Delete a parameter
aws ssm delete-parameter --name "/prod/app/log-level"

# List parameter history
aws ssm get-parameter-history --name "/prod/db/password" --with-decryption
```

---

## Session Manager

```bash
# Install the Session Manager plugin (macOS example)
brew install --cask session-manager-plugin

# Start an interactive session
aws ssm start-session --target <instance-id>

# Start a session with a specific document (e.g. port forwarding)
aws ssm start-session \
  --target <instance-id> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3306"],"localPortNumber":["9999"]}'

# Terminate a session
aws ssm terminate-session --session-id <session-id>

# List active sessions
aws ssm describe-sessions --state Active
```

---

## Run Command

```bash
# Run a shell command on one or more instances
aws ssm send-command \
  --instance-ids "<instance-id-1>" "<instance-id-2>" \
  --document-name "AWS-RunShellScript" \
  --parameters commands="df -h"

# Run against a tag-based target (entire fleet)
aws ssm send-command \
  --targets "Key=tag:Environment,Values=Production" \
  --document-name "AWS-RunShellScript" \
  --parameters commands="sudo yum update -y"

# Check the status of a command invocation
aws ssm list-command-invocations \
  --command-id <command-id> \
  --details

# Get output for one specific instance
aws ssm get-command-invocation \
  --command-id <command-id> \
  --instance-id <instance-id>
```

---

## Patch Manager

```bash
# Create a custom patch baseline
aws ssm create-patch-baseline \
  --name "Prod-Critical-Baseline" \
  --operating-system AMAZON_LINUX_2 \
  --approval-rules "PatchRules=[{PatchFilterGroup={PatchFilters=[{Key=CLASSIFICATION,Values=[Security]},{Key=SEVERITY,Values=[Critical]}]},ApproveAfterDays=0}]"

# Register the baseline with a patch group
aws ssm register-patch-baseline-for-patch-group \
  --baseline-id <baseline-id> \
  --patch-group "Production"

# Trigger an on-demand patch scan/install
aws ssm send-command \
  --document-name "AWS-RunPatchBaseline" \
  --targets "Key=tag:PatchGroup,Values=Production" \
  --parameters "Operation=Install"

# Check patch compliance state
aws ssm describe-instance-patch-states \
  --instance-ids <instance-id>

# List missing patches for an instance
aws ssm describe-instance-patches --instance-id <instance-id>
```

---

## Automation

```bash
# List available automation documents
aws ssm list-documents --document-filter-list "key=DocumentType,value=Automation"

# Start an automation execution (e.g. build an AMI)
aws ssm start-automation-execution \
  --document-name "AWS-CreateImage" \
  --parameters "InstanceId=<instance-id>,NoReboot=false"

# Check execution status
aws ssm describe-automation-executions \
  --filters "Key=DocumentNamePrefix,Values=AWS-CreateImage"

# Get detailed step-by-step execution output
aws ssm get-automation-execution \
  --automation-execution-id <execution-id>
```

---

## Change Manager

```bash
# Create a change template first (via console or CloudFormation is typical),
# then start a change request referencing it:
aws ssm start-change-request-execution \
  --document-name "MyChangeTemplate" \
  --runbooks "DocumentName=AWS-RunPatchBaseline,Parameters={Operation=[Install]},TargetLocation={}" \
  --scheduled-time "2026-07-10T02:00:00Z"

# List change requests
aws ssm describe-automation-executions \
  --filters "Key=AutomationType,Values=ChangeRequest"

# Approve a pending change request (as an approver)
aws ssm update-managed-instance-role --instance-id <instance-id> --iam-role <role>
# Actual approvals are typically done in-console under Change Manager
```

---

## OpsCenter / Incident Manager

```bash
# List open OpsItems
aws ssm describe-ops-items \
  --ops-item-filters "Key=Status,Values=Open,Operator=Equal"

# Create a new OpsItem
aws ssm create-ops-item \
  --title "High CPU on prod-web-01" \
  --description "CPU sustained above 90% for 15 minutes" \
  --source "CloudWatch" \
  --priority 2

# Update / resolve an OpsItem
aws ssm update-ops-item \
  --ops-item-id <ops-item-id> \
  --status Resolved

# List incident records (Incident Manager)
aws ssm-incidents list-incident-records
```

---

## AppConfig

```bash
# Create an application
aws appconfig create-application --name "MyApp"

# Create a configuration profile
aws appconfig create-configuration-profile \
  --application-id <app-id> \
  --name "feature-flags" \
  --location-uri "hosted"

# Create a hosted configuration version (the actual flag data)
aws appconfig create-hosted-configuration-version \
  --application-id <app-id> \
  --configuration-profile-id <profile-id> \
  --content-type "application/json" \
  --content file://flags.json

# Start a deployment (gradual rollout)
aws appconfig start-deployment \
  --application-id <app-id> \
  --environment-id <env-id> \
  --deployment-strategy-id <strategy-id> \
  --configuration-profile-id <profile-id> \
  --configuration-version 1
```

---

## Secrets Manager (for comparison)

```bash
# Create a secret
aws secretsmanager create-secret \
  --name "prod/db/credentials" \
  --secret-string '{"username":"admin","password":"SuperSecret123!"}'

# Retrieve a secret
aws secretsmanager get-secret-value --secret-id "prod/db/credentials"

# Enable automatic rotation (built-in — no custom Lambda needed for supported DBs)
aws secretsmanager rotate-secret \
  --secret-id "prod/db/credentials" \
  --rotation-lambda-arn <lambda-arn> \
  --rotation-rules AutomaticallyAfterDays=30

# Replicate a secret to another region
aws secretsmanager replicate-secret-to-regions \
  --secret-id "prod/db/credentials" \
  --add-replica-regions Region=us-west-2
```
