# CloudTrail — CLI Commands Cheatsheet

All commands use AWS CLI v2. Replace placeholders (`<...>`) with your values.

---

## Table of Contents

1. [Trail Lifecycle](#1-trail-lifecycle)
2. [Event Selectors (Data Events)](#2-event-selectors-data-events)
3. [Insights Events](#3-insights-events)
4. [Querying Event History (lookup-events)](#4-querying-event-history)
5. [Log File Integrity Validation](#5-log-file-integrity-validation)
6. [CloudWatch Logs Integration](#6-cloudwatch-logs-integration)
7. [KMS Encryption](#7-kms-encryption)
8. [S3 Bucket Policy for CloudTrail](#8-s3-bucket-policy-for-cloudtrail)
9. [Athena Setup & Forensic Queries](#9-athena-setup--forensic-queries)
10. [CloudTrail Lake](#10-cloudtrail-lake)
11. [Organization Trails](#11-organization-trails)

---

## 1. Trail Lifecycle

```bash
# Create a multi-region trail with integrity validation + global service events
aws cloudtrail create-trail \
  --name org-security-trail \
  --s3-bucket-name my-cloudtrail-audit-bucket \
  --is-multi-region-trail \
  --include-global-service-events \
  --enable-log-file-validation

# START logging (create-trail does NOT start it — a classic gotcha)
aws cloudtrail start-logging --name org-security-trail

# Status: is it actually logging? last delivery? last error?
aws cloudtrail get-trail-status --name org-security-trail

# List / describe trails
aws cloudtrail list-trails
aws cloudtrail describe-trails
aws cloudtrail describe-trails --trail-name-list org-security-trail

# Update a trail (e.g., add an S3 key prefix)
aws cloudtrail update-trail --name org-security-trail --s3-key-prefix prod-audit

# Stop / delete (alert on these API calls in production! see README §9)
aws cloudtrail stop-logging --name org-security-trail
aws cloudtrail delete-trail --name org-security-trail

# Tag a trail
aws cloudtrail add-tags --resource-id <trail-arn> \
  --tags-list Key=Owner,Value=SecurityTeam
```

---

## 2. Event Selectors (Data Events)

```bash
# View current selectors
aws cloudtrail get-event-selectors --trail-name org-security-trail

# Classic selectors: management events + S3 data events for ONE bucket
aws cloudtrail put-event-selectors \
  --trail-name org-security-trail \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [
      { "Type": "AWS::S3::Object",
        "Values": ["arn:aws:s3:::my-sensitive-bucket/"] },
      { "Type": "AWS::Lambda::Function",
        "Values": ["arn:aws:lambda:ap-south-1:111122223333:function:payment-fn"] }
    ]
  }]'

# Advanced event selectors: finer filtering (e.g., ONLY DeleteObject on one prefix)
aws cloudtrail put-event-selectors \
  --trail-name org-security-trail \
  --advanced-event-selectors '[
    { "Name": "Mgmt events",
      "FieldSelectors": [{ "Field": "eventCategory", "Equals": ["Management"] }] },
    { "Name": "S3 deletes on PII prefix",
      "FieldSelectors": [
        { "Field": "eventCategory", "Equals": ["Data"] },
        { "Field": "resources.type", "Equals": ["AWS::S3::Object"] },
        { "Field": "eventName", "Equals": ["DeleteObject"] },
        { "Field": "resources.ARN", "StartsWith": ["arn:aws:s3:::my-sensitive-bucket/pii/"] }
      ] }
  ]'
```

---

## 3. Insights Events

```bash
# Enable both anomaly types on a trail
aws cloudtrail put-insight-selectors \
  --trail-name org-security-trail \
  --insight-selectors '[
    {"InsightType": "ApiCallRateInsight"},
    {"InsightType": "ApiErrorRateInsight"}
  ]'

# Check what's enabled
aws cloudtrail get-insight-selectors --trail-name org-security-trail
```

---

## 4. Querying Event History

`lookup-events` searches the free 90-day management-event history. One attribute filter per call.

```bash
# Last 10 events
aws cloudtrail lookup-events --max-results 10

# Everything a specific user did
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=admin-praveen

# All occurrences of a specific API call
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket

# All console logins
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ConsoleLogin

# Events touching a specific resource
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=my-sensitive-bucket

# Time-bounded search
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --start-time 2026-07-01T00:00:00Z --end-time 2026-07-28T23:59:59Z

# Search *Insights* events instead of management events
aws cloudtrail lookup-events --event-category insight

# Pretty extraction with --query (who + when + from where)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateUser \
  --query 'Events[].{time:EventTime,user:Username,raw:CloudTrailEvent}' \
  --output table
```

---

## 5. Log File Integrity Validation

```bash
# Validate all digests + log files in a window (tamper detection)
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:ap-south-1:111122223333:trail/org-security-trail \
  --start-time 2026-07-01T00:00:00Z \
  --end-time 2026-07-28T00:00:00Z

# Verbose mode: show every file checked
aws cloudtrail validate-logs \
  --trail-arn <trail-arn> \
  --start-time 2026-07-01T00:00:00Z \
  --verbose
```

Healthy output ends with: `Results requested for ... / No log files were modified, deleted, or skipped.` Any `INVALID` line = investigate immediately.

---

## 6. CloudWatch Logs Integration

```bash
# 1. Log group for the real-time stream
aws logs create-log-group --log-group-name CloudTrail/SecurityEvents
aws logs put-retention-policy --log-group-name CloudTrail/SecurityEvents --retention-in-days 90

# 2. Role CloudTrail assumes to write to CW Logs
aws iam create-role --role-name CloudTrail_CWLogs_Role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow",
      "Principal":{"Service":"cloudtrail.amazonaws.com"},
      "Action":"sts:AssumeRole"}]}'

aws iam put-role-policy --role-name CloudTrail_CWLogs_Role \
  --policy-name write-cwlogs \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow",
      "Action":["logs:CreateLogStream","logs:PutLogEvents"],
      "Resource":"arn:aws:logs:ap-south-1:111122223333:log-group:CloudTrail/SecurityEvents:*"}]}'

# 3. Wire the trail to the log group
aws cloudtrail update-trail \
  --name org-security-trail \
  --cloud-watch-logs-log-group-arn arn:aws:logs:ap-south-1:111122223333:log-group:CloudTrail/SecurityEvents:* \
  --cloud-watch-logs-role-arn arn:aws:iam::111122223333:role/CloudTrail_CWLogs_Role

# 4. Example CIS metric filter: root account usage
aws logs put-metric-filter \
  --log-group-name CloudTrail/SecurityEvents \
  --filter-name RootAccountUsage \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations \
    metricName=RootUsageCount,metricNamespace=CloudTrail/Security,metricValue=1

# 5. Alarm it
aws cloudwatch put-metric-alarm \
  --alarm-name "SEC-RootAccountUsed" \
  --namespace CloudTrail/Security --metric-name RootUsageCount \
  --statistic Sum --period 300 --evaluation-periods 1 \
  --threshold 0 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-south-1:111122223333:security-alerts
```

**More CIS-style filter patterns (swap into step 4):**
```bash
# Console login WITHOUT MFA
'{ ($.eventName = "ConsoleLogin") && ($.additionalEventData.MFAUsed != "Yes") }'

# CloudTrail tampering
'{ ($.eventName = StopLogging) || ($.eventName = DeleteTrail) || ($.eventName = UpdateTrail) }'

# IAM policy changes
'{ ($.eventName = PutUserPolicy) || ($.eventName = PutRolePolicy) || ($.eventName = AttachUserPolicy) || ($.eventName = AttachRolePolicy) || ($.eventName = CreatePolicy) }'

# Security group changes
'{ ($.eventName = AuthorizeSecurityGroupIngress) || ($.eventName = AuthorizeSecurityGroupEgress) || ($.eventName = RevokeSecurityGroupIngress) }'

# Unauthorized API calls (permission probing)
'{ ($.errorCode = "*UnauthorizedOperation") || ($.errorCode = "AccessDenied*") }'

# KMS key disable / scheduled deletion
'{ ($.eventSource = kms.amazonaws.com) && (($.eventName = DisableKey) || ($.eventName = ScheduleKeyDeletion)) }'

# S3 bucket policy changes
'{ ($.eventSource = s3.amazonaws.com) && (($.eventName = PutBucketPolicy) || ($.eventName = PutBucketAcl) || ($.eventName = DeleteBucketPolicy)) }'
```

---

## 7. KMS Encryption

```bash
# Encrypt trail logs with a customer-managed key
aws cloudtrail update-trail \
  --name org-security-trail \
  --kms-key-id arn:aws:kms:ap-south-1:111122223333:key/<key-id>
```

The key policy must allow CloudTrail to use it:
```json
{
  "Sid": "AllowCloudTrailEncrypt",
  "Effect": "Allow",
  "Principal": { "Service": "cloudtrail.amazonaws.com" },
  "Action": "kms:GenerateDataKey*",
  "Resource": "*",
  "Condition": { "StringLike": {
    "kms:EncryptionContext:aws:cloudtrail:arn":
      "arn:aws:cloudtrail:*:111122223333:trail/*" } }
}
```

---

## 8. S3 Bucket Policy for CloudTrail

The audit bucket must grant the CloudTrail service principal write access (with the account condition to prevent the *confused deputy* problem):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::my-cloudtrail-audit-bucket"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-cloudtrail-audit-bucket/AWSLogs/111122223333/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control",
          "aws:SourceArn": "arn:aws:cloudtrail:ap-south-1:111122223333:trail/org-security-trail"
        }
      }
    }
  ]
}
```

Apply it:
```bash
aws s3api put-bucket-policy \
  --bucket my-cloudtrail-audit-bucket \
  --policy file://cloudtrail-bucket-policy.json
```

---

## 9. Athena Setup & Forensic Queries

**Create the table (partition projection — no manual partition maintenance):**
```sql
CREATE EXTERNAL TABLE cloudtrail_logs_prod (
  eventVersion STRING, userIdentity STRUCT<
    type:STRING, principalId:STRING, arn:STRING, accountId:STRING,
    userName:STRING, sessionContext:STRUCT<
      sessionIssuer:STRUCT<type:STRING,arn:STRING,userName:STRING>,
      attributes:STRUCT<mfaAuthenticated:STRING,creationDate:STRING>>>,
  eventTime STRING, eventSource STRING, eventName STRING,
  awsRegion STRING, sourceIPAddress STRING, userAgent STRING,
  errorCode STRING, errorMessage STRING,
  requestParameters STRING, responseElements STRING,
  requestId STRING, eventId STRING, readOnly STRING,
  eventType STRING, recipientAccountId STRING
)
PARTITIONED BY (region STRING, `date` STRING)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
LOCATION 's3://my-cloudtrail-audit-bucket/AWSLogs/111122223333/CloudTrail/'
TBLPROPERTIES (
  'projection.enabled'='true',
  'projection.region.type'='enum',
  'projection.region.values'='ap-south-1,us-east-1,eu-west-1',
  'projection.date.type'='date',
  'projection.date.range'='2025/01/01,NOW',
  'projection.date.format'='yyyy/MM/dd',
  'storage.location.template'='s3://my-cloudtrail-audit-bucket/AWSLogs/111122223333/CloudTrail/${region}/${date}'
);
```

**Forensic query bank:**
```sql
-- 1. Who deleted a DynamoDB table?
SELECT eventTime, userIdentity.arn, sourceIPAddress, requestParameters
FROM cloudtrail_logs_prod
WHERE eventName = 'DeleteTable' AND `date` >= '2026/07/01'
ORDER BY eventTime DESC;

-- 2. All actions from a suspicious IP
SELECT eventTime, eventName, eventSource, userIdentity.arn
FROM cloudtrail_logs_prod
WHERE sourceIPAddress = '203.0.113.42' AND `date` >= '2026/07/20';

-- 3. Access-denied bursts (permission probing) by principal
SELECT userIdentity.arn, count(*) AS denials
FROM cloudtrail_logs_prod
WHERE errorCode LIKE '%AccessDenied%' AND `date` >= '2026/07/25'
GROUP BY userIdentity.arn ORDER BY denials DESC;

-- 4. Every root-account action this month
SELECT eventTime, eventName, sourceIPAddress
FROM cloudtrail_logs_prod
WHERE userIdentity.type = 'Root' AND `date` >= '2026/07/01';

-- 5. Console logins and their MFA status
SELECT eventTime, userIdentity.userName, sourceIPAddress, responseElements
FROM cloudtrail_logs_prod
WHERE eventName = 'ConsoleLogin' AND `date` >= '2026/07/01';

-- 6. Who opened a security group to the world?
SELECT eventTime, userIdentity.arn, requestParameters
FROM cloudtrail_logs_prod
WHERE eventName = 'AuthorizeSecurityGroupIngress'
  AND requestParameters LIKE '%0.0.0.0/0%' AND `date` >= '2026/07/01';
```

---

## 10. CloudTrail Lake

```bash
# Create an event data store (7-year retention, org-wide, all regions)
aws cloudtrail create-event-data-store \
  --name org-audit-lake \
  --retention-period 2557 \
  --multi-region-enabled \
  --organization-enabled \
  --advanced-event-selectors '[{"Name":"mgmt","FieldSelectors":[{"Field":"eventCategory","Equals":["Management"]}]}]'

aws cloudtrail list-event-data-stores

# Run a SQL query against the lake
aws cloudtrail start-query \
  --query-statement "SELECT eventTime, userIdentity.arn, eventName FROM <eds-id> WHERE eventName='ConsoleLogin' ORDER BY eventTime DESC LIMIT 20"

# Fetch results
aws cloudtrail get-query-results --query-id <query-id>
```

---

## 11. Organization Trails

```bash
# From the ORG MANAGEMENT account (after enabling trusted access for CloudTrail):
aws organizations enable-aws-service-access --service-principal cloudtrail.amazonaws.com

aws cloudtrail create-trail \
  --name org-wide-trail \
  --s3-bucket-name central-log-archive-bucket \
  --is-multi-region-trail \
  --is-organization-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name org-wide-trail
```
