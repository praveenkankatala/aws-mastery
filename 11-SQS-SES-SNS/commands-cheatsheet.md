# AWS CLI Commands Cheatsheet — SQS · SNS · SES

Quick reference for every command used across the labs, plus IAM policies and boto3 equivalents. All examples assume `us-east-1` — swap the region as needed.

## 📑 Table of Contents
- [Setup](#setup)
- [SQS Commands](#sqs-commands)
- [SNS Commands](#sns-commands)
- [SES Commands](#ses-commands)
- [IAM Policy Examples](#iam-policy-examples)
- [boto3 Quick Reference](#boto3-quick-reference)

---

## Setup

```bash
# Install AWS CLI v2 (Linux example)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Configure credentials
aws configure
# AWS Access Key ID, Secret Access Key, default region (e.g. us-east-1), output format (json)

# Verify identity
aws sts get-caller-identity

# Python SDK
pip install boto3
```

---

## SQS Commands

### Create Queues
```bash
# Standard queue
aws sqs create-queue --queue-name Order-Processing-Queue

# FIFO queue (name MUST end in .fifo)
aws sqs create-queue --queue-name Order-Processing-Queue.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=true

# FIFO queue with High Throughput Mode
aws sqs create-queue --queue-name HighThroughput-Queue.fifo \
  --attributes FifoQueue=true,DeduplicationScope=messageGroup,FifoThroughputLimit=perMessageGroupId

# Queue with visibility timeout + retention configured at creation
aws sqs create-queue --queue-name My-Queue \
  --attributes VisibilityTimeout=30,MessageRetentionPeriod=345600
```

### Inspect & Manage Queues
```bash
# List all queues
aws sqs list-queues

# Get queue URL from name
aws sqs get-queue-url --queue-name Order-Processing-Queue

# Get every attribute (ARN, message counts, etc.)
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/Order-Processing-Queue \
  --attribute-names All

# Update an attribute (e.g. visibility timeout)
aws sqs set-queue-attributes \
  --queue-url <QUEUE_URL> \
  --attributes VisibilityTimeout=45

# Tag a queue
aws sqs tag-queue --queue-url <QUEUE_URL> --tags Project=MessagingDemo,Env=dev

# Purge all messages (irreversible, use with care)
aws sqs purge-queue --queue-url <QUEUE_URL>

# Delete the queue entirely
aws sqs delete-queue --queue-url <QUEUE_URL>
```

### Send / Receive / Delete Messages
```bash
# Send a message (Standard queue)
aws sqs send-message \
  --queue-url <QUEUE_URL> \
  --message-body '{"event":"USER_SIGNUP","user_id":"usr_1"}'

# Send a message to a FIFO queue (requires MessageGroupId)
aws sqs send-message \
  --queue-url <FIFO_QUEUE_URL> \
  --message-body '{"event":"ORDER_PLACED"}' \
  --message-group-id "orders" \
  --message-deduplication-id "order-1001"

# Receive messages (long polling, up to 10 at a time)
aws sqs receive-message \
  --queue-url <QUEUE_URL> \
  --max-number-of-messages 10 \
  --wait-time-seconds 20 \
  --visibility-timeout 30

# Delete a message after processing (ReceiptHandle from receive-message output)
aws sqs delete-message \
  --queue-url <QUEUE_URL> \
  --receipt-handle "<RECEIPT_HANDLE>"

# Change visibility timeout of an in-flight message
aws sqs change-message-visibility \
  --queue-url <QUEUE_URL> \
  --receipt-handle "<RECEIPT_HANDLE>" \
  --visibility-timeout 60

# Send in a batch (up to 10 messages, cheaper)
aws sqs send-message-batch \
  --queue-url <QUEUE_URL> \
  --entries file://batch-entries.json
```

### Dead Letter Queues (DLQ)
```bash
# 1. Create the DLQ first
aws sqs create-queue --queue-name Order-Processing-DLQ

# 2. Get its ARN
aws sqs get-queue-attributes --queue-url <DLQ_URL> --attribute-names QueueArn

# 3. Attach the DLQ to the source queue via RedrivePolicy
aws sqs set-queue-attributes \
  --queue-url <SOURCE_QUEUE_URL> \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"<DLQ_ARN>\",\"maxReceiveCount\":\"3\"}"
  }'

# Redrive messages back from DLQ to source queue (after fixing the bug)
aws sqs start-message-move-task --source-arn <DLQ_ARN>
```

---

## SNS Commands

### Topics
```bash
# Create a standard topic
aws sns create-topic --name User-Signups

# Create a FIFO topic (must end in .fifo)
aws sns create-topic --name Order-Events.fifo \
  --attributes FifoTopic=true,ContentBasedDeduplication=true

# List topics
aws sns list-topics

# Delete a topic
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:123456789012:User-Signups

# Set a topic attribute (e.g. display name for SMS)
aws sns set-topic-attributes \
  --topic-arn <TOPIC_ARN> \
  --attribute-name DisplayName \
  --attribute-value "MyApp"
```

### Subscriptions
```bash
# Subscribe an SQS queue (protocol=sqs, endpoint=queue ARN)
aws sns subscribe \
  --topic-arn <TOPIC_ARN> \
  --protocol sqs \
  --notification-endpoint <SQS_QUEUE_ARN>

# Subscribe an email address (requires manual confirmation via emailed link)
aws sns subscribe \
  --topic-arn <TOPIC_ARN> \
  --protocol email \
  --notification-endpoint you@example.com

# Subscribe an SMS number
aws sns subscribe \
  --topic-arn <TOPIC_ARN> \
  --protocol sms \
  --notification-endpoint +15555550123

# Subscribe a Lambda function
aws sns subscribe \
  --topic-arn <TOPIC_ARN> \
  --protocol lambda \
  --notification-endpoint <LAMBDA_ARN>

# List subscriptions for a topic
aws sns list-subscriptions-by-topic --topic-arn <TOPIC_ARN>

# Unsubscribe
aws sns unsubscribe --subscription-arn <SUBSCRIPTION_ARN>
```

### Message Filtering
```bash
# Attach a filter policy to a subscription — only deliver messages
# where the "status" message attribute equals "shipped"
aws sns set-subscription-attributes \
  --subscription-arn <SUBSCRIPTION_ARN> \
  --attribute-name FilterPolicy \
  --attribute-value '{"status": ["shipped"]}'
```

### Publishing
```bash
# Publish a plain message
aws sns publish \
  --topic-arn <TOPIC_ARN> \
  --message "New user signed up" \
  --subject "User Signup Event"

# Publish with message attributes (used for filtering)
aws sns publish \
  --topic-arn <TOPIC_ARN> \
  --message '{"event":"ORDER_SHIPPED","order_id":"1001"}' \
  --message-attributes '{"status":{"DataType":"String","StringValue":"shipped"}}'

# Publish to a FIFO topic (requires MessageGroupId)
aws sns publish \
  --topic-arn <FIFO_TOPIC_ARN> \
  --message "Order 1001 placed" \
  --message-group-id "orders" \
  --message-deduplication-id "order-1001"

# Publish an SMS directly (no topic needed)
aws sns publish --phone-number +15555550123 --message "Your OTP is 482913"
```

---

## SES Commands

### Identity Verification
```bash
# Verify a single email address (sandbox testing)
aws ses verify-email-identity --email-address you@example.com

# Verify an entire domain (production — required for DKIM on all addresses)
aws ses verify-domain-identity --domain example.com

# Set up Easy DKIM for a domain (returns 3 CNAME tokens to add to DNS)
aws ses verify-domain-dkim --domain example.com

# Check verification status
aws ses get-identity-verification-attributes --identities example.com you@example.com

# List all identities
aws ses list-identities

# Delete an identity
aws ses delete-identity --identity you@example.com
```

### Sending Email
```bash
# Simple email
aws ses send-email \
  --from "verified-sender@example.com" \
  --destination "ToAddresses=recipient@example.com" \
  --message '{
    "Subject": {"Data": "Welcome!"},
    "Body": {"Text": {"Data": "Thanks for signing up."}}
  }'

# Raw email (needed for attachments / custom MIME)
aws ses send-raw-email --raw-message file://raw-email.txt

# Templated bulk email (up to 50 destinations per call)
aws ses send-bulk-templated-email \
  --source "verified-sender@example.com" \
  --template "WelcomeTemplate" \
  --default-template-data '{"name":"there"}' \
  --destinations file://bulk-destinations.json
```

### Quotas, Stats & Configuration Sets
```bash
# Check current sandbox/production sending quota + rate
aws ses get-send-quota

# Check sending statistics (bounces, complaints, rejects)
aws ses get-send-statistics

# Create a configuration set (for event tracking)
aws ses create-configuration-set --configuration-set '{"Name":"prod-tracking"}'

# Route bounce/complaint events from a config set to an SNS topic
aws ses create-configuration-set-event-destination \
  --configuration-set-name prod-tracking \
  --event-destination '{
    "Name": "bounce-complaint-notifications",
    "Enabled": true,
    "MatchingEventTypes": ["bounce", "complaint"],
    "SNSDestination": {"TopicARN": "<TOPIC_ARN>"}
  }'
```

### Receiving Email (Inbound)
```bash
# Create a receipt rule set
aws ses create-receipt-rule-set --rule-set-name default-inbound

# Set it as active
aws ses set-active-receipt-rule-set --rule-set-name default-inbound

# Add a rule that stores incoming mail in S3
aws ses create-receipt-rule \
  --rule-set-name default-inbound \
  --rule '{
    "Name": "store-support-emails",
    "Enabled": true,
    "Recipients": ["support@example.com"],
    "Actions": [{"S3Action": {"BucketName": "my-inbound-email-bucket"}}]
  }'
```

---

## IAM Policy Examples

### Minimal SQS worker policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:Order-Processing-Queue"
    }
  ]
}
```

### Minimal SNS publisher policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["sns:Publish"],
      "Resource": "arn:aws:sns:us-east-1:123456789012:User-Signups"
    }
  ]
}
```

### Minimal SES sending policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ses:SendEmail", "ses:SendRawEmail"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {"ses:FromAddress": "verified-sender@example.com"}
      }
    }
  ]
}
```

### SNS → SQS resource policy (attach to the SQS queue itself)
This is the piece people most often forget when wiring SNS→SQS by CLI/IaC instead of the console — the queue must explicitly allow the topic to deliver to it.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSNSPublish",
      "Effect": "Allow",
      "Principal": {"Service": "sns.amazonaws.com"},
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:123456789012:Order-Processing-Queue",
      "Condition": {
        "ArnEquals": {"aws:SourceArn": "arn:aws:sns:us-east-1:123456789012:User-Signups"}
      }
    }
  ]
}
```
Attach it with:
```bash
aws sqs set-queue-attributes --queue-url <QUEUE_URL> \
  --attributes file://sqs-policy.json
```

---

## boto3 Quick Reference

```python
import boto3

sqs = boto3.client('sqs', region_name='us-east-1')
sns = boto3.client('sns', region_name='us-east-1')
ses = boto3.client('ses', region_name='us-east-1')

# SQS: send
sqs.send_message(QueueUrl=QUEUE_URL, MessageBody='{"hello":"world"}')

# SQS: receive + delete
resp = sqs.receive_message(QueueUrl=QUEUE_URL, MaxNumberOfMessages=1, WaitTimeSeconds=20)
for msg in resp.get('Messages', []):
    print(msg['Body'])
    sqs.delete_message(QueueUrl=QUEUE_URL, ReceiptHandle=msg['ReceiptHandle'])

# SNS: publish
sns.publish(TopicArn=TOPIC_ARN, Message='Hello subscribers', Subject='Test')

# SNS: subscribe SQS queue
sns.subscribe(TopicArn=TOPIC_ARN, Protocol='sqs', Endpoint=QUEUE_ARN)

# SES: send email
ses.send_email(
    Source='verified-sender@example.com',
    Destination={'ToAddresses': ['recipient@example.com']},
    Message={
        'Subject': {'Data': 'Welcome!'},
        'Body': {'Text': {'Data': 'Thanks for signing up.'}}
    }
)
```

---

**Tip:** Every command above uses placeholders like `<QUEUE_URL>`, `<TOPIC_ARN>`. Grab the real values from `list-queues` / `list-topics` / the console, or store them as environment variables while working through `hands-on-labs.md`.
