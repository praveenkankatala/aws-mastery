# Hands-On Labs — SQS · SNS · SES

Ten progressive labs. Do them in order — each one builds toward the full production-style pipeline in **Lab 8**. Every lab lists an objective, steps (console + CLI), expected output, and a cleanup note.

## 📑 Labs Overview

| # | Lab | Service(s) |
|---|---|---|
| 0 | Environment setup | IAM, CLI, boto3 |
| 1 | Create & use a Standard SQS queue | SQS |
| 2 | Create & use a FIFO SQS queue | SQS |
| 3 | Configure a Dead Letter Queue | SQS |
| 4 | Create an SNS topic + direct subscriptions | SNS |
| 5 | SNS → SQS fan-out pattern | SNS + SQS |
| 6 | SNS message filtering | SNS |
| 7 | Verify identities & send email in SES sandbox | SES |
| 8 | **Full pipeline:** SNS publish → SQS consume → SES email | SQS + SNS + SES |
| 9 | Monitoring with CloudWatch alarms | CloudWatch |
| 10 | Clean up all resources | All |

---

## Lab 0 — Environment Setup

**Objective:** Get your CLI and Python environment ready.

```bash
aws configure                     # enter access key, secret key, region (e.g. us-east-1)
aws sts get-caller-identity       # confirm you're authenticated
pip install boto3
```

Create a working IAM user/role (or use one already scoped) with permissions for `sqs:*`, `sns:*`, `ses:*` for the duration of these labs — tighten to least-privilege afterward using the policies in `commands-cheatsheet.md`.

**Expected output:** `get-caller-identity` returns your Account ID, User ID, and ARN.

---

## Lab 1 — Standard SQS Queue

**Objective:** Create a queue and manually send/receive/delete a message.

**Console:**
1. Open SQS Console → **Create queue**
2. Type: **Standard**
3. Name: `Order-Processing-Queue`
4. Configuration: **Visibility timeout** = 30 sec, **Message retention** = 4 days (default)
5. Click **Create queue**

**CLI equivalent:**
```bash
aws sqs create-queue --queue-name Order-Processing-Queue \
  --attributes VisibilityTimeout=30

QUEUE_URL=$(aws sqs get-queue-url --queue-name Order-Processing-Queue --query QueueUrl --output text)

aws sqs send-message --queue-url $QUEUE_URL --message-body '{"order_id": 1001}'

aws sqs receive-message --queue-url $QUEUE_URL --wait-time-seconds 10
```

**Expected output:** `receive-message` returns your JSON body plus a `ReceiptHandle`. Use it to delete:
```bash
aws sqs delete-message --queue-url $QUEUE_URL --receipt-handle "<paste handle>"
```

**Try it:** Receive the same message twice without deleting it — notice it disappears for the visibility timeout window (30s), then reappears. This *is* the at-least-once guarantee in action.

---

## Lab 2 — FIFO SQS Queue

**Objective:** See strict ordering and exactly-once processing in action.

```bash
aws sqs create-queue --queue-name Orders.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=true

FIFO_URL=$(aws sqs get-queue-url --queue-name Orders.fifo --query QueueUrl --output text)

# Send 3 messages, same MessageGroupId → strict order guaranteed within the group
aws sqs send-message --queue-url $FIFO_URL --message-body "Order A" --message-group-id "customer-42"
aws sqs send-message --queue-url $FIFO_URL --message-body "Order B" --message-group-id "customer-42"
aws sqs send-message --queue-url $FIFO_URL --message-body "Order C" --message-group-id "customer-42"

aws sqs receive-message --queue-url $FIFO_URL --max-number-of-messages 3
```

**Expected output:** Messages return in the exact order A → B → C (guaranteed only within the same `MessageGroupId`; different groups can interleave).

**Note:** Sending the exact same message body + `MessageGroupId` again within the 5-minute dedup window (with `ContentBasedDeduplication=true`) is silently dropped as a duplicate — try it and confirm only 3 unique messages ever existed.

---

## Lab 3 — Dead Letter Queue (DLQ)

**Objective:** Simulate a failing consumer and watch messages move to a DLQ automatically.

```bash
# 1. Create the DLQ
aws sqs create-queue --queue-name Order-Processing-DLQ
DLQ_ARN=$(aws sqs get-queue-attributes --queue-url \
  $(aws sqs get-queue-url --queue-name Order-Processing-DLQ --query QueueUrl --output text) \
  --attribute-names QueueArn --query Attributes.QueueArn --output text)

# 2. Attach it to the source queue with maxReceiveCount=3
aws sqs set-queue-attributes --queue-url $QUEUE_URL --attributes '{
  "RedrivePolicy": "{\"deadLetterTargetArn\":\"'"$DLQ_ARN"'\",\"maxReceiveCount\":\"3\"}"
}'

# 3. Send a "poison pill" message and receive it 3 times WITHOUT deleting it
aws sqs send-message --queue-url $QUEUE_URL --message-body '{"bad":"data"}'
for i in 1 2 3; do
  aws sqs receive-message --queue-url $QUEUE_URL --visibility-timeout 1
  sleep 2
done

# 4. Check the DLQ — the message should now be there
aws sqs receive-message --queue-url $(aws sqs get-queue-url --queue-name Order-Processing-DLQ --query QueueUrl --output text)
```

**Expected output:** After the 3rd failed receive, the message no longer exists in the source queue and appears in `Order-Processing-DLQ`.

---

## Lab 4 — SNS Topic with Direct Subscriptions

**Objective:** Broadcast a message to email and (optionally) SMS.

```bash
aws sns create-topic --name User-Signups
TOPIC_ARN=$(aws sns list-topics --query "Topics[?contains(TopicArn,'User-Signups')].TopicArn" --output text)

aws sns subscribe --topic-arn $TOPIC_ARN --protocol email --notification-endpoint you@example.com
```

**Manual step:** Check your inbox and click **Confirm subscription** — SNS subscriptions stay `PendingConfirmation` until confirmed (this trips up almost everyone the first time).

```bash
aws sns publish --topic-arn $TOPIC_ARN --subject "Test" --message "Hello from SNS!"
```

**Expected output:** An email arrives in your inbox within seconds.

---

## Lab 5 — SNS → SQS Fan-out

**Objective:** One publish, multiple queues receive a copy — the core decoupling pattern.

```bash
# Create 2 queues
aws sqs create-queue --queue-name Inventory-Queue
aws sqs create-queue --queue-name Analytics-Queue

INV_ARN=$(aws sqs get-queue-attributes --queue-url $(aws sqs get-queue-url --queue-name Inventory-Queue --query QueueUrl --output text) --attribute-names QueueArn --query Attributes.QueueArn --output text)
ANA_ARN=$(aws sqs get-queue-attributes --queue-url $(aws sqs get-queue-url --queue-name Analytics-Queue --query QueueUrl --output text) --attribute-names QueueArn --query Attributes.QueueArn --output text)

# Subscribe both to the topic
aws sns subscribe --topic-arn $TOPIC_ARN --protocol sqs --notification-endpoint $INV_ARN
aws sns subscribe --topic-arn $TOPIC_ARN --protocol sqs --notification-endpoint $ANA_ARN
```

⚠️ **Important:** Unlike email/SMS subscriptions, SQS subscriptions need the queue's **access policy** to explicitly allow the SNS topic to deliver to it. The console does this automatically; via CLI you must attach it yourself — see the "SNS → SQS resource policy" snippet in `commands-cheatsheet.md`. Skipping this is the #1 cause of "I subscribed but no messages arrive" (see `troubleshooting.md`).

```bash
aws sns publish --topic-arn $TOPIC_ARN --message '{"event":"ORDER_COMPLETED","order_id":1001}'

# Check both queues — each should have received an independent copy
aws sqs receive-message --queue-url $(aws sqs get-queue-url --queue-name Inventory-Queue --query QueueUrl --output text)
aws sqs receive-message --queue-url $(aws sqs get-queue-url --queue-name Analytics-Queue --query QueueUrl --output text)
```

**Expected output:** Both queues contain the *same* message body, wrapped in an SNS envelope (`Type`, `MessageId`, `TopicArn`, `Message`, `Timestamp`, `SignatureVersion`, etc.).

---

## Lab 6 — SNS Message Filtering

**Objective:** Only deliver relevant events to a subscriber instead of everything on the topic.

```bash
SUB_ARN=$(aws sns list-subscriptions-by-topic --topic-arn $TOPIC_ARN \
  --query "Subscriptions[?Endpoint=='$INV_ARN'].SubscriptionArn" --output text)

# Only forward messages where status == "shipped"
aws sns set-subscription-attributes \
  --subscription-arn $SUB_ARN \
  --attribute-name FilterPolicy \
  --attribute-value '{"status": ["shipped"]}'

# This one is filtered OUT (no matching attribute)
aws sns publish --topic-arn $TOPIC_ARN --message "order created" \
  --message-attributes '{"status":{"DataType":"String","StringValue":"created"}}'

# This one is filtered IN
aws sns publish --topic-arn $TOPIC_ARN --message "order shipped" \
  --message-attributes '{"status":{"DataType":"String","StringValue":"shipped"}}'
```

**Expected output:** Only the "order shipped" message shows up in the Inventory queue.

---

## Lab 7 — SES Sandbox: Verify & Send

**Objective:** Send your first programmatic email.

```bash
# Verify a sender AND a recipient address (sandbox requires both)
aws ses verify-email-identity --email-address sender@example.com
aws ses verify-email-identity --email-address recipient@example.com
```

**Manual step:** Click the verification link AWS emails to each address.

```bash
aws ses get-identity-verification-attributes --identities sender@example.com recipient@example.com

aws ses send-email \
  --from sender@example.com \
  --destination "ToAddresses=recipient@example.com" \
  --message '{"Subject":{"Data":"Hello from SES"},"Body":{"Text":{"Data":"This is a sandbox test email."}}}'

aws ses get-send-quota
```

**Expected output:** `get-send-quota` shows `Max24HourSend: 200.0`, `MaxSendRate: 1.0` — confirming sandbox mode. The email arrives in the recipient's inbox.

---

## Lab 8 — Full Pipeline: SNS → SQS → SES (Python)

**Objective:** Wire everything together — the architecture diagram from `README.md`, running as real code.

### Prerequisites
- `Order-Processing-Queue` exists (Lab 1) and is subscribed to `User-Signups` topic (Lab 5, adapted)
- Sender + recipient verified in SES (Lab 7)

### Step 1 — The Publisher (`sns_publisher.py`)
Acts as your application backend (e.g. a signup service). Publishes an event that SNS fans out to SQS automatically.

```python
import boto3
import json

sns_client = boto3.client('sns', region_name='us-east-1')

TOPIC_ARN = 'arn:aws:sns:us-east-1:123456789012:User-Signups'

def publish_signup_event(user_id, email):
    message_body = {
        'event': 'USER_SIGNUP',
        'user_id': user_id,
        'email': email
    }
    print("Publishing event to SNS...")
    response = sns_client.publish(
        TopicArn=TOPIC_ARN,
        Message=json.dumps(message_body),
        Subject='New User Registration'
    )
    print(f"Message published! MessageId: {response['MessageId']}")

publish_signup_event(user_id='usr_98765', email='newuser@example.com')
```

### Step 2 — The Worker (`sqs_worker_with_ses.py`)
Polls SQS, unwraps the SNS envelope, and fires a real welcome email through SES.

```python
import boto3
import json
import time
from botocore.exceptions import ClientError

sqs_client = boto3.client('sqs', region_name='us-east-1')
ses_client = boto3.client('ses', region_name='us-east-1')

QUEUE_URL = 'https://sqs.us-east-1.amazonaws.com/123456789012/Order-Processing-Queue'
SENDER_EMAIL = 'verified-sender@example.com'  # must be verified in SES


def send_welcome_email(recipient_email, user_id):
    """Uses AWS SES to send a transactional email."""
    print(f"Sending welcome email to {recipient_email}...")

    CHARSET = "UTF-8"
    SUBJECT = "Welcome to Our Platform!"

    BODY_HTML = f"""<html>
    <head></head>
    <body>
      <h1>Welcome aboard!</h1>
      <p>Thank you for signing up. Your User ID is <strong>{user_id}</strong>.</p>
      <p>We are thrilled to have you here.</p>
    </body>
    </html>"""

    BODY_TEXT = f"Welcome aboard!\nThank you for signing up. Your User ID is {user_id}."

    try:
        response = ses_client.send_email(
            Destination={'ToAddresses': [recipient_email]},
            Message={
                'Body': {
                    'Html': {'Charset': CHARSET, 'Data': BODY_HTML},
                    'Text': {'Charset': CHARSET, 'Data': BODY_TEXT},
                },
                'Subject': {'Charset': CHARSET, 'Data': SUBJECT},
            },
            Source=SENDER_EMAIL,
        )
    except ClientError as e:
        print(f"SES Error: {e.response['Error']['Message']}")
        return False
    else:
        print(f"Email sent! Message ID: {response['MessageId']}")
        return True


def process_messages():
    print("Polling SQS queue for messages...")

    while True:
        response = sqs_client.receive_message(
            QueueUrl=QUEUE_URL,
            MaxNumberOfMessages=1,
            WaitTimeSeconds=20,       # long polling
            VisibilityTimeout=45      # gives SES time to respond before retry
        )

        if 'Messages' not in response:
            print("No messages in queue. Waiting...")
            time.sleep(5)
            continue

        for message in response['Messages']:
            # Unwrap the SNS envelope inside the SQS message body
            sns_envelope = json.loads(message['Body'])
            actual_event = json.loads(sns_envelope['Message'])

            print(f"\n--- Processing Event for User: {actual_event['user_id']} ---")

            email_success = send_welcome_email(
                recipient_email=actual_event['email'],
                user_id=actual_event['user_id']
            )

            if email_success:
                sqs_client.delete_message(
                    QueueUrl=QUEUE_URL,
                    ReceiptHandle=message['ReceiptHandle']
                )
                print("Processed and removed message from queue.")
            else:
                print("Processing failed — message stays in queue for retry.")


if __name__ == "__main__":
    try:
        process_messages()
    except KeyboardInterrupt:
        print("\nWorker stopped.")
```

### Step 3 — Run It
```bash
# Terminal 1 — start the worker, it waits patiently
python sqs_worker_with_ses.py

# Terminal 2 — trigger a signup event
python sns_publisher.py
```

**Expected output:** Terminal 1 immediately receives the message, unwraps it, sends the email via SES, prints the `MessageId`, and deletes the SQS message.

### Why This Design Is Production-Grade
- **Failure isolation** — if SES throttles or errors, `email_success = False` and the message is *not* deleted.
- **Safe automatic retries** — because it wasn't deleted, the message becomes visible again after the 45s `VisibilityTimeout` and gets retried.
- **DLQ as a safety net** — wire the DLQ from Lab 3 onto this queue; a permanently broken message (e.g. malformed address) lands there after a few attempts instead of looping forever.

---

## Lab 9 — Monitoring with CloudWatch

**Objective:** Get alerted before a silent failure becomes a customer-facing outage.

```bash
# Alarm if the DLQ ever receives a message (should normally be 0)
aws cloudwatch put-metric-alarm \
  --alarm-name DLQ-Has-Messages \
  --namespace AWS/SQS \
  --metric-name ApproximateNumberOfMessagesVisible \
  --dimensions Name=QueueName,Value=Order-Processing-DLQ \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions <SNS_TOPIC_ARN_FOR_ALERTS>

# Alarm if the main queue backs up (consumers falling behind)
aws cloudwatch put-metric-alarm \
  --alarm-name Queue-Backlog-High \
  --namespace AWS/SQS \
  --metric-name ApproximateNumberOfMessagesVisible \
  --dimensions Name=QueueName,Value=Order-Processing-Queue \
  --statistic Average \
  --period 300 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions <SNS_TOPIC_ARN_FOR_ALERTS>
```

Also monitor SES reputation via `Reputation.BounceRate` / `Reputation.ComplaintRate` in the **SES Console → Reputation dashboard**, or subscribe an SNS topic to bounce/complaint events (see `commands-cheatsheet.md` → configuration sets).

---

## Lab 10 — Clean Up

**Objective:** Avoid leftover resources (SQS/SNS are nearly free at this scale, but good hygiene matters).

```bash
# Delete queues
for q in Order-Processing-Queue Order-Processing-DLQ Orders.fifo Inventory-Queue Analytics-Queue; do
  URL=$(aws sqs get-queue-url --queue-name $q --query QueueUrl --output text 2>/dev/null)
  [ -n "$URL" ] && aws sqs delete-queue --queue-url $URL
done

# Delete SNS topic (this also removes its subscriptions)
aws sns delete-topic --topic-arn $TOPIC_ARN

# Remove CloudWatch alarms
aws cloudwatch delete-alarms --alarm-names DLQ-Has-Messages Queue-Backlog-High

# SES identities can stay verified for future testing, or remove with:
aws ses delete-identity --identity sender@example.com
aws ses delete-identity --identity recipient@example.com
```

---

**Next:** Something not behaving as expected? Check [`troubleshooting.md`](./troubleshooting.md) before assuming it's a bug — 90% of issues here are permissions or confirmation steps.
