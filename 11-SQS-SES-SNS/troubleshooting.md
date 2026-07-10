# Troubleshooting Guide — SQS · SNS · SES

Real errors you'll hit while working through the labs, why they happen, and how to fix them.

## 📑 Table of Contents
- [SQS Issues](#sqs-issues)
- [SNS Issues](#sns-issues)
- [SES Issues](#ses-issues)
- [Cross-Service / IAM Issues](#cross-service--iam-issues)
- [Debugging Checklist](#debugging-checklist)
- [FAQ](#faq)

---

## SQS Issues

### ❌ "I processed the message but it came back again"
**Cause:** You forgot to call `delete_message()` / `delete-message`, or your processing took longer than the `VisibilityTimeout`, so SQS assumed the consumer died and made the message visible again.
**Fix:**
- Always delete the message only *after* processing succeeds.
- Set `VisibilityTimeout` comfortably longer than your worst-case processing time (e.g. if SES calls can take a few seconds, use 30–45s, not 5s).
- For long-running jobs, call `change-message-visibility` periodically to extend the timeout ("heartbeat" pattern) instead of setting one huge timeout upfront.

### ❌ "I'm getting duplicate messages on a Standard queue"
**Cause:** This is expected behavior — Standard queues guarantee **at-least-once** delivery, not exactly-once. Duplicates happen especially under high throughput or network retries.
**Fix:** Design consumers to be **idempotent** (e.g. check if `order_id` was already processed before acting on it), or switch to a **FIFO queue** if you truly need exactly-once semantics (at the cost of throughput).

### ❌ `receive-message` returns nothing even though I know a message was sent
**Cause 1:** Short polling (`WaitTimeSeconds=0`, the default) only checks a subset of servers per call and can return empty even when messages exist.
**Fix:** Use long polling — set `WaitTimeSeconds` to 10–20.

**Cause 2:** The message is currently in-flight (another consumer/test call already received it and its visibility timeout hasn't expired).
**Fix:** Wait out the visibility timeout, or check `ApproximateNumberOfMessages` vs `ApproximateNumberOfMessagesNotVisible` via `get-queue-attributes`.

### ❌ `AccessDenied` when sending/receiving
**Cause:** The IAM identity making the call lacks `sqs:SendMessage` / `sqs:ReceiveMessage` / `sqs:DeleteMessage` on that specific queue ARN.
**Fix:** Attach the minimal policy from `commands-cheatsheet.md`, double-check the **Resource ARN** matches the queue exactly (region + account ID + name).

### ❌ Messages never reach the DLQ even though processing keeps failing
**Cause:** `RedrivePolicy` wasn't set, or `maxReceiveCount` is set too high, or the DLQ ARN in the policy is wrong/points to a queue of the wrong type (Standard source queue needs a Standard DLQ; FIFO needs a FIFO DLQ).
**Fix:** Re-verify with `get-queue-attributes --attribute-names RedrivePolicy`, and confirm the DLQ type matches the source queue type.

### ❌ `InvalidParameterValue` on a FIFO queue send
**Cause:** Missing `MessageGroupId` (required for every FIFO send), or reusing a `MessageDeduplicationId` unintentionally.
**Fix:** Always pass `--message-group-id`. If not using `ContentBasedDeduplication`, pass a unique `--message-deduplication-id` per logically-distinct message.

### ❌ `OverLimit: Too many in-flight messages`
**Cause:** You've exceeded 120,000 in-flight messages (Standard) or 20,000 (FIFO) — usually means consumers are receiving faster than they delete.
**Fix:** Delete messages promptly after processing; split load across multiple queues; scale up consumers.

---

## SNS Issues

### ❌ "I subscribed my email but I never get notifications"
**Cause:** Email/SMS-HTTP(S) subscriptions sit in `PendingConfirmation` state until the recipient clicks the confirmation link AWS sends. Publishes to unconfirmed subscriptions are silently skipped.
**Fix:** Check `list-subscriptions-by-topic` — look for `SubscriptionArn: PendingConfirmation`. Resend/click the confirmation email.

### ❌ "I subscribed an SQS queue but messages never arrive there"
**Cause (the #1 SNS→SQS gotcha):** The SQS queue's **access policy** doesn't grant the SNS topic permission to call `sqs:SendMessage` on it. The AWS Console does this automatically when you subscribe through the UI; the raw API/CLI does **not**.
**Fix:** Attach the resource policy shown in `commands-cheatsheet.md` under "SNS → SQS resource policy" directly to the queue, referencing the topic's ARN as `aws:SourceArn`.

### ❌ Filter policy isn't filtering anything
**Cause:** `message-attributes` weren't included in the `publish` call — filter policies match on **message attributes**, not on the message body content.
**Fix:** Always pass `--message-attributes` with a matching key/value when publishing to a filtered topic; verify the exact attribute name and value types match the filter policy JSON.

### ❌ Messages arrive out of order on a "FIFO" setup
**Cause:** Either you're actually using a Standard topic (name doesn't end in `.fifo`), or messages are spread across different `MessageGroupId`s (ordering is only guaranteed *within* a group, not across groups).
**Fix:** Confirm the topic name ends in `.fifo` and `FifoTopic=true`; use a consistent `MessageGroupId` for messages that must stay ordered relative to each other.

### ❌ `InvalidParameter: Invalid parameter: TopicArn or TargetArn Reason: no value for either`
**Cause:** Neither `--topic-arn`, `--target-arn`, nor `--phone-number` was supplied to `publish`.
**Fix:** Provide exactly one destination parameter.

---

## SES Issues

### ❌ `MessageRejected: Email address is not verified`
**Cause:** You're in the SES **Sandbox** and either the sender or recipient (or both) isn't a verified identity.
**Fix:** Verify both addresses (`verify-email-identity`) and click the confirmation link in each inbox, or request **Production Access** to lift the "verified recipients only" restriction.

### ❌ `Throttling: Maximum sending rate exceeded`
**Cause:** You're sending faster than your account's `MaxSendRate` (1/sec in sandbox; higher after production approval, but still capped).
**Fix:** Add client-side throttling/backoff between sends; check current limits with `aws ses get-send-quota`; request a quota increase once your reputation is established.

### ❌ `Daily message quota exceeded`
**Cause:** You hit the 200 emails/24h sandbox ceiling (or your production quota).
**Fix:** Wait for the rolling 24h window to reset, or request a quota increase via **Request production access** / a Support case.

### ❌ Emails land in Spam/Junk instead of the inbox
**Cause:** Missing or misconfigured **SPF**, **DKIM**, or **DMARC** records for your sending domain, or a cold/unestablished sender reputation.
**Fix:**
- Verify the **domain** (not just an address) and enable **Easy DKIM** (`verify-domain-dkim`) — then add the 3 returned CNAME records to your DNS.
- Publish an SPF TXT record authorizing Amazon SES to send on your domain's behalf.
- Add a DMARC policy once SPF/DKIM are passing.
- Warm up sending volume gradually rather than blasting large volumes on a brand-new domain.

### ❌ Can't get out of the Sandbox
**Cause:** Production access requests need a clear use case description (marketing vs. transactional), a website URL, and an explanation of how you'll handle bounces/complaints/unsubscribes.
**Fix:** Submit the request from **SES Console → Account dashboard → Request production access**, answering all fields thoroughly. Approval typically takes 24–48 hours.

### ❌ High bounce or complaint rate warning / account under review
**Cause:** Sending to invalid, unengaged, or purchased email lists.
**Fix:** Subscribe an SNS topic to bounce/complaint events (via a **Configuration Set**) and automatically suppress those addresses from future sends. Keep bounce rate under ~5% and complaint rate under ~0.1%.

### ❌ `send-raw-email` attachment makes the message get rejected for size
**Cause:** SES caps total message size (post-MIME-encoding) at 10 MB; MIME/base64 encoding inflates a file's size by roughly 37%.
**Fix:** Host large attachments externally (e.g. S3 pre-signed URL) and link to them instead of attaching directly.

---

## Cross-Service / IAM Issues

### ❌ `AccessDenied` calling any of the three services from Lambda/EC2
**Cause:** The execution role attached to the compute resource doesn't include the relevant `sqs:*` / `sns:*` / `ses:*` permissions.
**Fix:** Attach an inline or managed policy to the **role**, not just an IAM user — compute resources assume roles, they don't use user credentials.

### ❌ Everything "works" in the console but fails via CLI/SDK
**Cause:** Different credentials/region are active in your CLI profile than what you tested in the console (which uses your browser session's region).
**Fix:** Explicitly pass `--region` on every command, or set `AWS_DEFAULT_REGION`; double check `aws configure list`.

### ❌ ARN typos ("QueueDoesNotExist", "NotFound", "InvalidClientTokenId")
**Cause:** Copy-paste error, wrong AWS account ID in the ARN, or resource created in a different region than you're querying.
**Fix:** Always fetch ARNs programmatically (`get-queue-url` → `get-queue-attributes`, `list-topics`) rather than hand-typing them.

---

## Debugging Checklist

When something isn't working, check in this order:

1. **Region** — is the resource and your CLI/SDK call in the same region?
2. **ARN/URL correctness** — did you copy the exact ARN/URL, not a guessed one?
3. **IAM permissions** — does the caller's role/user have the specific action allowed on that specific resource?
4. **Resource policy** — for SNS→SQS, does the queue's own access policy trust the topic?
5. **Confirmation state** — for SNS email/SMS/HTTP subscriptions, is it confirmed (not `PendingConfirmation`)?
6. **Sandbox restrictions** — for SES, are both sender and recipient verified, and are you under the rate/daily quota?
7. **Visibility timeout / in-flight state** — for SQS, is the message currently invisible because another receive already grabbed it?
8. **CloudWatch metrics** — `NumberOfMessagesSent`, `NumberOfMessagesReceived`, `ApproximateNumberOfMessagesVisible` (SQS); `NumberOfMessagesPublished`, `NumberOfNotificationsFailed` (SNS); `Send`, `Bounce`, `Complaint`, `Reject` (SES) will usually confirm exactly where the pipeline is breaking.

---

## FAQ

**Q: Standard or FIFO — which should I default to?**
A: Standard, unless you have a specific need for strict ordering or hard exactly-once guarantees. Standard is cheaper, has far higher throughput ceilings, and is simpler to operate; most systems can tolerate at-least-once delivery with idempotent consumers.

**Q: Can SNS deliver directly to an email inbox without SES?**
A: Yes — SNS supports an `email` protocol subscription directly. But SES is the better choice for anything transactional/templated/branded (HTML emails, attachments, tracking, high volume), since SNS email subscriptions are meant for simple alerting, not production email delivery.

**Q: Do I need SQS in between SNS and my worker at all?**
A: Not strictly — SNS can push directly to Lambda or HTTP(S) endpoints. But SQS adds a durable buffer: if your consumer is down, SNS messages without a queue behind them are lost forever (SNS is not persistent), whereas SQS retains them for up to 14 days.

**Q: My SES production access request got denied — now what?**
A: Re-submit with a more detailed use case: how you build your mailing list (opt-in only), how you handle bounces/complaints/unsubscribes, and a link to a live website showing the sender identity/domain in context. Vague or missing detail is the most common rejection reason.

**Q: How do I simulate bounces/complaints in the SES sandbox without spamming real people?**
A: Use the built-in [SES mailbox simulator addresses](https://docs.aws.amazon.com/ses/latest/dg/send-an-email-from-console.html) (e.g. `bounce@simulator.amazonses.com`, `complaint@simulator.amazonses.com`, `success@simulator.amazonses.com`) — safe to use even in sandbox mode.
