# 🧾 Commands Cheatsheet — AWS Lambda

Quick reference for everything used in [`README.md`](./README.md) and [`hands-on-labs.md`](./hands-on-labs.md).

---

## 1. AWS CLI — Function Lifecycle

```bash
# Create a function
aws lambda create-function \
  --function-name ProcessDataFunction \
  --runtime python3.13 \
  --architectures arm64 \
  --role arn:aws:iam::<ACCOUNT_ID>:role/lambda-execution-role \
  --handler app.lambda_handler \
  --zip-file fileb://function.zip \
  --memory-size 1536 \
  --timeout 30

# Invoke a function
aws lambda invoke \
  --function-name ProcessDataFunction \
  --payload '{"key": "value"}' \
  --cli-binary-format raw-in-base64-out \
  response.json

# Get function config
aws lambda get-function-configuration --function-name ProcessDataFunction

# Get full function details (config + code location)
aws lambda get-function --function-name ProcessDataFunction

# List all functions
aws lambda list-functions

# Update function code (new zip)
aws lambda update-function-code \
  --function-name ProcessDataFunction \
  --zip-file fileb://function.zip

# Update configuration (memory, timeout, env vars)
aws lambda update-function-configuration \
  --function-name ProcessDataFunction \
  --memory-size 2048 \
  --timeout 60 \
  --environment "Variables={DB_HOST=production-db.cluster.internal,LOG_LEVEL=INFO}"

# Delete a function
aws lambda delete-function --function-name ProcessDataFunction
```

---

## 2. Versions & Aliases

```bash
# Publish an immutable version
aws lambda publish-version --function-name ProcessDataFunction

# Create an alias pointing to a version (e.g. "prod")
aws lambda create-alias \
  --function-name ProcessDataFunction \
  --name prod \
  --function-version 3

# Shift traffic gradually between versions (canary)
aws lambda update-alias \
  --function-name ProcessDataFunction \
  --name prod \
  --function-version 4 \
  --routing-config AdditionalVersionWeights={"3"=0.1}
```

---

## 3. Concurrency Controls

```bash
# Reserved concurrency — caps max concurrent executions
aws lambda put-function-concurrency \
  --function-name ProcessDataFunction \
  --reserved-concurrent-executions 50

# Remove reserved concurrency
aws lambda delete-function-concurrency --function-name ProcessDataFunction

# Provisioned concurrency — keeps containers warm (eliminates cold starts)
aws lambda put-provisioned-concurrency-config \
  --function-name ProcessDataFunction \
  --qualifier prod \
  --provisioned-concurrent-executions 2

# Check current account-level concurrency limits
aws lambda get-account-settings
```

---

## 4. SnapStart

```bash
# Enable SnapStart on published versions
aws lambda update-function-configuration \
  --function-name ProcessDataFunction \
  --snap-start ApplyOn=PublishedVersions

# Publish a version to actually get a SnapStart-optimized snapshot
aws lambda publish-version --function-name ProcessDataFunction
```

---

## 5. Layers (shared dependencies)

```bash
# Publish a new layer version
aws lambda publish-layer-version \
  --layer-name shared-deps \
  --zip-file fileb://layer.zip \
  --compatible-runtimes python3.13

# Attach a layer to a function
aws lambda update-function-configuration \
  --function-name ProcessDataFunction \
  --layers arn:aws:lambda:<REGION>:<ACCOUNT_ID>:layer:shared-deps:1

# List layer versions
aws lambda list-layer-versions --layer-name shared-deps
```

---

## 6. Permissions (resource-based policy)

```bash
# Allow S3 to invoke this function
aws lambda add-permission \
  --function-name ProcessDataFunction \
  --statement-id s3-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::my-bucket

# View the resource policy
aws lambda get-policy --function-name ProcessDataFunction
```

---

## 7. VPC Configuration

```bash
aws lambda update-function-configuration \
  --function-name ProcessDataFunction \
  --vpc-config SubnetIds=subnet-abc123,subnet-def456,SecurityGroupIds=sg-0123abcd
```

---

## 8. Logs & Monitoring

```bash
# Tail live logs (requires CloudWatch Logs Insights access)
aws logs tail /aws/lambda/ProcessDataFunction --follow

# Get the last N log events
aws logs filter-log-events \
  --log-group-name /aws/lambda/ProcessDataFunction \
  --limit 20

# Filter specifically for cold-start Init Duration lines
aws logs filter-log-events \
  --log-group-name /aws/lambda/ProcessDataFunction \
  --filter-pattern "Init Duration"
```

---

## 9. SAM CLI (Infrastructure as Code)

```bash
# Scaffold a new SAM project
sam init

# Build the deployment package
sam build

# Deploy (interactive, guided first time)
sam deploy --guided

# Invoke locally, without deploying
sam local invoke ProcessDataFunction --event event.json

# Run a local API Gateway + Lambda emulator
sam local start-api

# Tail logs for a deployed stack
sam logs -n ProcessDataFunction --stack-name my-stack --tail

# Tear down the stack
sam delete
```

### Full SAM Template Reference

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Production-ready Optimized Lambda Configuration

Resources:
  ProcessDataFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: app.lambda_handler
      Runtime: python3.13

      # Architecture Optimization: ARM64 is ~20% cheaper & faster than x86
      Architectures:
        - arm64

      # Performance Tuning: 1.5GB memory provides proportional CPU boost
      MemorySize: 1536
      Timeout: 30

      # Storage expansion for large processing jobs
      EphemeralStorage:
        Size: 1024

      # Advanced: Activate stateful checkpointing for complex steps
      SnapStart:
        ApplyOn: PublishedVersions

      # Environment variables for application logic
      Environment:
        Variables:
          DB_HOST: "production-db.cluster.internal"
          LOG_LEVEL: "INFO"

      # Restrict concurrent executions to avoid overwhelming downstream DBs
      ReservedConcurrentExecutions: 50
```

---

## 10. Boto3 (Python SDK) Quick Snippets

```python
import boto3

client = boto3.client("lambda")

# Invoke a function
response = client.invoke(
    FunctionName="ProcessDataFunction",
    InvocationType="RequestResponse",  # or "Event" for async
    Payload=b'{"key": "value"}'
)
print(response["Payload"].read())

# Get current configuration
config = client.get_function_configuration(FunctionName="ProcessDataFunction")
print(config["MemorySize"], config["Timeout"])

# Update memory/timeout programmatically
client.update_function_configuration(
    FunctionName="ProcessDataFunction",
    MemorySize=2048,
    Timeout=60
)
```
