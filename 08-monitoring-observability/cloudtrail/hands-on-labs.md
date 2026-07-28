# CloudTrail — Hands-On Labs (Scratch → Forensics)

Eight progressive labs building a production-grade audit pipeline, then using it like a security engineer would.

> **Cost note:** management events on your first trail are free; S3/CW Logs storage is pennies at lab scale. Data events (Lab 5) and CloudTrail Lake (Lab 8) bill per event — keep them scoped and clean up.

---

## Lab Index

| # | Lab | Skills proven |
|---|---|---|
| 1 | Explore Event History + `lookup-events` | Reading audit events, CLI filtering |
| 2 | First multi-region trail → S3 | Trail creation, bucket policy, delivery |
| 3 | Integrity validation + tamper detection | `validate-logs`, digest files |
| 4 | Real-time security alerts (CIS pattern) | CW Logs stream, metric filters, alarms |
| 5 | S3 data events on a sensitive bucket | Advanced event selectors |
| 6 | Insights events (anomaly detection) | ML-based burst detection |
| 7 | Athena forensics investigation | SQL over audit logs, incident workflow |
| 8 | CloudTrail Lake | Managed queryable audit store |

---

## Lab 1 — Read the Black Box: Event History

**Goal:** get fluent at reading CloudTrail events before building anything. Zero setup — management events are already recorded.

### Steps

```bash
# 1. Generate some auditable activity
aws s3 mb s3://lab1-trailtest-$RANDOM
aws iam create-user --user-name lab1-tempuser

# 2. Wait ~5-15 minutes (Event History has ingestion delay), then find your actions
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateUser \
  --max-results 5

aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateBucket \
  --max-results 5
```

**3. Dissect one event.** Pull out the five forensic fields — WHO (`userIdentity.arn`), WHAT (`eventName`), WHEN (`eventTime`), WHERE FROM (`sourceIPAddress`), and RESULT (`errorCode` present only on failure). Compare a call made via console vs CLI — note the different `userAgent`.

**4. See a denied call** (permission probing looks exactly like this):
```bash
# Run as a role/user WITHOUT iam:DeleteRole permission, or just note errorCode on any denied call
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ConsoleLogin
```

### Cleanup
```bash
aws iam delete-user --user-name lab1-tempuser
```

---

## Lab 2 — Your First Production Trail

**Goal:** permanent, multi-region audit archive in S3 — the foundation every later lab builds on.

### Steps

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET=cloudtrail-audit-$ACCOUNT_ID

# 1. Audit bucket: private, versioned
aws s3api create-bucket --bucket $BUCKET \
  --create-bucket-configuration LocationConstraint=ap-south-1
aws s3api put-public-access-block --bucket $BUCKET \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-versioning --bucket $BUCKET \
  --versioning-configuration Status=Enabled

# 2. Bucket policy (use the JSON from cheatsheet §8, substituting bucket/account/trail)
aws s3api put-bucket-policy --bucket $BUCKET --policy file://cloudtrail-bucket-policy.json

# 3. Trail: multi-region + global events + validation
aws cloudtrail create-trail \
  --name lab-security-trail \
  --s3-bucket-name $BUCKET \
  --is-multi-region-trail \
  --include-global-service-events \
  --enable-log-file-validation

# 4. START it — create-trail alone logs nothing!
aws cloudtrail start-logging --name lab-security-trail
aws cloudtrail get-trail-status --name lab-security-trail   # IsLogging: true
```

### Verify (after ~10–15 min)

```bash
aws s3 ls s3://$BUCKET/AWSLogs/$ACCOUNT_ID/CloudTrail/ --recursive | head

# Download and read one delivery
aws s3 cp s3://$BUCKET/AWSLogs/.../somefile.json.gz .
gunzip -c somefile.json.gz | python3 -m json.tool | head -50
```

**Prove multi-region works:** create a resource in another region (`aws s3 mb s3://lab2-testeast-$RANDOM --region us-east-1`) and find its events under the `us-east-1/` prefix of the *same* bucket.

---

## Lab 3 — Tamper Detection: Integrity Validation

**Goal:** verify the cryptographic digest chain, then break it deliberately and watch validation fail — the exact check auditors ask about.

### Steps

```bash
# 1. Let the trail run 1+ hour so digest files exist, then validate
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:ap-south-1:$ACCOUNT_ID:trail/lab-security-trail \
  --start-time $(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --verbose
# Expected: "... log files were valid" / nothing modified, deleted, or skipped
```

**2. Play the attacker.** Download one log file, delete a line, re-upload it over itself:
```bash
aws s3 cp s3://$BUCKET/AWSLogs/.../log.json.gz .
gunzip log.json.gz && sed -i '2d' log.json && gzip log.json
aws s3 cp log.json.gz s3://$BUCKET/AWSLogs/.../log.json.gz
```

**3. Re-run validation:**
```bash
aws cloudtrail validate-logs --trail-arn <arn> \
  --start-time $(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%SZ) --verbose
# Expected: the tampered file reported as INVALID (hash mismatch)
```

**Reflection:** this is why production adds **KMS encryption** + **S3 Object Lock (Compliance mode)** on top — validation *detects* tampering; Object Lock *prevents* it, even by root. (Optional stretch: recreate the bucket with Object Lock and attempt a delete.)

---

## Lab 4 — Real-Time Security Alerts (CIS Pattern)

**Goal:** the flagship pipeline — `CloudTrail → CloudWatch Logs → metric filter → alarm → SNS`. Alert within minutes on root usage and CloudTrail tampering.

### Steps

```bash
# 1. Log group + retention
aws logs create-log-group --log-group-name CloudTrail/SecurityEvents
aws logs put-retention-policy --log-group-name CloudTrail/SecurityEvents --retention-in-days 30

# 2. IAM role CloudTrail assumes (full JSON in cheatsheet §6, steps 2)
#    ... create CloudTrail_CWLogs_Role with CreateLogStream/PutLogEvents ...

# 3. Wire the trail
aws cloudtrail update-trail \
  --name lab-security-trail \
  --cloud-watch-logs-log-group-arn arn:aws:logs:ap-south-1:$ACCOUNT_ID:log-group:CloudTrail/SecurityEvents:* \
  --cloud-watch-logs-role-arn arn:aws:iam::$ACCOUNT_ID:role/CloudTrail_CWLogs_Role

# 4. SNS topic for the security team
aws sns create-topic --name security-alerts
aws sns subscribe --topic-arn arn:aws:sns:ap-south-1:$ACCOUNT_ID:security-alerts \
  --protocol email --notification-endpoint you@example.com   # confirm the email!

# 5. Metric filter + alarm: CloudTrail tampering
aws logs put-metric-filter \
  --log-group-name CloudTrail/SecurityEvents \
  --filter-name TrailTampering \
  --filter-pattern '{ ($.eventName = StopLogging) || ($.eventName = DeleteTrail) || ($.eventName = UpdateTrail) }' \
  --metric-transformations metricName=TrailTamperCount,metricNamespace=CloudTrail/Security,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name "SEC-TrailTampering" \
  --namespace CloudTrail/Security --metric-name TrailTamperCount \
  --statistic Sum --period 300 --evaluation-periods 1 \
  --threshold 0 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-south-1:$ACCOUNT_ID:security-alerts

# 6. Add the root-usage and unauthorized-calls filters from cheatsheet §6
```

### Trigger it (safely)

```bash
# StopLogging IS the tamper event — stop, get alerted, restart
aws cloudtrail stop-logging --name lab-security-trail
sleep 60
aws cloudtrail start-logging --name lab-security-trail
```

### Verify
Within ~5–10 minutes: alarm `SEC-TrailTampering` → `ALARM`, email arrives. You just caught the first move of nearly every real intrusion.

---

## Lab 5 — S3 Data Events on a Sensitive Bucket

**Goal:** object-level auditing (`GetObject`/`PutObject`/`DeleteObject`) — the thing management events do NOT cover — scoped tightly with advanced event selectors to control cost.

### Steps

```bash
# 1. A "sensitive" bucket + object
aws s3 mb s3://lab5-pii-$ACCOUNT_ID
echo "ssn,name" > sample.csv && aws s3 cp sample.csv s3://lab5-pii-$ACCOUNT_ID/pii/sample.csv

# 2. Advanced event selectors: data events ONLY for this bucket (plus mgmt events)
aws cloudtrail put-event-selectors \
  --trail-name lab-security-trail \
  --advanced-event-selectors '[
    {"Name":"mgmt","FieldSelectors":[{"Field":"eventCategory","Equals":["Management"]}]},
    {"Name":"pii-bucket-objects","FieldSelectors":[
      {"Field":"eventCategory","Equals":["Data"]},
      {"Field":"resources.type","Equals":["AWS::S3::Object"]},
      {"Field":"resources.ARN","StartsWith":["arn:aws:s3:::lab5-pii-'"$ACCOUNT_ID"'/"]}]}]'

# 3. Generate object-level activity
aws s3 cp s3://lab5-pii-$ACCOUNT_ID/pii/sample.csv /tmp/x.csv     # GetObject
aws s3 rm s3://lab5-pii-$ACCOUNT_ID/pii/sample.csv                # DeleteObject
```

### Verify
After ~10 min, search the CW Logs stream (or S3 files) for `"eventName": "GetObject"` with your bucket ARN. Note these do **not** appear in `lookup-events` Event History — data events only land in the trail outputs.

**Reflection:** this is where *Data Events Aggregation* matters at enterprise scale — 5-minute summaries instead of drowning in per-object lines.

### Cleanup
Remove the data selector (revert to management-only) and delete the bucket.

---

## Lab 6 — Insights Events: Catching the Burst

**Goal:** enable ML-based anomaly detection for API call-rate/error-rate bursts (the crypto-mining / privilege-escalation tell).

### Steps

```bash
aws cloudtrail put-insight-selectors \
  --trail-name lab-security-trail \
  --insight-selectors '[{"InsightType":"ApiCallRateInsight"},{"InsightType":"ApiErrorRateInsight"}]'

aws cloudtrail get-insight-selectors --trail-name lab-security-trail
```

**Simulate a burst** (Insights needs a baseline first — it learns your normal for ~ days; in a fresh lab account results may take time):
```bash
for i in {1..200}; do aws s3api list-buckets > /dev/null; done
```

### Verify
```bash
aws cloudtrail lookup-events --event-category insight
```
Insights events land under a separate S3 prefix (`.../CloudTrail-Insight/`) and describe the anomaly: which API, expected rate vs observed rate, start/end.

**Honest note:** in a quiet lab account, Insights may not fire — it flags deviation *from your baseline*, and a new account barely has one. Understand the mechanism; expect real value in busy production accounts.

---

## Lab 7 — Athena Forensics: The Incident Investigation

**Goal:** the capstone investigation. Scenario: *"Someone opened SSH to the world on a prod security group last night. Who, when, from where — and what else did they touch?"*

### Steps

**1. Create the incident evidence:**
```bash
SG=$(aws ec2 create-security-group --group-name lab7-victim --description "lab" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG \
  --protocol tcp --port 22 --cidr 0.0.0.0/0
```

**2. Set up Athena** (wait ~15 min for delivery): run the `CREATE EXTERNAL TABLE` with partition projection from [cheatsheet §9](commands-cheatsheet.md#9-athena-setup--forensic-queries), pointing at your Lab 2 bucket. Set an Athena query-results bucket when prompted.

**3. Run the investigation, question by question:**
```sql
-- Q1: WHO opened the security group?
SELECT eventTime, userIdentity.arn, sourceIPAddress, requestParameters
FROM cloudtrail_logs_prod
WHERE eventName = 'AuthorizeSecurityGroupIngress'
  AND requestParameters LIKE '%0.0.0.0/0%'
ORDER BY eventTime DESC LIMIT 10;

-- Q2: What ELSE did that identity do around that time? (pivot on the ARN)
SELECT eventTime, eventName, eventSource, sourceIPAddress, errorCode
FROM cloudtrail_logs_prod
WHERE userIdentity.arn = '<arn-from-Q1>'
ORDER BY eventTime DESC LIMIT 50;

-- Q3: Any other activity from the same source IP?
SELECT eventTime, eventName, userIdentity.arn
FROM cloudtrail_logs_prod
WHERE sourceIPAddress = '<ip-from-Q1>'
ORDER BY eventTime DESC;

-- Q4: Were they probing before acting? (denied calls)
SELECT eventTime, eventName, errorCode
FROM cloudtrail_logs_prod
WHERE userIdentity.arn = '<arn-from-Q1>' AND errorCode IS NOT NULL;
```

**4. Write it up** — timeline of who/what/when/where, exactly like a real post-incident report. (This four-query pivot — action → identity → IP → denials — is the universal CloudTrail investigation pattern.)

### Cleanup
```bash
aws ec2 delete-security-group --group-id $SG
```

---

## Lab 8 — CloudTrail Lake: The Managed Alternative

**Goal:** same forensic power, zero Athena/Glue plumbing.

### Steps

```bash
# 1. Event data store — start with minimal retention for the lab
aws cloudtrail create-event-data-store \
  --name lab-audit-lake \
  --retention-period 7 \
  --multi-region-enabled \
  --advanced-event-selectors '[{"Name":"mgmt","FieldSelectors":[{"Field":"eventCategory","Equals":["Management"]}]}]'

EDS_ID=$(aws cloudtrail list-event-data-stores \
  --query "EventDataStores[?Name=='lab-audit-lake'].EventDataStoreArn" --output text | awk -F/ '{print $NF}')

# 2. Generate some activity, wait ~15 min, then query with SQL directly:
QID=$(aws cloudtrail start-query --query-statement \
  "SELECT eventTime, userIdentity.arn, eventName, sourceIPAddress
   FROM $EDS_ID
   WHERE eventTime > '2026-07-28 00:00:00'
   ORDER BY eventTime DESC LIMIT 20" \
  --query 'QueryId' --output text)

aws cloudtrail get-query-results --query-id $QID
```

### Verify & compare
Re-run Lab 7's Q1 as a Lake query. Reflect on the trade-off: Lake = instant SQL, 10-year retention option, org/multi-source ingestion, but per-event ingestion pricing; S3+Athena = cheapest at massive scale, more plumbing.

### Cleanup
```bash
aws cloudtrail delete-event-data-store --event-data-store $EDS_ID   # (disable termination protection first if set)
```

---

## Full Cleanup Checklist

```bash
aws cloudtrail stop-logging --name lab-security-trail
aws cloudtrail delete-trail --name lab-security-trail
aws logs delete-log-group --log-group-name CloudTrail/SecurityEvents
aws cloudwatch delete-alarms --alarm-names SEC-TrailTampering SEC-RootAccountUsed
aws sns delete-topic --topic-arn arn:aws:sns:ap-south-1:$ACCOUNT_ID:security-alerts
aws s3 rb s3://$BUCKET --force        # only if you don't want to keep the archive
```

## Where to Go Next

- Convert Lab 2's trail into an **Organization trail** delivering to a dedicated log-archive account.
- Add **EventBridge auto-remediation**: on `AuthorizeSecurityGroupIngress` with `0.0.0.0/0` → Lambda revokes the rule automatically.
- Feed the trail into **Amazon GuardDuty / Security Hub** and compare their findings with your hand-built alarms.
