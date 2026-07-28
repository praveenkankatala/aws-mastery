# AWS Billing & Cost Management — CLI Commands Cheatsheet

> All commands assume AWS CLI v2 with a configured profile. Most billing APIs are **global** and must be called against **`us-east-1`** — add `--region us-east-1` if your default region differs.
>
> 💡 Cost Explorer API calls (`aws ce ...`) cost **$0.01 per paginated request**. Everything else here is free to call.

---

## Table of Contents

1. [Setup & Identity](#1-setup--identity)
2. [Cost Explorer (`ce`)](#2-cost-explorer-ce)
3. [Budgets (`budgets`)](#3-budgets-budgets)
4. [Cost Anomaly Detection (`ce` anomaly APIs)](#4-cost-anomaly-detection-ce-anomaly-apis)
5. [Cost & Usage Reports / Data Exports](#5-cost--usage-reports--data-exports)
6. [Cost Allocation Tags & Cost Categories](#6-cost-allocation-tags--cost-categories)
7. [Savings Plans (`savingsplans`)](#7-savings-plans-savingsplans)
8. [Reserved Instances](#8-reserved-instances)
9. [AWS Organizations (consolidated billing)](#9-aws-organizations-consolidated-billing)
10. [Pricing APIs (`pricing`)](#10-pricing-apis-pricing)
11. [Compute Optimizer & Trusted Advisor](#11-compute-optimizer--trusted-advisor)
12. [Resource Tagging at Scale](#12-resource-tagging-at-scale)
13. [Legacy CloudWatch Billing Alarm](#13-legacy-cloudwatch-billing-alarm)
14. [Athena Queries for CUR 2.0](#14-athena-queries-for-cur-20)
15. [Waste-Hunting One-Liners](#15-waste-hunting-one-liners)

---

## 1. Setup & Identity

```bash
# Verify CLI and identity
aws --version
aws sts get-caller-identity

# Set variables used throughout this sheet
export AWS_REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# List AWS-managed billing policies you may attach to a FinOps role
aws iam list-policies --scope AWS --query \
  "Policies[?contains(PolicyName,'Billing') || contains(PolicyName,'Budgets') || contains(PolicyName,'Cost')].PolicyName"
```

---

## 2. Cost Explorer (`ce`)

### 2.1 Cost and usage

```bash
# Monthly unblended cost for the current month, grouped by service
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE

# Daily cost for the last 7 days (spike hunting)
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '7 days ago' +%F),End=$(date +%F) \
  --granularity DAILY \
  --metrics "UnblendedCost"

# Amortized cost grouped by linked account (org view)
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY \
  --metrics "AmortizedCost" \
  --group-by Type=DIMENSION,Key=LINKED_ACCOUNT

# Cost filtered to one service (EC2) and grouped by usage type
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE

# Cost grouped by a cost allocation tag
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=TAG,Key=Environment

# Resource-level detail (requires hourly/resource granularity opt-in; last 14 days)
aws ce get-cost-and-usage-with-resources \
  --time-period Start=$(date -d '3 days ago' +%F),End=$(date +%F) \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute"]}}' \
  --group-by Type=DIMENSION,Key=RESOURCE_ID
```

### 2.2 Forecasting

```bash
# Forecast total cost to end of month
aws ce get-cost-forecast \
  --time-period Start=$(date +%F),End=2026-08-01 \
  --granularity MONTHLY \
  --metric UNBLENDED_COST

# Usage forecast (e.g., for a usage budget)
aws ce get-usage-forecast \
  --time-period Start=$(date +%F),End=2026-08-01 \
  --granularity MONTHLY \
  --metric USAGE_QUANTITY \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute"]}}'
```

### 2.3 Dimensions & discovery

```bash
# What values exist for a dimension? (find exact SERVICE names for filters)
aws ce get-dimension-values \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --dimension SERVICE

aws ce get-dimension-values \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --dimension LINKED_ACCOUNT

# Tag keys/values seen in cost data
aws ce get-tags --time-period Start=2026-07-01,End=2026-08-01
aws ce get-tags --time-period Start=2026-07-01,End=2026-08-01 --tag-key Environment
```

### 2.4 RI / Savings Plans analytics

```bash
# Savings Plans purchase recommendations
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days THIRTY_DAYS

# Savings Plans utilization & coverage
aws ce get-savings-plans-utilization \
  --time-period Start=2026-07-01,End=2026-08-01
aws ce get-savings-plans-coverage \
  --time-period Start=2026-07-01,End=2026-08-01

# RI recommendations / utilization / coverage
aws ce get-reservation-purchase-recommendation \
  --service "Amazon Relational Database Service"
aws ce get-reservation-utilization \
  --time-period Start=2026-07-01,End=2026-08-01
aws ce get-reservation-coverage \
  --time-period Start=2026-07-01,End=2026-08-01

# Rightsizing recommendations (Cost Explorer flavor)
aws ce get-rightsizing-recommendation \
  --service "AmazonEC2" \
  --configuration '{"RecommendationTarget":"SAME_INSTANCE_FAMILY","BenefitsConsidered":true}'
```

---

## 3. Budgets (`budgets`)

```bash
# List budgets
aws budgets describe-budgets --account-id $ACCOUNT_ID

# Create a monthly cost budget of $100 with 80% actual + 100% forecast email alerts
aws budgets create-budget \
  --account-id $ACCOUNT_ID \
  --budget '{
    "BudgetName": "monthly-total-100",
    "BudgetLimit": {"Amount": "100", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"},
      "Subscribers": [{"SubscriptionType":"EMAIL","Address":"you@example.com"}]
    },
    {
      "Notification": {"NotificationType":"FORECASTED","ComparisonOperator":"GREATER_THAN","Threshold":100,"ThresholdType":"PERCENTAGE"},
      "Subscribers": [{"SubscriptionType":"EMAIL","Address":"you@example.com"}]
    }
  ]'

# Budget scoped by tag filter (CostFilters uses "TagKeyValue" format user:Key$Value)
aws budgets create-budget \
  --account-id $ACCOUNT_ID \
  --budget '{
    "BudgetName": "prod-env-budget",
    "BudgetLimit": {"Amount":"500","Unit":"USD"},
    "TimeUnit":"MONTHLY","BudgetType":"COST",
    "CostFilters": {"TagKeyValue":["user:Environment$prod"]}
  }'

# Add an SNS subscriber to an existing budget notification
aws budgets create-subscriber \
  --account-id $ACCOUNT_ID \
  --budget-name monthly-total-100 \
  --notification '{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"}' \
  --subscriber '{"SubscriptionType":"SNS","Address":"arn:aws:sns:us-east-1:'"$ACCOUNT_ID"':budget-alerts"}'

# Update / delete
aws budgets update-budget --account-id $ACCOUNT_ID --new-budget file://budget.json
aws budgets delete-budget --account-id $ACCOUNT_ID --budget-name monthly-total-100

# Budget actions
aws budgets describe-budget-actions-for-account --account-id $ACCOUNT_ID
aws budgets describe-budget-action-histories \
  --account-id $ACCOUNT_ID --budget-name sandbox-budget --action-id <action-id>
aws budgets execute-budget-action \
  --account-id $ACCOUNT_ID --budget-name sandbox-budget \
  --action-id <action-id> --execution-type APPROVE_BUDGET_ACTION
```

> 🔎 **SNS gotcha:** the SNS topic's **access policy must allow `budgets.amazonaws.com` to `SNS:Publish`** — see troubleshooting.md §4.

---

## 4. Cost Anomaly Detection (`ce` anomaly APIs)

```bash
# Create an ML monitor across all AWS services
aws ce create-anomaly-monitor \
  --anomaly-monitor '{"MonitorName":"all-services","MonitorType":"DIMENSIONAL","MonitorDimension":"SERVICE"}'

# Create an alert subscription: immediate SNS alert for anomalies with impact > $50
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName":"finops-alerts",
    "MonitorArnList":["<monitor-arn>"],
    "Subscribers":[{"Type":"SNS","Address":"arn:aws:sns:us-east-1:'"$ACCOUNT_ID"':anomaly-alerts"}],
    "Frequency":"IMMEDIATE",
    "ThresholdExpression":{"Dimensions":{"Key":"ANOMALY_TOTAL_IMPACT_ABSOLUTE","MatchOptions":["GREATER_THAN_OR_EQUAL"],"Values":["50"]}}
  }'

# List monitors / subscriptions / detected anomalies
aws ce get-anomaly-monitors
aws ce get-anomaly-subscriptions
aws ce get-anomalies \
  --date-interval StartDate=2026-07-01,EndDate=2026-07-28

# Give feedback on an anomaly (improves the model)
aws ce provide-anomaly-feedback --anomaly-id <id> --feedback YES
```

---

## 5. Cost & Usage Reports / Data Exports

### 5.1 Legacy CUR (`cur`)

```bash
aws cur describe-report-definitions --region us-east-1

aws cur put-report-definition --region us-east-1 --report-definition '{
  "ReportName": "my-cur",
  "TimeUnit": "HOURLY",
  "Format": "Parquet",
  "Compression": "Parquet",
  "AdditionalSchemaElements": ["RESOURCES"],
  "S3Bucket": "my-cur-bucket",
  "S3Prefix": "cur",
  "S3Region": "us-east-1",
  "AdditionalArtifacts": ["ATHENA"],
  "RefreshClosedReports": true,
  "ReportVersioning": "OVERWRITE_REPORT"
}'

aws cur delete-report-definition --report-name my-cur --region us-east-1
```

### 5.2 Data Exports / CUR 2.0 (`bcm-data-exports`)

```bash
# List exports and tables
aws bcm-data-exports list-exports
aws bcm-data-exports list-tables

# Create a CUR 2.0 export (SQL-selected columns) to S3 as Parquet
aws bcm-data-exports create-export --export '{
  "Name": "cur2-daily",
  "DataQuery": {
    "QueryStatement": "SELECT bill_billing_period_start_date, line_item_usage_account_id, line_item_product_code, line_item_usage_type, line_item_unblended_cost, line_item_usage_start_date, line_item_resource_id, product_region_code FROM COST_AND_USAGE_REPORT",
    "TableConfigurations": {"COST_AND_USAGE_REPORT": {"TIME_GRANULARITY":"DAILY","INCLUDE_RESOURCES":"TRUE","INCLUDE_SPLIT_COST_ALLOCATION_DATA":"FALSE","INCLUDE_MANUAL_DISCOUNT_COMPATIBILITY":"FALSE"}}
  },
  "DestinationConfigurations": {
    "S3Destination": {
      "S3Bucket": "my-cur-bucket",
      "S3Prefix": "cur2",
      "S3Region": "us-east-1",
      "S3OutputConfigurations": {"OutputType":"CUSTOM","Format":"PARQUET","Compression":"PARQUET","Overwrite":"OVERWRITE_REPORT"}
    }
  },
  "RefreshCadence": {"Frequency": "SYNCHRONOUS"}
}'

aws bcm-data-exports get-execution --export-arn <arn> --execution-id <id>
aws bcm-data-exports delete-export --export-arn <arn>
```

---

## 6. Cost Allocation Tags & Cost Categories

```bash
# List cost allocation tag keys and their activation status
aws ce list-cost-allocation-tags

# Activate / deactivate tag keys for cost allocation
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Environment,Status=Active TagKey=Project,Status=Active

# Request historical backfill of newly-activated tags (up to 12 months)
aws ce start-cost-allocation-tag-backfill --backfill-from 2026-01-01T00:00:00Z
aws ce list-cost-allocation-tag-backfill-history

# Cost categories
aws ce list-cost-category-definitions
aws ce describe-cost-category-definition --cost-category-arn <arn>

aws ce create-cost-category-definition \
  --name "Teams" \
  --rule-version "CostCategoryExpression.v1" \
  --rules '[
    {"Value":"Platform","Rule":{"Tags":{"Key":"Team","Values":["platform"],"MatchOptions":["EQUALS"]}}},
    {"Value":"Payments","Rule":{"Tags":{"Key":"Team","Values":["payments"],"MatchOptions":["EQUALS"]}}},
    {"Value":"Unallocated","Rule":{"Not":{"Tags":{"Key":"Team","MatchOptions":["ABSENT"]}}}}
  ]'

aws ce delete-cost-category-definition --cost-category-arn <arn>
```

---

## 7. Savings Plans (`savingsplans`)

```bash
# What plans do I own?
aws savingsplans describe-savings-plans

# Available offerings (rates/terms)
aws savingsplans describe-savings-plans-offerings \
  --plan-types Compute --durations 31536000 --payment-options "No Upfront"

aws savingsplans describe-savings-plans-offering-rates \
  --savings-plan-offering-ids <offering-id>

# Purchase (⚠️ real financial commitment — triple check!)
aws savingsplans create-savings-plan \
  --savings-plan-offering-id <offering-id> \
  --commitment 1.50 \
  --client-token $(uuidgen)

# 7-day return window for eligible plans
aws savingsplans return-savings-plan --savings-plan-id <id>

# Utilization & coverage → see §2.4 (aws ce get-savings-plans-*)
```

---

## 8. Reserved Instances

```bash
# EC2 RIs
aws ec2 describe-reserved-instances
aws ec2 describe-reserved-instances-offerings \
  --instance-type t3.medium --product-description "Linux/UNIX" \
  --offering-class standard --max-results 5
aws ec2 purchase-reserved-instances-offering \
  --reserved-instances-offering-id <id> --instance-count 1
aws ec2 modify-reserved-instances \
  --reserved-instances-ids <ri-id> \
  --target-configurations AvailabilityZone=us-east-1a,InstanceCount=1
aws ec2 get-reserved-instances-exchange-quote \
  --reserved-instance-ids <convertible-ri-id> \
  --target-configurations OfferingId=<offering-id>

# Sell a Standard RI on the marketplace
aws ec2 create-reserved-instances-listing \
  --reserved-instances-id <ri-id> --instance-count 1 \
  --price-schedules CurrencyCode=USD,Price=200,Term=6 \
  --client-token $(uuidgen)

# RDS RIs
aws rds describe-reserved-db-instances
aws rds describe-reserved-db-instances-offerings \
  --db-instance-class db.t3.medium --product-description mysql
aws rds purchase-reserved-db-instances-offering \
  --reserved-db-instances-offering-id <id>
```

---

## 9. AWS Organizations (consolidated billing)

```bash
# Create an org (run from the future management account)
aws organizations create-organization --feature-set ALL

aws organizations describe-organization
aws organizations list-accounts
aws organizations list-roots

# Create / invite member accounts
aws organizations create-account \
  --email dev-team@example.com --account-name "dev-account"
aws organizations describe-create-account-status \
  --create-account-request-id <id>
aws organizations invite-account-to-organization \
  --target Id=222233334444,Type=ACCOUNT

# OUs and SCPs (cost guardrails)
aws organizations create-organizational-unit --parent-id <root-id> --name Sandbox
aws organizations create-policy \
  --name deny-big-instances --type SERVICE_CONTROL_POLICY \
  --description "Block expensive instance types" \
  --content '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Deny","Action":"ec2:RunInstances",
      "Resource":"arn:aws:ec2:*:*:instance/*",
      "Condition":{"StringLike":{"ec2:InstanceType":["*.8xlarge","*.12xlarge","*.16xlarge","*.24xlarge","p4*","p5*"]}}}]
  }'
aws organizations attach-policy --policy-id <p-id> --target-id <ou-id>

# Tag policies (enforce tagging standards)
aws organizations enable-policy-type --root-id <root-id> --policy-type TAG_POLICY
```

---

## 10. Pricing APIs (`pricing`)

> Endpoints only in `us-east-1` and `ap-south-1`.

```bash
aws pricing describe-services --region us-east-1 --max-results 20
aws pricing describe-services --service-code AmazonEC2 --region us-east-1

# Attribute values (e.g., valid instanceType values)
aws pricing get-attribute-values \
  --service-code AmazonEC2 --attribute-name instanceType --region us-east-1

# On-demand price of t3.medium Linux in us-east-1
aws pricing get-products --region us-east-1 \
  --service-code AmazonEC2 \
  --filters \
    Type=TERM_MATCH,Field=instanceType,Value=t3.medium \
    Type=TERM_MATCH,Field=location,Value="US East (N. Virginia)" \
    Type=TERM_MATCH,Field=operatingSystem,Value=Linux \
    Type=TERM_MATCH,Field=tenancy,Value=Shared \
    Type=TERM_MATCH,Field=preInstalledSw,Value=NA \
    Type=TERM_MATCH,Field=capacitystatus,Value=Used \
  --max-results 1
```

---

## 11. Compute Optimizer & Trusted Advisor

```bash
# Compute Optimizer
aws compute-optimizer get-enrollment-status
aws compute-optimizer update-enrollment-status --status Active   # opt in
aws compute-optimizer get-ec2-instance-recommendations
aws compute-optimizer get-ebs-volume-recommendations
aws compute-optimizer get-lambda-function-recommendations
aws compute-optimizer get-auto-scaling-group-recommendations
aws compute-optimizer get-recommendation-summaries

# Cost Optimization Hub
aws cost-optimization-hub update-enrollment-status --status Active
aws cost-optimization-hub list-recommendations \
  --query "items[].{res:resourceId,action:actionType,save:estimatedMonthlySavings}"
aws cost-optimization-hub list-recommendation-summaries --group-by ActionType

# Trusted Advisor (requires Business/Enterprise support; support API in us-east-1)
aws support describe-trusted-advisor-checks --language en --region us-east-1 \
  --query "checks[?category=='cost_optimizing'].{id:id,name:name}"
aws support describe-trusted-advisor-check-result \
  --check-id <check-id> --region us-east-1
aws support refresh-trusted-advisor-check --check-id <check-id> --region us-east-1
```

---

## 12. Resource Tagging at Scale

```bash
# Find resources by tag (or missing tags) across services
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Environment,Values=prod

# All tag keys / values in the region
aws resourcegroupstaggingapi get-tag-keys
aws resourcegroupstaggingapi get-tag-values --key Environment

# Bulk-tag resources by ARN
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list arn:aws:ec2:us-east-1:$ACCOUNT_ID:instance/i-0123456789abcdef0 \
  --tags Environment=prod,Team=platform

# Tag on create (EC2 example)
aws ec2 run-instances --image-id ami-xxxx --instance-type t3.micro \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Environment,Value=dev},{Key=Team,Value=platform}]'

# Find EC2 instances MISSING a required tag
aws ec2 describe-instances \
  --query "Reservations[].Instances[?!not_null(Tags[?Key=='Team'])].InstanceId" \
  --output text
```

---

## 13. Legacy CloudWatch Billing Alarm

```bash
# Prereq: enable "Receive Billing Alerts" in Billing preferences (console, once)
# The metric only exists in us-east-1

aws sns create-topic --name billing-alarm --region us-east-1
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:$ACCOUNT_ID:billing-alarm \
  --protocol email --notification-endpoint you@example.com --region us-east-1

aws cloudwatch put-metric-alarm \
  --alarm-name total-bill-over-50 \
  --namespace AWS/Billing \
  --metric-name EstimatedCharges \
  --dimensions Name=Currency,Value=USD \
  --statistic Maximum \
  --period 21600 \
  --evaluation-periods 1 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:$ACCOUNT_ID:billing-alarm \
  --region us-east-1
```

---

## 14. Athena Queries for CUR 2.0

```sql
-- Monthly cost by service
SELECT line_item_product_code,
       ROUND(SUM(line_item_unblended_cost), 2) AS cost
FROM   cur2_daily
WHERE  bill_billing_period_start_date = DATE '2026-07-01'
GROUP  BY 1
ORDER  BY cost DESC
LIMIT  20;

-- Daily trend for one service
SELECT DATE(line_item_usage_start_date) AS day,
       ROUND(SUM(line_item_unblended_cost), 2) AS cost
FROM   cur2_daily
WHERE  line_item_product_code = 'AmazonEC2'
GROUP  BY 1 ORDER BY 1;

-- Top 20 most expensive resources this month
SELECT line_item_resource_id,
       line_item_product_code,
       ROUND(SUM(line_item_unblended_cost), 2) AS cost
FROM   cur2_daily
WHERE  bill_billing_period_start_date = DATE '2026-07-01'
  AND  line_item_resource_id <> ''
GROUP  BY 1, 2 ORDER BY cost DESC LIMIT 20;

-- Untagged spend (allocation gap KPI) — tag columns appear after activation
SELECT ROUND(SUM(line_item_unblended_cost), 2) AS untagged_cost
FROM   cur2_daily
WHERE  bill_billing_period_start_date = DATE '2026-07-01'
  AND  (resource_tags['user_team'] IS NULL OR resource_tags['user_team'] = '');

-- Data transfer costs (a classic hidden cost)
SELECT line_item_usage_type,
       ROUND(SUM(line_item_unblended_cost), 2) AS cost
FROM   cur2_daily
WHERE  line_item_usage_type LIKE '%DataTransfer%'
   OR  line_item_usage_type LIKE '%NatGateway%'
GROUP  BY 1 ORDER BY cost DESC;
```

---

## 15. Waste-Hunting One-Liners

```bash
# Unattached EBS volumes
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query "Volumes[].{ID:VolumeId,Size:Size,Type:VolumeType,Created:CreateTime}" --output table

# Unassociated Elastic IPs (billed when idle)
aws ec2 describe-addresses \
  --query "Addresses[?AssociationId==null].{IP:PublicIp,AllocId:AllocationId}" --output table

# gp2 volumes that should be gp3
aws ec2 describe-volumes --filters Name=volume-type,Values=gp2 \
  --query "Volumes[].{ID:VolumeId,Size:Size}" --output table
aws ec2 modify-volume --volume-id vol-xxxx --volume-type gp3

# Stopped instances still paying for EBS
aws ec2 describe-instances --filters Name=instance-state-name,Values=stopped \
  --query "Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,Stopped:StateTransitionReason}" --output table

# Old snapshots (> 90 days)
aws ec2 describe-snapshots --owner-ids self \
  --query "Snapshots[?StartTime<='$(date -d '90 days ago' +%F)'].{ID:SnapshotId,Vol:VolumeId,Date:StartTime}" --output table

# Idle NAT gateways / unused load balancers → correlate with CloudWatch metrics
aws ec2 describe-nat-gateways --query "NatGateways[].{ID:NatGatewayId,VPC:VpcId,State:State}" --output table
aws elbv2 describe-load-balancers --query "LoadBalancers[].{Name:LoadBalancerName,Created:CreatedTime}" --output table

# Incomplete S3 multipart uploads (invisible storage cost)
aws s3api list-multipart-uploads --bucket my-bucket

# Stop all instances with a tag (night-time saver)
aws ec2 describe-instances \
  --filters Name=tag:AutoStop,Values=true Name=instance-state-name,Values=running \
  --query "Reservations[].Instances[].InstanceId" --output text | \
  xargs -r aws ec2 stop-instances --instance-ids
```
