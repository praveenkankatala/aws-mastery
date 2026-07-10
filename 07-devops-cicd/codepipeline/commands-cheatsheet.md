# AWS CodePipeline — Commands & Snippets Cheat Sheet

A ready-to-copy reference of every command, buildspec pattern, appspec block, and IaC snippet you'll actually use in production.

---

## 📑 Contents

1. [AWS CLI Setup](#1-aws-cli-setup)
2. [CodePipeline CLI](#2-codepipeline-cli)
3. [CodeCommit CLI](#3-codecommit-cli)
4. [CodeBuild CLI](#4-codebuild-cli)
5. [CodeDeploy CLI](#5-codedeploy-cli)
6. [CodeStar Connections](#6-codestar-connections)
7. [S3, KMS, IAM Setup](#7-s3-kms-iam-setup)
8. [buildspec.yml — Every Pattern](#8-buildspecyml--every-pattern)
9. [appspec.yml — Every Target](#9-appspecyml--every-target)
10. [imagedefinitions.json / taskdef.json](#10-imagedefinitionsjson--taskdefjson)
11. [Notification Rules](#11-notification-rules)
12. [Pipeline JSON Structure](#12-pipeline-json-structure)
13. [IaC Snippets](#13-iac-snippets)

---

## 1. AWS CLI Setup

```bash
# Configure default profile
aws configure

# Named profile for a specific account
aws configure --profile tooling
aws configure --profile prod

# Assume a cross-account role (one-off)
aws sts assume-role \
  --role-arn arn:aws:iam::PROD_ACCT:role/PipelineDeployRole \
  --role-session-name manual-check

# Check identity
aws sts get-caller-identity
```

---

## 2. CodePipeline CLI

```bash
# List pipelines
aws codepipeline list-pipelines

# Get pipeline definition
aws codepipeline get-pipeline --name my-pipeline > my-pipeline.json

# Update pipeline from JSON
aws codepipeline update-pipeline --cli-input-json file://my-pipeline.json

# Create pipeline from JSON
aws codepipeline create-pipeline --cli-input-json file://my-pipeline.json

# Start execution manually
aws codepipeline start-pipeline-execution --name my-pipeline

# Stop execution
aws codepipeline stop-pipeline-execution \
  --pipeline-name my-pipeline \
  --pipeline-execution-id abcd-1234 \
  --abandon \
  --reason "Deploying wrong branch"

# List executions
aws codepipeline list-pipeline-executions --pipeline-name my-pipeline

# Get execution detail
aws codepipeline get-pipeline-execution \
  --pipeline-name my-pipeline \
  --pipeline-execution-id abcd-1234

# List action executions in a run
aws codepipeline list-action-executions \
  --pipeline-name my-pipeline \
  --filter pipelineExecutionId=abcd-1234

# Disable a transition (freeze deploys)
aws codepipeline disable-stage-transition \
  --pipeline-name my-pipeline \
  --stage-name Production \
  --transition-type Inbound \
  --reason "Holiday freeze"

# Re-enable
aws codepipeline enable-stage-transition \
  --pipeline-name my-pipeline \
  --stage-name Production \
  --transition-type Inbound

# Retry a failed stage
aws codepipeline retry-stage-execution \
  --pipeline-name my-pipeline \
  --stage-name Deploy \
  --pipeline-execution-id abcd-1234 \
  --retry-mode FAILED_ACTIONS

# Approve / reject a manual approval
aws codepipeline put-approval-result \
  --pipeline-name my-pipeline \
  --stage-name Approve \
  --action-name ProdApproval \
  --token <TOKEN_FROM_SNS_MSG> \
  --result summary="LGTM",status=Approved

# Rollback a stage (V2)
aws codepipeline rollback-stage \
  --pipeline-name my-pipeline \
  --stage-name Deploy \
  --target-pipeline-execution-id <previous_successful_id>

# Delete pipeline
aws codepipeline delete-pipeline --name my-pipeline
```

---

## 3. CodeCommit CLI

```bash
# Create repo
aws codecommit create-repository --repository-name my-app

# Get clone URL (HTTPS-GRC uses git-remote-codecommit helper)
aws codecommit get-repository --repository-name my-app \
  --query 'repositoryMetadata.cloneUrlHttp'

# Clone
git clone codecommit::us-east-1://my-app          # requires git-remote-codecommit
# OR
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-app

# List branches
aws codecommit list-branches --repository-name my-app
```

---

## 4. CodeBuild CLI

```bash
# List projects
aws codebuild list-projects

# Get project detail
aws codebuild batch-get-projects --names my-build-project

# Start a build (with buildspec override)
aws codebuild start-build \
  --project-name my-build-project \
  --buildspec-override file://buildspec-override.yml \
  --environment-variables-override name=DEBUG,value=true,type=PLAINTEXT

# Stop a running build
aws codebuild stop-build --id my-build-project:abc-123

# List recent builds
aws codebuild list-builds-for-project --project-name my-build-project

# Get build detail (status, phases, logs)
aws codebuild batch-get-builds --ids my-build-project:abc-123

# Tail build logs
aws logs tail /aws/codebuild/my-build-project --follow
```

**Create a CodeBuild project (JSON):**

```json
{
  "name": "my-build-project",
  "source": { "type": "CODEPIPELINE" },
  "artifacts": { "type": "CODEPIPELINE" },
  "environment": {
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/standard:7.0",
    "computeType": "BUILD_GENERAL1_MEDIUM",
    "privilegedMode": true,
    "environmentVariables": [
      { "name": "AWS_DEFAULT_REGION", "value": "us-east-1" }
    ]
  },
  "serviceRole": "arn:aws:iam::123456789012:role/CodeBuildRole",
  "timeoutInMinutes": 30,
  "cache": { "type": "LOCAL", "modes": ["LOCAL_DOCKER_LAYER_CACHE", "LOCAL_SOURCE_CACHE"] },
  "logsConfig": {
    "cloudWatchLogs": { "status": "ENABLED", "groupName": "/aws/codebuild/my-build-project" }
  }
}
```

```bash
aws codebuild create-project --cli-input-json file://build-project.json
```

---

## 5. CodeDeploy CLI

```bash
# Create application
aws deploy create-application \
  --application-name my-app \
  --compute-platform Server        # or Lambda / ECS

# Create deployment group (EC2 Auto Scaling example)
aws deploy create-deployment-group \
  --application-name my-app \
  --deployment-group-name prod \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployRole \
  --auto-scaling-groups my-prod-asg \
  --deployment-config-name CodeDeployDefault.HalfAtATime \
  --auto-rollback-configuration enabled=true,events=DEPLOYMENT_FAILURE,DEPLOYMENT_STOP_ON_ALARM \
  --alarm-configuration enabled=true,alarms=[{name=HighErrorRate}]

# Trigger deployment
aws deploy create-deployment \
  --application-name my-app \
  --deployment-group-name prod \
  --revision revisionType=S3,s3Location={bucket=my-artifacts,key=my-app.zip,bundleType=zip}

# List deployments
aws deploy list-deployments --application-name my-app --deployment-group-name prod

# Get deployment status
aws deploy get-deployment --deployment-id d-ABC12345

# Stop deployment
aws deploy stop-deployment --deployment-id d-ABC12345 --auto-rollback-enabled

# Register on-prem instance
aws deploy register-on-premises-instance \
  --instance-name onprem-web-01 \
  --iam-user-arn arn:aws:iam::123456789012:user/onprem-deploy
```

---

## 6. CodeStar Connections

```bash
# Create pending connection
aws codestar-connections create-connection \
  --provider-type GitHub \
  --connection-name my-github-conn

# List connections
aws codestar-connections list-connections

# Get one
aws codestar-connections get-connection \
  --connection-arn arn:aws:codestar-connections:...

# Delete
aws codestar-connections delete-connection \
  --connection-arn arn:aws:codestar-connections:...
```

*Complete the OAuth handshake in the AWS Console after `create-connection` — the connection stays in `PENDING` until you do.*

---

## 7. S3, KMS, IAM Setup

### 7.1 Artifact Bucket

```bash
aws s3api create-bucket --bucket my-pipeline-artifacts-usea1 --region us-east-1
aws s3api put-bucket-versioning \
  --bucket my-pipeline-artifacts-usea1 \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption \
  --bucket my-pipeline-artifacts-usea1 \
  --server-side-encryption-configuration '{
    "Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms","KMSMasterKeyID":"alias/pipeline-key"}}]
  }'
```

### 7.2 KMS Key for Cross-Account

```bash
aws kms create-key \
  --description "CodePipeline cross-account artifact key" \
  --policy file://kms-policy.json
aws kms create-alias --alias-name alias/pipeline-key --target-key-id <key-id>
```

**kms-policy.json:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TootingAccountRoot",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::TOOLING_ACCT:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowTargetAccounts",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::STAGE_ACCT:role/PipelineDeployRole",
          "arn:aws:iam::PROD_ACCT:role/PipelineDeployRole"
        ]
      },
      "Action": ["kms:Decrypt", "kms:DescribeKey", "kms:GenerateDataKey"],
      "Resource": "*"
    }
  ]
}
```

### 7.3 Pipeline Service Role

**Trust policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "codepipeline.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

**Inline permissions (minimum viable):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject","s3:PutObject","s3:GetObjectVersion","s3:GetBucketVersioning"],
      "Resource": ["arn:aws:s3:::my-pipeline-artifacts-usea1","arn:aws:s3:::my-pipeline-artifacts-usea1/*"]
    },
    {
      "Effect": "Allow",
      "Action": ["kms:Decrypt","kms:GenerateDataKey"],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/*"
    },
    {
      "Effect": "Allow",
      "Action": ["codebuild:BatchGetBuilds","codebuild:StartBuild","codebuild:StopBuild"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["codedeploy:CreateDeployment","codedeploy:GetDeployment","codedeploy:GetApplication","codedeploy:GetApplicationRevision","codedeploy:RegisterApplicationRevision","codedeploy:GetDeploymentConfig"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["cloudformation:CreateStack","cloudformation:UpdateStack","cloudformation:DescribeStacks","cloudformation:DescribeStackEvents","cloudformation:DeleteStack","cloudformation:CreateChangeSet","cloudformation:ExecuteChangeSet","cloudformation:DescribeChangeSet"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": [
        "arn:aws:iam::STAGE_ACCT:role/PipelineDeployRole",
        "arn:aws:iam::PROD_ACCT:role/PipelineDeployRole"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["codestar-connections:UseConnection"],
      "Resource": "arn:aws:codestar-connections:us-east-1:123456789012:connection/*"
    },
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:us-east-1:123456789012:PipelineApprovals"
    }
  ]
}
```

---

## 8. buildspec.yml — Every Pattern

### 8.1 Node.js / React

```yaml
version: 0.2
phases:
  install:
    runtime-versions: { nodejs: 20 }
    commands:
      - npm ci
  pre_build:
    commands:
      - npm run lint
      - npm test -- --watchAll=false
  build:
    commands:
      - npm run build
artifacts:
  files: ['**/*']
  base-directory: build
cache:
  paths: ['node_modules/**/*']
```

### 8.2 Python

```yaml
version: 0.2
phases:
  install:
    runtime-versions: { python: 3.12 }
    commands:
      - pip install -r requirements.txt -t ./package
  pre_build:
    commands:
      - pytest --junitxml=test-results.xml
  build:
    commands:
      - cd package && zip -r ../function.zip . && cd ..
      - zip -g function.zip lambda_function.py
reports:
  pytest_reports:
    files: [test-results.xml]
    file-format: JUNITXML
artifacts:
  files: [function.zip]
```

### 8.3 Java / Maven

```yaml
version: 0.2
phases:
  install:
    runtime-versions: { java: corretto21 }
  build:
    commands:
      - mvn -B clean verify
reports:
  surefire_reports:
    files: ['**/target/surefire-reports/*.xml']
    file-format: JUNITXML
artifacts:
  files: ['target/*.jar']
  discard-paths: yes
cache:
  paths: ['/root/.m2/**/*']
```

### 8.4 Docker Build + Push to ECR

```yaml
version: 0.2
env:
  variables:
    AWS_DEFAULT_REGION: us-east-1
    IMAGE_REPO_NAME: my-app
    IMAGE_TAG: latest
  exported-variables:
    - IMAGE_TAG
phases:
  pre_build:
    commands:
      - echo "Logging in to Amazon ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
  build:
    commands:
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  post_build:
    commands:
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
      - printf '[{"name":"web","imageUri":"%s"}]' $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG > imagedefinitions.json
artifacts:
  files: [imagedefinitions.json]
```

*Requires `privilegedMode: true` on the CodeBuild project.*

### 8.5 Multi-Artifact Output (for CodeDeploy Blue/Green ECS)

```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - IMAGE_TAG=$(date +%Y%m%d-%H%M%S)-$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
  build:
    commands:
      - docker build -t $ECR_URI:$IMAGE_TAG .
      - docker push $ECR_URI:$IMAGE_TAG
  post_build:
    commands:
      - sed -i "s|<IMAGE_URI>|$ECR_URI:$IMAGE_TAG|g" taskdef.json
      - cat appspec.yaml
artifacts:
  secondary-artifacts:
    BuildArtifact:
      files: [appspec.yaml, taskdef.json]
    ImageArtifact:
      files: [imagedefinitions.json]
```

### 8.6 Secrets & Parameters

```yaml
version: 0.2
env:
  parameter-store:
    DB_HOST: /prod/db/host
    DB_PORT: /prod/db/port
  secrets-manager:
    DB_PASSWORD: prod/db:password
    STRIPE_KEY: prod/stripe:api_key
phases:
  build:
    commands:
      - echo "Connecting to $DB_HOST:$DB_PORT"
      - ./scripts/run-migrations.sh
```

---

## 9. appspec.yml — Every Target

### 9.1 EC2 / On-Prem

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html
file_exists_behavior: OVERWRITE
permissions:
  - object: /var/www/html
    pattern: "**"
    owner: apache
    group: apache
    mode: 644
hooks:
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 60
      runas: root
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 120
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 180
      runas: root
  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 60
      runas: root
```

### 9.2 EC2 Windows

```yaml
version: 0.0
os: windows
files:
  - source: \index.html
    destination: c:\inetpub\wwwroot
hooks:
  BeforeInstall:
    - location: scripts\install.ps1
      timeout: 300
  ApplicationStart:
    - location: scripts\start.ps1
      timeout: 180
  ValidateService:
    - location: scripts\validate.ps1
      timeout: 60
```

### 9.3 ECS Blue/Green

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "web-container"
          ContainerPort: 8080
        PlatformVersion: "LATEST"
Hooks:
  - BeforeInstall:         "arn:aws:lambda:us-east-1:123456789012:function:pre-install"
  - AfterInstall:          "arn:aws:lambda:us-east-1:123456789012:function:post-install"
  - AfterAllowTestTraffic: "arn:aws:lambda:us-east-1:123456789012:function:run-e2e-tests"
  - BeforeAllowTraffic:    "arn:aws:lambda:us-east-1:123456789012:function:smoke-test"
  - AfterAllowTraffic:     "arn:aws:lambda:us-east-1:123456789012:function:notify-slack"
```

### 9.4 Lambda

```yaml
version: 0.0
Resources:
  - MyLambdaFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: "my-processor-function"
        Alias: "live"
        CurrentVersion: "1"
        TargetVersion: "2"
Hooks:
  - PreTraffic:  "arn:aws:lambda:us-east-1:123456789012:function:BeforeTrafficCheck"
  - PostTraffic: "arn:aws:lambda:us-east-1:123456789012:function:AfterTrafficCheck"
```

### 9.5 EC2 Lifecycle Scripts

**scripts/install_dependencies.sh**
```bash
#!/bin/bash
set -e
sudo apt-get update -y
sudo apt-get install -y apache2
sudo rm -rf /var/www/html/*
```

**scripts/start_server.sh**
```bash
#!/bin/bash
set -e
sudo systemctl enable apache2
sudo systemctl restart apache2
```

**scripts/validate_service.sh**
```bash
#!/bin/bash
set -e
for i in {1..10}; do
  CODE=$(curl -o /dev/null -s -w "%{http_code}" http://localhost:80/)
  if [ "$CODE" -eq 200 ]; then
    echo "Server healthy (HTTP $CODE)"
    exit 0
  fi
  echo "Attempt $i: HTTP $CODE, retrying..."
  sleep 3
done
echo "Server never became healthy"
exit 1
```

**scripts/stop_server.sh**
```bash
#!/bin/bash
if systemctl is-active --quiet apache2; then
  sudo systemctl stop apache2
fi
exit 0
```

---

## 10. imagedefinitions.json / taskdef.json

### 10.1 imagedefinitions.json (Standard ECS)

```json
[
  { "name": "web-container", "imageUri": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:abc1234" }
]
```

### 10.2 taskdef.json (Blue/Green ECS)

```json
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "web-container",
      "image": "<IMAGE1_NAME>",
      "essential": true,
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "web"
        }
      }
    }
  ]
}
```

The `CodeDeployToECS` action replaces `<IMAGE1_NAME>` at deploy time.

---

## 11. Notification Rules

```bash
# Create SNS topic first
aws sns create-topic --name PipelineApprovals

# Subscribe
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:PipelineApprovals \
  --protocol email --notification-endpoint devops@example.com

# Attach notification rule (event IDs at bottom)
aws codestar-notifications create-notification-rule \
  --name my-pipeline-notifs \
  --resource arn:aws:codepipeline:us-east-1:123456789012:my-pipeline \
  --event-type-ids \
      codepipeline-pipeline-pipeline-execution-succeeded \
      codepipeline-pipeline-pipeline-execution-failed \
      codepipeline-pipeline-manual-approval-needed \
      codepipeline-pipeline-manual-approval-succeeded \
      codepipeline-pipeline-manual-approval-failed \
  --targets TargetType=SNS,TargetAddress=arn:aws:sns:us-east-1:123456789012:PipelineApprovals \
  --detail-type FULL
```

Common event type IDs:
- `codepipeline-pipeline-pipeline-execution-started`
- `codepipeline-pipeline-pipeline-execution-succeeded`
- `codepipeline-pipeline-pipeline-execution-failed`
- `codepipeline-pipeline-stage-execution-started`
- `codepipeline-pipeline-stage-execution-succeeded`
- `codepipeline-pipeline-stage-execution-failed`
- `codepipeline-pipeline-manual-approval-needed`
- `codepipeline-pipeline-manual-approval-succeeded`
- `codepipeline-pipeline-manual-approval-failed`

---

## 12. Pipeline JSON Structure

Minimal V2 pipeline JSON:

```json
{
  "pipeline": {
    "name": "my-pipeline",
    "roleArn": "arn:aws:iam::123456789012:role/CodePipelineServiceRole",
    "pipelineType": "V2",
    "executionMode": "SUPERSEDED",
    "artifactStore": {
      "type": "S3",
      "location": "my-pipeline-artifacts-usea1",
      "encryptionKey": {
        "id": "arn:aws:kms:us-east-1:123456789012:alias/pipeline-key",
        "type": "KMS"
      }
    },
    "stages": [
      {
        "name": "Source",
        "actions": [{
          "name": "Source",
          "actionTypeId": {
            "category": "Source", "owner": "AWS",
            "provider": "CodeStarSourceConnection", "version": "1"
          },
          "configuration": {
            "ConnectionArn": "arn:aws:codestar-connections:...",
            "FullRepositoryId": "my-org/my-repo",
            "BranchName": "main"
          },
          "outputArtifacts": [{ "name": "SourceOutput" }]
        }]
      },
      {
        "name": "Build",
        "actions": [{
          "name": "Build",
          "actionTypeId": {
            "category": "Build", "owner": "AWS",
            "provider": "CodeBuild", "version": "1"
          },
          "configuration": { "ProjectName": "my-build-project" },
          "inputArtifacts": [{ "name": "SourceOutput" }],
          "outputArtifacts": [{ "name": "BuildOutput" }]
        }]
      },
      {
        "name": "Deploy",
        "actions": [{
          "name": "Deploy",
          "actionTypeId": {
            "category": "Deploy", "owner": "AWS",
            "provider": "CloudFormation", "version": "1"
          },
          "configuration": {
            "ActionMode": "CREATE_UPDATE",
            "StackName": "my-app-stack",
            "TemplatePath": "BuildOutput::template.yaml",
            "RoleArn": "arn:aws:iam::123456789012:role/CFNDeployRole",
            "Capabilities": "CAPABILITY_NAMED_IAM"
          },
          "inputArtifacts": [{ "name": "BuildOutput" }]
        }]
      }
    ],
    "triggers": [
      {
        "providerType": "CodeStarSourceConnection",
        "gitConfiguration": {
          "sourceActionName": "Source",
          "push": [{
            "branches": { "includes": ["main"] },
            "filePaths": { "includes": ["src/**"] }
          }]
        }
      }
    ]
  }
}
```

---

## 13. IaC Snippets

### 13.1 CloudFormation — Full Working Pipeline

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Parameters:
  GitHubConnectionArn: { Type: String }
  RepoId:              { Type: String, Default: 'my-org/my-repo' }
  Branch:              { Type: String, Default: 'main' }

Resources:
  ArtifactBucket:
    Type: AWS::S3::Bucket
    Properties:
      VersioningConfiguration: { Status: Enabled }
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault: { SSEAlgorithm: aws:kms }

  BuildRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: { Service: codebuild.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonS3FullAccess
        - arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

  BuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: my-build
      ServiceRole: !GetAtt BuildRole.Arn
      Source: { Type: CODEPIPELINE }
      Artifacts: { Type: CODEPIPELINE }
      Environment:
        Type: LINUX_CONTAINER
        Image: aws/codebuild/standard:7.0
        ComputeType: BUILD_GENERAL1_MEDIUM

  PipelineRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: { Service: codepipeline.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AWSCodePipeline_FullAccess

  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      RoleArn: !GetAtt PipelineRole.Arn
      PipelineType: V2
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactBucket
      Stages:
        - Name: Source
          Actions:
            - Name: Source
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: CodeStarSourceConnection
                Version: '1'
              Configuration:
                ConnectionArn: !Ref GitHubConnectionArn
                FullRepositoryId: !Ref RepoId
                BranchName: !Ref Branch
              OutputArtifacts: [{ Name: SourceOutput }]
        - Name: Build
          Actions:
            - Name: Build
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration: { ProjectName: !Ref BuildProject }
              InputArtifacts:  [{ Name: SourceOutput }]
              OutputArtifacts: [{ Name: BuildOutput }]
```

### 13.2 CDK (TypeScript) — Cross-Account Ready

```typescript
import * as cdk from 'aws-cdk-lib';
import * as pipelines from 'aws-cdk-lib/pipelines';
import { Construct } from 'constructs';

export class MyPipelineStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const pipeline = new pipelines.CodePipeline(this, 'Pipeline', {
      pipelineName: 'WebServicePipeline',
      crossAccountKeys: true,
      synth: new pipelines.ShellStep('Synth', {
        input: pipelines.CodePipelineSource.connection('my-org/my-repo', 'main', {
          connectionArn: 'arn:aws:codestar-connections:us-east-1:123456789012:connection/xyz',
        }),
        commands: ['npm ci', 'npm run build', 'npx cdk synth'],
      }),
    });

    pipeline.addStage(new MyAppStage(this, 'Stage', {
      env: { account: 'STAGE_ACCT', region: 'us-east-1' },
    }));

    pipeline.addStage(new MyAppStage(this, 'Prod', {
      env: { account: 'PROD_ACCT', region: 'us-east-1' },
    }), {
      pre: [new pipelines.ManualApprovalStep('PromoteToProd')],
    });
  }
}
```

### 13.3 Terraform — CodeBuild Project

```hcl
resource "aws_codebuild_project" "app" {
  name         = "my-build"
  service_role = aws_iam_role.codebuild.arn
  build_timeout = 30

  artifacts { type = "CODEPIPELINE" }
  source    { type = "CODEPIPELINE" }

  environment {
    compute_type    = "BUILD_GENERAL1_MEDIUM"
    image           = "aws/codebuild/standard:7.0"
    type            = "LINUX_CONTAINER"
    privileged_mode = true
    environment_variable {
      name  = "AWS_DEFAULT_REGION"
      value = "us-east-1"
    }
  }

  cache {
    type  = "LOCAL"
    modes = ["LOCAL_DOCKER_LAYER_CACHE", "LOCAL_SOURCE_CACHE"]
  }

  logs_config {
    cloudwatch_logs {
      status      = "ENABLED"
      group_name  = "/aws/codebuild/my-build"
    }
  }
}
```

---

*See [hands-on-labs.md](./hands-on-labs.md) to put these into practice.*
