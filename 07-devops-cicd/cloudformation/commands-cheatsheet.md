# AWS CloudFormation — CLI Commands Cheat Sheet

All commands use AWS CLI v2. Add `--region <region>` or `--profile <profile>` to any command as needed.

---

## 1. Template Validation & Linting

```bash
# Validate template syntax (local file)
aws cloudformation validate-template --template-body file://template.yaml

# Validate template stored in S3
aws cloudformation validate-template \
  --template-url https://s3.amazonaws.com/my-bucket/template.yaml

# Deep static analysis (install once: pip install cfn-lint)
cfn-lint template.yaml
cfn-lint templates/*.yaml --format pretty

# Policy-as-code checks with CloudFormation Guard
cfn-guard validate --data template.yaml --rules rules.guard
```

---

## 2. Stack Lifecycle — Create / Update / Delete

### Create

```bash
# Basic create
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml

# Create with parameters, tags, and IAM capability
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=EnvType,ParameterValue=dev \
               ParameterKey=KeyName,ParameterValue=my-key \
  --tags Key=Project,Value=Demo Key=Owner,Value=Tharun \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM

# Keep failed resources for debugging (no rollback)
aws cloudformation create-stack --stack-name my-stack \
  --template-body file://template.yaml \
  --on-failure DO_NOTHING          # or DELETE | ROLLBACK (default)

# Create with a service role (least-privilege operations)
aws cloudformation create-stack --stack-name my-stack \
  --template-body file://template.yaml \
  --role-arn arn:aws:iam::123456789012:role/cfn-service-role

# Create with termination protection + timeout + SNS notifications
aws cloudformation create-stack --stack-name my-stack \
  --template-body file://template.yaml \
  --enable-termination-protection \
  --timeout-in-minutes 30 \
  --notification-arns arn:aws:sns:ap-south-1:123456789012:cfn-events

# Template stored in S3 (required when > 51,200 bytes)
aws cloudformation create-stack --stack-name my-stack \
  --template-url https://s3.amazonaws.com/my-bucket/template.yaml

# Pass parameters from a JSON file
aws cloudformation create-stack --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters file://params-dev.json
# params-dev.json:
# [ {"ParameterKey":"EnvType","ParameterValue":"dev"} ]
```

### Update

```bash
# Direct update (prefer change sets in production!)
aws cloudformation update-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=EnvType,UsePreviousValue=true \
  --capabilities CAPABILITY_NAMED_IAM

# Update only parameters, keep the current template
aws cloudformation update-stack --stack-name my-stack \
  --use-previous-template \
  --parameters ParameterKey=InstanceType,ParameterValue=t3.small
```

### Delete

```bash
aws cloudformation delete-stack --stack-name my-stack

# Delete but keep specific resources (only from DELETE_FAILED state)
aws cloudformation delete-stack --stack-name my-stack \
  --retain-resources LogicalId1 LogicalId2

# Disable termination protection first if enabled
aws cloudformation update-termination-protection \
  --stack-name my-stack --no-enable-termination-protection
```

### Waiters (block until done — great for scripts/CI)

```bash
aws cloudformation wait stack-create-complete  --stack-name my-stack
aws cloudformation wait stack-update-complete  --stack-name my-stack
aws cloudformation wait stack-delete-complete  --stack-name my-stack
aws cloudformation wait change-set-create-complete \
  --stack-name my-stack --change-set-name my-changes
```

---

## 3. Change Sets — preview before you commit

```bash
# 1. Create a change set for an existing stack
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name add-alarm \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_IAM

# (for a NEW stack use: --change-set-type CREATE)

# 2. Review it — look for "Replacement": "True"!
aws cloudformation describe-change-set \
  --stack-name my-stack --change-set-name add-alarm \
  --query "Changes[].ResourceChange.{Action:Action,Resource:LogicalResourceId,Replace:Replacement}" \
  --output table

# 3a. Apply
aws cloudformation execute-change-set \
  --stack-name my-stack --change-set-name add-alarm

# 3b. ...or discard
aws cloudformation delete-change-set \
  --stack-name my-stack --change-set-name add-alarm

# List all change sets on a stack
aws cloudformation list-change-sets --stack-name my-stack
```

---

## 4. `deploy` & `package` — the CI/CD workhorses

```bash
# deploy = create-or-update + change set in one idempotent command
aws cloudformation deploy \
  --stack-name my-stack \
  --template-file template.yaml \
  --parameter-overrides EnvType=prod InstanceType=t3.small \
  --capabilities CAPABILITY_NAMED_IAM \
  --tags Project=Demo \
  --no-fail-on-empty-changeset

# package = upload local artifacts (Lambda zips, nested templates) to S3
# and rewrite local paths into S3 URLs
aws cloudformation package \
  --template-file template.yaml \
  --s3-bucket my-artifacts-bucket \
  --output-template-file packaged.yaml

aws cloudformation deploy --stack-name my-app \
  --template-file packaged.yaml --capabilities CAPABILITY_IAM
```

---

## 5. Inspect Stacks, Resources & Events

```bash
# List stacks (filter out deleted ones)
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE ROLLBACK_COMPLETE

# Full stack detail (status, parameters, outputs, tags)
aws cloudformation describe-stacks --stack-name my-stack

# Just the outputs
aws cloudformation describe-stacks --stack-name my-stack \
  --query "Stacks[0].Outputs" --output table

# Just the status
aws cloudformation describe-stacks --stack-name my-stack \
  --query "Stacks[0].StackStatus" --output text

# Event history — your #1 debugging tool (newest first)
aws cloudformation describe-stack-events --stack-name my-stack \
  --query "StackEvents[].{Time:Timestamp,Status:ResourceStatus,Type:ResourceType,Reason:ResourceStatusReason}" \
  --output table

# Only the failures
aws cloudformation describe-stack-events --stack-name my-stack \
  --query "StackEvents[?contains(ResourceStatus,'FAILED')].[LogicalResourceId,ResourceStatusReason]" \
  --output table

# Resources in a stack
aws cloudformation list-stack-resources --stack-name my-stack
aws cloudformation describe-stack-resource \
  --stack-name my-stack --logical-resource-id MyBucket

# Retrieve the deployed template
aws cloudformation get-template --stack-name my-stack \
  --query TemplateBody --output text

# Which stack does a resource belong to?
aws cloudformation describe-stack-resources \
  --physical-resource-id i-0123456789abcdef0

# Estimated resource list for a template before deploying
aws cloudformation get-template-summary --template-body file://template.yaml
```

---

## 6. Exports & Cross-Stack References

```bash
# List all exports in the region
aws cloudformation list-exports --output table

# Which stacks import a given export? (must be empty before you can remove it)
aws cloudformation list-imports --export-name network-stack-VpcId
```

---

## 7. Drift Detection

```bash
# Start drift detection on the whole stack
aws cloudformation detect-stack-drift --stack-name my-stack
# → returns a StackDriftDetectionId

# Check detection progress/result
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <id>

# Per-resource drift details
aws cloudformation describe-stack-resource-drifts --stack-name my-stack \
  --stack-resource-drift-status-filters MODIFIED DELETED

# Single-resource drift
aws cloudformation detect-stack-resource-drift \
  --stack-name my-stack --logical-resource-id WebSecurityGroup
```

---

## 8. Stack Policies (protect resources from updates)

```bash
# Apply a stack policy
aws cloudformation set-stack-policy --stack-name my-stack \
  --stack-policy-body file://policy.json

# View current policy
aws cloudformation get-stack-policy --stack-name my-stack

# Temporarily override during one update
aws cloudformation update-stack --stack-name my-stack \
  --use-previous-template \
  --stack-policy-during-update-body file://allow-all.json
```

Example `policy.json` — deny any update action on the database:

```json
{
  "Statement": [
    { "Effect": "Allow", "Action": "Update:*", "Principal": "*", "Resource": "*" },
    { "Effect": "Deny",  "Action": "Update:*", "Principal": "*",
      "Resource": "LogicalResourceId/ProdDatabase" }
  ]
}
```

---

## 9. Resource Import (adopt existing resources)

```bash
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name import-bucket \
  --change-set-type IMPORT \
  --template-body file://template.yaml \
  --resources-to-import file://import.json

# import.json:
# [{
#   "ResourceType": "AWS::S3::Bucket",
#   "LogicalResourceId": "LegacyBucket",
#   "ResourceIdentifier": { "BucketName": "my-existing-bucket" }
# }]

aws cloudformation execute-change-set \
  --stack-name my-stack --change-set-name import-bucket
```

---

## 10. StackSets — multi-account / multi-region

```bash
# Create the stack set (self-managed permissions shown)
aws cloudformation create-stack-set \
  --stack-set-name org-baseline \
  --template-body file://baseline.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# Deploy instances into accounts × regions
aws cloudformation create-stack-instances \
  --stack-set-name org-baseline \
  --accounts 111111111111 222222222222 \
  --regions ap-south-1 us-east-1 \
  --operation-preferences FailureToleranceCount=1,MaxConcurrentCount=2

# Update the stack set (rolls to all instances)
aws cloudformation update-stack-set \
  --stack-set-name org-baseline \
  --template-body file://baseline-v2.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# Inspect
aws cloudformation list-stack-sets
aws cloudformation list-stack-instances --stack-set-name org-baseline
aws cloudformation describe-stack-set-operation \
  --stack-set-name org-baseline --operation-id <id>

# Remove instances, then the set
aws cloudformation delete-stack-instances \
  --stack-set-name org-baseline \
  --accounts 111111111111 --regions ap-south-1 \
  --no-retain-stacks
aws cloudformation delete-stack-set --stack-set-name org-baseline
```

---

## 11. Recovering Broken Stacks

```bash
# Retry a stuck UPDATE_ROLLBACK_FAILED, skipping dead resources
aws cloudformation continue-update-rollback --stack-name my-stack \
  --resources-to-skip BrokenResourceLogicalId

# Cancel an in-progress update (triggers rollback)
aws cloudformation cancel-update-stack --stack-name my-stack

# Roll a stack back to its last stable state (newer API)
aws cloudformation rollback-stack --stack-name my-stack
```

---

## 12. cfn Helper Scripts (run ON the EC2 instance)

```bash
# Apply AWS::CloudFormation::Init metadata
/opt/aws/bin/cfn-init -v \
  --stack <stack-name> --resource <logical-id> --region <region> \
  --configsets default

# Signal success (exit code of previous command) back to CloudFormation
/opt/aws/bin/cfn-signal -e $? \
  --stack <stack-name> --resource <logical-id> --region <region>

# Signal a WaitConditionHandle URL instead
/opt/aws/bin/cfn-signal -e 0 "<pre-signed-handle-url>"

# View metadata (debugging)
/opt/aws/bin/cfn-get-metadata \
  --stack <stack-name> --resource <logical-id> --region <region>

# cfn-hup: re-run cfn-init when metadata changes (config lives in
# /etc/cfn/cfn-hup.conf and /etc/cfn/hooks.d/cfn-auto-reloader.conf)
systemctl status cfn-hup
```

Bootstrap log locations on the instance: `/var/log/cfn-init.log`, `/var/log/cfn-init-cmd.log`, `/var/log/cloud-init-output.log`.

---

## 13. Handy One-Liners

```bash
# Delete ALL stacks matching a prefix (⚠️ dangerous — sandbox only)
for s in $(aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[?starts_with(StackName,'lab-')].StackName" --output text); do
  aws cloudformation delete-stack --stack-name "$s"
done

# Export a single output value into a shell variable
VPC_ID=$(aws cloudformation describe-stacks --stack-name network \
  --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue" --output text)

# Watch stack events live (poll every 5s)
watch -n 5 "aws cloudformation describe-stack-events --stack-name my-stack \
  --query 'StackEvents[0:8].[Timestamp,LogicalResourceId,ResourceStatus]' --output table"

# Detect account-wide: list stacks not updated in 90+ days
aws cloudformation list-stacks --stack-status-filter UPDATE_COMPLETE CREATE_COMPLETE \
  --query "StackSummaries[?LastUpdatedTime<'$(date -d '90 days ago' -Iseconds)'].StackName"
```

---

## 14. Capability Flags — quick reference

| Flag | Required when the template… |
|---|---|
| `CAPABILITY_IAM` | creates IAM resources (unnamed) |
| `CAPABILITY_NAMED_IAM` | creates IAM resources **with custom names** |
| `CAPABILITY_AUTO_EXPAND` | uses macros/transforms (SAM, Include, LanguageExtensions) without a change-set review |
