# AWS Billing & Cost Management — CLI Commands Cheatsheet

> All commands assume AWS CLI v2 with credentials configured (`aws configure`).
> **Note:** Most Cost Explorer (`ce`) API calls are **$0.01 per paginated request**. Billing APIs generally only work in `us-east-1` — add `--region us-east-1` if you see endpoint errors.

---

## Table of Contents
1. [Cost Explorer (`ce`)](#1-cost-explorer-ce)
2. [Cost Anomaly Detection (`ce`)](#2-cost-anomaly-detection-ce)
3. [AWS Budgets (`budgets`)](#3-aws-budgets-budgets)
4. [CUR Legacy (`cur`) & Data Exports (`bcm-data-exports`)](#4-cur--data-exports)
5. [Cost Allocation Tags & Cost Categories](#5-cost-allocation-tags--cost-categories)
6. [Resource Tagging (`resourcegroupstaggingapi`, `ec2`)](#6-resource-tagging)
7. [Compute Optimizer (`compute-optimizer`)](#7-compute-optimizer)
8. [Cost Optimization Hub (`cost-optimization-hub`)](#8-cost-optimization-hub)
9. [Trusted Advisor (`support`)](#9-trusted-advisor-support)
10. [Service Quotas (`service-quotas`)](#10-service-quotas)
11. [Savings Plans (`savingsplans`) & RIs](#11-savings-plans--reserved-instances)
12. [Athena Queries for CUR](#12-athena-queries-for-cur)
13. [Supporting Services (SNS, CloudWatch, EventBridge)](#13-supporting-services)

---

## 1. Cost Explorer (`ce`)

### Get cost and usage (monthly, grouped by service)
```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-06-01,End=2026-07-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" "UsageQuantity" \
  --group-by Type=DIMENSION,Key=SERVICE
```

### Daily costs for one service (EC2), filtered
```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-07-28 \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute"]}}'
```

### Group by a cost allocation tag
```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-07-28 \
  --granularity MONTHLY \
  --metrics "AmortizedCost" \
  --group-by Type=TAG,Key=Environment
```

### Filter by tag value (only Production)
```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-07-28 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --filter '{"Tags":{"Key":"Environment","Values":["Production"]}}'
```

### Group by linked account (org view)
```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY \
  --metrics "NetUnblendedCost" \
  --group-by Type=DIMENSION,Key=LINKED_ACCOUNT
```

### Cost forecast (next month, 80% confidence)
```bash
aws ce get-cost-forecast \
  --time-period Start=2026-08-01,End=2026-09-01 \
  --granularity MONTHLY \
  --metric UNBLENDED_COST \
  --prediction-interval-level 80
```

### Usage forecast
```bash
aws ce get-usage-forecast \
  --time-period Start=2026-08-01,End=2026-09-01 \
  --granularity MONTHLY \
  --metric USAGE_QUANTITY \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Simple Storage Service"]}}'
```

### List available dimension values (e.g., all services billed)
```bash
aws ce get-dimension-values \
  --time-period Start=2026-07-01,End=2026-07-28 \
  --dimension SERVICE
```

### List tag keys/values seen in billing data
```bash
aws ce get-tags \
  --time-period Start=2026-07-01,End=2026-07-28

aws ce get-tags \
  --time-period Start=2026-07-01,End=2026-07-28 \
  --tag-key Environment
```

### Rightsizing recommendations (EC2)
```bash
aws ce get-rightsizing-recommendation \
  --service "AmazonEC2" \
  --configuration '{"RecommendationTarget":"CROSS_INSTANCE_FAMILY","BenefitsConsidered":true}'
```

### RI / Savings Plans analytics
```bash
# RI utilization
aws ce get-reservation-utilization \
  --time-period Start=2026-06-01,End=2026-07-01

# RI coverage
aws ce get-reservation-coverage \
  --time-period Start=2026-06-01,End=2026-07-01

# RI purchase recommendations
aws ce get-reservation-purchase-recommendation \
  --service "Amazon Elastic Compute Cloud - Compute" \
  --lookback-period-in-days SIXTY_DAYS \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT

# Savings Plans utilization / coverage / recommendations
aws ce get-savings-plans-utilization \
  --time-period Start=2026-06-01,End=2026-07-01

aws ce get-savings-plans-coverage \
  --time-period Start=2026-06-01,End=2026-07-01

aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days THIRTY_DAYS
```

---

## 2. Cost Anomaly Detection (`ce`)

```bash
# Create a monitor across all AWS services (recommended default)
aws ce create-anomaly-monitor \
  --anomaly-monitor '{"MonitorName":"AllServicesMonitor","MonitorType":"DIMENSIONAL","MonitorDimension":"SERVICE"}'

# Create an alert subscription (immediate alerts over $50 impact)
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName":"FinOpsAlerts",
    "MonitorArnList":["<monitor-arn>"],
    "Subscribers":[{"Type":"EMAIL","Address":"finops@example.com"}],
    "Frequency":"IMMEDIATE",
    "ThresholdExpression":{"Dimensions":{"Key":"ANOMALY_TOTAL_IMPACT_ABSOLUTE","Values":["50"],"MatchOptions":["GREATER_THAN_OR_EQUAL"]}}
  }'

# List monitors / subscriptions
aws ce get-anomaly-monitors
aws ce get-anomaly-subscriptions

# Retrieve detected anomalies for a date range
aws ce get-anomalies \
  --date-interval StartDate=2026-07-01,EndDate=2026-07-28

# Provide feedback on an anomaly
aws ce provide-anomaly-feedback \
  --anomaly-id <anomaly-id> \
  --feedback PLANNED_ACTIVITY

# Delete
aws ce delete-anomaly-subscription --subscription-arn <arn>
aws ce delete-anomaly-monitor --monitor-arn <arn>
```

---

## 3. AWS Budgets (`budgets`)

```bash
# Create a monthly $1000 cost budget with 80% actual + 100% forecast alerts
aws budgets create-budget \
  --account-id 111122223333 \
  --budget '{
    "BudgetName":"Monthly-Overall",
    "BudgetLimit":{"Amount":"1000","Unit":"USD"},
    "TimeUnit":"MONTHLY",
    "BudgetType":"COST"
  }' \
  --notifications-with-subscribers '[
    {"Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"},
     "Subscribers":[{"SubscriptionType":"EMAIL","Address":"team@example.com"}]},
    {"Notification":{"NotificationType":"FORECASTED","ComparisonOperator":"GREATER_THAN","Threshold":100,"ThresholdType":"PERCENTAGE"},
     "Subscribers":[{"SubscriptionType":"SNS","Address":"arn:aws:sns:us-east-1:111122223333:budget-alerts"}]}
  ]'

# Tag-scoped budget (Production only) — CostFilters style
aws budgets create-budget \
  --account-id 111122223333 \
  --budget '{
    "BudgetName":"Prod-Only",
    "BudgetLimit":{"Amount":"500","Unit":"USD"},
    "TimeUnit":"MONTHLY",
    "BudgetType":"COST",
    "CostFilters":{"TagKeyValue":["user:Environment$Production"]}
  }'

# List / describe / delete
aws budgets describe-budgets --account-id 111122223333
aws budgets describe-budget --account-id 111122223333 --budget-name "Monthly-Overall"
aws budgets delete-budget  --account-id 111122223333 --budget-name "Monthly-Overall"

# Budget Actions — attach a deny-policy when threshold breaches
aws budgets create-budget-action \
  --account-id 111122223333 \
  --budget-name "Prod-Only" \
  --notification-type ACTUAL \
  --action-type APPLY_IAM_POLICY \
  --action-threshold ActionThresholdValue=100,ActionThresholdType=PERCENTAGE \
  --definition '{"IamActionDefinition":{"PolicyArn":"arn:aws:iam::111122223333:policy/DenyExpensiveLaunches","Roles":["Developers"]}}' \
  --execution-role-arn arn:aws:iam::111122223333:role/BudgetsActionRole \
  --approval-model AUTOMATIC \
  --subscribers SubscriptionType=EMAIL,Address=finops@example.com

aws budgets describe-budget-actions-for-account --account-id 111122223333
aws budgets execute-budget-action --account-id 111122223333 --budget-name "Prod-Only" \
  --action-id <id> --execution-type APPROVE_BUDGET_ACTION
```

---

## 4. CUR & Data Exports

### Legacy CUR (`cur`) — region must be us-east-1
```bash
aws cur put-report-definition --region us-east-1 \
  --report-definition '{
    "ReportName":"my-cur",
    "TimeUnit":"HOURLY",
    "Format":"Parquet",
    "Compression":"Parquet",
    "AdditionalSchemaElements":["RESOURCES","SPLIT_COST_ALLOCATION_DATA"],
    "S3Bucket":"my-cur-bucket",
    "S3Prefix":"cur/",
    "S3Region":"ap-south-1",
    "AdditionalArtifacts":["ATHENA"],
    "RefreshClosedReports":true,
    "ReportVersioning":"OVERWRITE_REPORT"
  }'

aws cur describe-report-definitions --region us-east-1
aws cur delete-report-definition --region us-east-1 --report-name my-cur
```

### Modern Data Exports / CUR 2.0 (`bcm-data-exports`)
```bash
# Create a CUR 2.0 export with column selection via SQL
aws bcm-data-exports create-export \
  --export '{
    "Name":"cur2-daily",
    "DataQuery":{
      "QueryStatement":"SELECT identity_line_item_id, line_item_usage_account_id, line_item_usage_start_date, line_item_resource_id, product_region, line_item_unblended_cost, resource_tags FROM COST_AND_USAGE_REPORT",
      "TableConfigurations":{"COST_AND_USAGE_REPORT":{"TIME_GRANULARITY":"HOURLY","INCLUDE_RESOURCES":"TRUE","INCLUDE_SPLIT_COST_ALLOCATION_DATA":"TRUE"}}
    },
    "DestinationConfigurations":{"S3Destination":{"S3Bucket":"my-cur-bucket","S3Prefix":"cur2","S3Region":"ap-south-1","S3OutputConfigurations":{"OutputType":"CUSTOM","Format":"PARQUET","Compression":"PARQUET","Overwrite":"OVERWRITE_REPORT"}}},
    "RefreshCadence":{"Frequency":"SYNCHRONOUS"}
  }'

aws bcm-data-exports list-exports
aws bcm-data-exports get-export --export-arn <arn>
aws bcm-data-exports list-tables            # see FOCUS_1_0, COST_AND_USAGE_REPORT, etc.
aws bcm-data-exports delete-export --export-arn <arn>
```

---

## 5. Cost Allocation Tags & Cost Categories

```bash
# List cost allocation tag status (Active / Inactive)
aws ce list-cost-allocation-tags

# Activate tag keys for billing
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Environment,Status=Active TagKey=Project,Status=Active

# Backfill tags for past months (quarterly limit applies)
aws ce start-cost-allocation-tag-backfill --backfill-from 2026-04-01T00:00:00Z
aws ce list-cost-allocation-tag-backfill-history

# Cost Categories — create with ordered rules
aws ce create-cost-category-definition \
  --name "BusinessUnit" \
  --rule-version "CostCategoryExpression.v1" \
  --rules '[
    {"Value":"Platform","Rule":{"Dimensions":{"Key":"LINKED_ACCOUNT","Values":["111122223333"]}}},
    {"Value":"Retail","Rule":{"Tags":{"Key":"Team","Values":["retail"]}}}
  ]'

aws ce list-cost-category-definitions
aws ce describe-cost-category-definition --cost-category-arn <arn>
aws ce delete-cost-category-definition --cost-category-arn <arn>
```

---

## 6. Resource Tagging

```bash
# Tag EC2 resources
aws ec2 create-tags \
  --resources i-0abcdef1234567890 vol-0123456789abcdef0 \
  --tags Key=Environment,Value=Production Key=Owner,Value=DevOpsTeam

# Find ALL untagged resources for a key (org hygiene)
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Environment \
  --query 'ResourceTagMappingList[].ResourceARN'

# Everything WITHOUT a given tag — list all, then diff; or use AWS Config rule `required-tags`
aws resourcegroupstaggingapi get-resources --query 'ResourceTagMappingList[?!not_null(Tags[?Key==`Environment`])].ResourceARN'

# Bulk tag via ARNs
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list arn:aws:s3:::my-bucket \
  --tags Environment=Production

# Organizations Tag Policy attach (governance)
aws organizations create-policy --type TAG_POLICY --name StandardTags \
  --description "Enforce Environment tag casing" \
  --content file://tag-policy.json
aws organizations attach-policy --policy-id p-xxxx --target-id ou-xxxx
```

---

## 7. Compute Optimizer

```bash
# Opt in (single account)
aws compute-optimizer update-enrollment-status --status Active

# Opt in for the whole organization (run from management account)
aws compute-optimizer update-enrollment-status --status Active --include-member-accounts

aws compute-optimizer get-enrollment-status

# EC2 recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[].{id:instanceArn,finding:finding,top:recommendationOptions[0].instanceType,savings:recommendationOptions[0].savingsOpportunity.estimatedMonthlySavings.value}'

# Filter to over-provisioned only
aws compute-optimizer get-ec2-instance-recommendations \
  --filters name=Finding,values=Overprovisioned

# Other resource types
aws compute-optimizer get-auto-scaling-group-recommendations
aws compute-optimizer get-ebs-volume-recommendations
aws compute-optimizer get-lambda-function-recommendations
aws compute-optimizer get-ecs-service-recommendations
aws compute-optimizer get-rds-database-recommendations
aws compute-optimizer get-idle-recommendations
aws compute-optimizer get-license-recommendations

# Projected utilization on recommended type
aws compute-optimizer get-ec2-recommendation-projected-metrics \
  --instance-arn arn:aws:ec2:ap-south-1:111122223333:instance/i-0abc... \
  --stat MAXIMUM --period 3600 \
  --start-time 2026-07-14T00:00:00Z --end-time 2026-07-28T00:00:00Z

# Preferences: extend lookback / exclude Graviton
aws compute-optimizer put-recommendation-preferences \
  --resource-type Ec2Instance \
  --scope name=ORGANIZATION,value=ALL_ACCOUNTS \
  --enhanced-infrastructure-metrics Active

aws compute-optimizer put-recommendation-preferences \
  --resource-type Ec2Instance \
  --scope name=ACCOUNT_ID,value=111122223333 \
  --preferred-resources '[{"name":"Ec2InstanceTypes","excludeList":["m6g","c6g","r6g"]}]'

# Bulk export to S3
aws compute-optimizer export-ec2-instance-recommendations \
  --s3-destination-config bucket=my-optimizer-exports,keyPrefix=ec2/ \
  --file-format Csv
```

---

## 8. Cost Optimization Hub

```bash
# Enroll (account or org-wide from management account)
aws cost-optimization-hub update-enrollment-status --status Active --include-member-accounts

aws cost-optimization-hub get-enrollment-statuses

# Summarize savings by type / account / region
aws cost-optimization-hub list-recommendation-summaries --group-by ActionType
aws cost-optimization-hub list-recommendation-summaries --group-by AccountId

# List individual recommendations, highest savings first
aws cost-optimization-hub list-recommendations \
  --order-by dimension=Savings,order=Desc \
  --include-all-recommendations

aws cost-optimization-hub get-recommendation --recommendation-id <id>

# Preferences (e.g., savings mode after discounts)
aws cost-optimization-hub update-preferences \
  --savings-estimation-mode AfterDiscounts \
  --member-account-discount-visibility All
```

---

## 9. Trusted Advisor (`support`)

> Requires **Business/Enterprise Support**; API endpoint lives in `us-east-1`.

```bash
# List all checks
aws support describe-trusted-advisor-checks --language en --region us-east-1 \
  --query 'checks[].{id:id,name:name,category:category}'

# Get results of a specific check (e.g., Low Utilization EC2 = Qch7DwouX1)
aws support describe-trusted-advisor-check-result \
  --check-id Qch7DwouX1 --language en --region us-east-1

# Summaries for several checks
aws support describe-trusted-advisor-check-summaries \
  --check-ids Qch7DwouX1 DAvU99Dc4C --region us-east-1

# Refresh a check manually
aws support refresh-trusted-advisor-check --check-id Qch7DwouX1 --region us-east-1
aws support describe-trusted-advisor-check-refresh-statuses --check-ids Qch7DwouX1 --region us-east-1
```

Newer TrustedAdvisor API (Business+):
```bash
aws trustedadvisor list-checks --language en
aws trustedadvisor list-recommendations
aws trustedadvisor get-recommendation --recommendation-identifier <arn>
aws trustedadvisor list-recommendation-resources --recommendation-identifier <arn>
```

---

## 10. Service Quotas

```bash
# List services / quotas
aws service-quotas list-services
aws service-quotas list-service-quotas --service-code ec2
aws service-quotas list-aws-default-service-quotas --service-code ec2

# Get one quota (e.g., Running On-Demand Standard vCPUs = L-1216C47A)
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A

# Request an increase
aws service-quotas request-service-quota-increase \
  --service-code ec2 --quota-code L-1216C47A --desired-value 256

# Track requests
aws service-quotas list-requested-service-quota-change-history
aws service-quotas list-requested-service-quota-change-history-by-quota \
  --service-code ec2 --quota-code L-1216C47A

# Automatic Management (NotifyOnly / NotifyAndAdjust)
aws service-quotas start-quota-utilization-notifications --opt-in-type NotifyOnly
aws service-quotas start-quota-utilization-notifications --opt-in-type NotifyAndAdjust
aws service-quotas stop-quota-utilization-notifications

# Org-wide quota request template (management account)
aws service-quotas associate-service-quota-template
aws service-quotas put-service-quota-increase-request-into-template \
  --service-code ec2 --quota-code L-1216C47A --aws-region ap-south-1 --desired-value 256
aws service-quotas list-service-quota-increase-requests-in-template
```

---

## 11. Savings Plans & Reserved Instances

```bash
# Savings Plans
aws savingsplans describe-savings-plans
aws savingsplans describe-savings-plans-offerings \
  --plan-types Compute --payment-options "No Upfront" --durations 31536000
aws savingsplans create-savings-plan \
  --savings-plan-offering-id <offering-id> --commitment 1.50   # $/hour

# Reserved Instances (EC2)
aws ec2 describe-reserved-instances
aws ec2 describe-reserved-instances-offerings \
  --instance-type m6g.xlarge --offering-class standard --product-description "Linux/UNIX"
aws ec2 purchase-reserved-instances-offering \
  --reserved-instances-offering-id <id> --instance-count 2
aws ec2 describe-reserved-instances-modifications
```

---

## 12. Athena Queries for CUR

```sql
-- Top 10 most expensive resources this month
SELECT line_item_resource_id,
       SUM(line_item_unblended_cost) AS cost
FROM   cur_db.cur_table
WHERE  month = '7' AND year = '2026'
  AND  line_item_line_item_type = 'Usage'
GROUP  BY line_item_resource_id
ORDER  BY cost DESC
LIMIT  10;

-- Cost by account and service
SELECT line_item_usage_account_id, product_servicecode,
       SUM(line_item_unblended_cost) AS cost
FROM   cur_db.cur_table
WHERE  month = '7' AND year = '2026'
GROUP  BY 1, 2 ORDER BY cost DESC;

-- Spend per Environment tag
SELECT resource_tags_user_environment,
       SUM(line_item_unblended_cost) AS cost
FROM   cur_db.cur_table
WHERE  month = '7' AND year = '2026'
GROUP  BY 1 ORDER BY cost DESC;

-- Data-transfer cost breakdown
SELECT line_item_usage_type, SUM(line_item_unblended_cost) AS cost
FROM   cur_db.cur_table
WHERE  line_item_usage_type LIKE '%DataTransfer%'
  AND  month = '7' AND year = '2026'
GROUP  BY 1 ORDER BY cost DESC;

-- Amortized cost (true daily ops view incl. commitments)
SELECT line_item_usage_start_date,
       SUM(CASE
             WHEN line_item_line_item_type = 'SavingsPlanCoveredUsage' THEN savings_plan_savings_plan_effective_cost
             WHEN line_item_line_item_type = 'DiscountedUsage'         THEN reservation_effective_cost
             WHEN line_item_line_item_type = 'Usage'                   THEN line_item_unblended_cost
             ELSE 0 END) AS amortized_cost
FROM   cur_db.cur_table
WHERE  month = '7' AND year = '2026'
GROUP  BY 1 ORDER BY 1;
```

---

## 13. Supporting Services

```bash
# SNS topic for alerts + email subscription
aws sns create-topic --name budget-alerts
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:111122223333:budget-alerts \
  --protocol email --notification-endpoint finops@example.com

# SNS topic policy must allow budgets.amazonaws.com / costalerts.amazonaws.com to publish
aws sns set-topic-attributes --topic-arn <arn> --attribute-name Policy --attribute-value file://sns-policy.json

# Install CloudWatch Agent (memory metrics for rightsizing) via SSM
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{"action":["Install"],"name":["AmazonCloudWatchAgent"]}'

aws ssm send-command \
  --document-name "AmazonCloudWatch-ManageAgent" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{"action":["configure"],"mode":["ec2"],"optionalConfigurationSource":["ssm"],"optionalConfigurationLocation":["AmazonCloudWatch-Config"],"optionalRestart":["yes"]}'

# EventBridge rule for Trusted Advisor status changes (must be us-east-1)
aws events put-rule --region us-east-1 \
  --name ta-cost-alerts \
  --event-pattern '{
    "source":["aws.trustedadvisor"],
    "detail-type":["Trusted Advisor Check Item Refresh Notification"],
    "detail":{"status":["WARN","ERROR"],"check-name":["Low Utilization Amazon EC2 Instances","Unassociated Elastic IP Addresses"]}
  }'
aws events put-targets --region us-east-1 --rule ta-cost-alerts \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:111122223333:function:ta-remediate"

# CloudWatch alarm on a Service Quota (80% of vCPU limit)
aws cloudwatch put-metric-alarm \
  --alarm-name vcpu-quota-80 \
  --namespace AWS/Usage \
  --metric-name ResourceCount \
  --dimensions Name=Service,Value=EC2 Name=Type,Value=Resource Name=Resource,Value=vCPU Name=Class,Value=Standard/OnDemand \
  --statistic Maximum --period 300 --threshold 80 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:ap-south-1:111122223333:budget-alerts
```

---

*Tip: wrap frequently used `ce` queries in shell functions — every paginated call costs $0.01, so cache results where you can.*
