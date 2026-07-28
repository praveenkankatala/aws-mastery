# CloudWatch — Hands-On Labs (Scratch → Production)

Ten progressive labs. Each is self-contained, states its goal, prerequisites, steps, verification, and cleanup. Estimated total: a weekend well spent.

> **Cost note:** most labs fit AWS Free Tier. Labs 6 (Container Insights) and 8 (Synthetics) incur small charges — clean up promptly.

---

## Lab Index

| # | Lab | Skills proven |
|---|---|---|
| 1 | First alarm: CPU → SNS email | Metrics, alarms, SNS wiring |
| 2 | CloudWatch Agent: memory & disk + app logs | Agent install, IAM, log shipping |
| 3 | Metric filter → alarm on log errors | Logs-to-metrics pipeline |
| 4 | Custom metrics via CLI & Python SDK | `PutMetricData`, dimensions, high-res |
| 5 | EMF from Lambda (the modern way) | Serverless metrics, cardinality control |
| 6 | EKS Container Insights + control-plane logs | Kubernetes observability |
| 7 | Golden Trio dashboard | Dashboard-as-JSON, metric math |
| 8 | Synthetics canary heartbeat | Outside-in monitoring |
| 9 | Anomaly-detection & composite alarms | Advanced alerting, de-noising |
| 10 | X-Ray trace-to-log correlation | Distributed tracing, MTTR reduction |

---

## Lab 1 — Your First Alarm: High CPU → Email

**Goal:** get paged when an EC2 instance's CPU exceeds 70%.

**Prereqs:** one running EC2 instance (t2/t3.micro is fine).

### Steps

```bash
# 1. Create an SNS topic and subscribe your email
aws sns create-topic --name ops-alerts
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-south-1:<ACCOUNT_ID>:ops-alerts \
  --protocol email --notification-endpoint you@example.com
# -> Check your inbox and CONFIRM the subscription (alarm emails silently fail otherwise)

# 2. Create the alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "Lab1-HighCPU" \
  --namespace "AWS/EC2" --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<INSTANCE_ID> \
  --statistic Average --period 60 --evaluation-periods 3 \
  --threshold 70 --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:<ACCOUNT_ID>:ops-alerts

# 3. Generate load on the instance (SSH in)
sudo yum install -y stress   # or: sudo apt install stress
stress --cpu 2 --timeout 300
```

### Verify
- Console → CloudWatch → Alarms: state moves `OK → ALARM` within ~3–4 minutes.
- You receive the SNS email.
- Instant wiring test without load: `aws cloudwatch set-alarm-state --alarm-name Lab1-HighCPU --state-value ALARM --state-reason "test"`

### Cleanup
```bash
aws cloudwatch delete-alarms --alarm-names Lab1-HighCPU
```

---

## Lab 2 — Unified Agent: Memory, Disk % and Application Logs

**Goal:** collect the metrics EC2 *doesn't* give you by default (memory, disk %) and ship an application log file to CloudWatch Logs.

### Steps

**1. IAM:** attach the managed policy `CloudWatchAgentServerPolicy` to the instance's IAM role (create the role if the instance has none — agent auth fails without it).

**2. Install the agent:**
```bash
sudo yum install -y amazon-cloudwatch-agent
```

**3. Write the config** at `/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`:
```json
{
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": { "InstanceId": "${aws:InstanceId}" },
    "metrics_collected": {
      "mem":  { "measurement": ["mem_used_percent"], "metrics_collection_interval": 60 },
      "disk": { "measurement": ["used_percent"], "resources": ["/"], "metrics_collection_interval": 60 }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          { "file_path": "/var/log/myapp/app.log",
            "log_group_name": "/lab2/myapp",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 7 }
        ]
      }
    }
  }
}
```

**4. Start and generate data:**
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s

sudo mkdir -p /var/log/myapp
echo "$(date) INFO application started" | sudo tee -a /var/log/myapp/app.log
echo "$(date) ERROR db connection timeout" | sudo tee -a /var/log/myapp/app.log
```

### Verify
```bash
aws cloudwatch list-metrics --namespace CWAgent          # mem_used_percent, disk used_percent
aws logs tail /lab2/myapp --follow                       # your log lines
```

---

## Lab 3 — Metric Filter: Alarm on Log Errors

**Goal:** turn `ERROR` lines in the Lab 2 log group into a metric, then alarm on it. This is the classic logs→metrics→action pipeline.

### Steps

```bash
# 1. Metric filter
aws logs put-metric-filter \
  --log-group-name /lab2/myapp \
  --filter-name ErrorCount \
  --filter-pattern "ERROR" \
  --metric-transformations \
    metricName=AppErrors,metricNamespace=Lab3/Logs,metricValue=1,defaultValue=0

# 2. Alarm: any error in a 1-minute window
aws cloudwatch put-metric-alarm \
  --alarm-name "Lab3-AppErrors" \
  --namespace "Lab3/Logs" --metric-name AppErrors \
  --statistic Sum --period 60 --evaluation-periods 1 \
  --threshold 0 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-south-1:<ACCOUNT_ID>:ops-alerts

# 3. Trigger it
echo "$(date) ERROR payment gateway 503" | sudo tee -a /var/log/myapp/app.log
```

> Note `--treat-missing-data notBreaching`: without it, quiet periods (no logs at all) leave the alarm in `INSUFFICIENT_DATA`.

### Verify
Alarm fires within ~2 minutes; email arrives referencing `Lab3/Logs AppErrors`.

---

## Lab 4 — Custom Metrics via CLI & Python SDK

**Goal:** publish business metrics with dimensions; observe cardinality behavior.

### Steps

**CLI:**
```bash
for i in 1 2 3 4 5; do
  aws cloudwatch put-metric-data \
    --namespace "Lab4/Shop" \
    --metric-name CheckoutValue \
    --dimensions Environment=Prod,Tier=Premium \
    --value $((RANDOM % 500)) --unit None
  sleep 5
done
```

**Python (boto3) — including a high-resolution metric:**
```python
import boto3, random, time
cw = boto3.client("cloudwatch")

for _ in range(10):
    cw.put_metric_data(
        Namespace="Lab4/Shop",
        MetricData=[
            {   # standard resolution, low-cardinality dimensions
                "MetricName": "CheckoutValue",
                "Dimensions": [
                    {"Name": "Environment", "Value": "Prod"},
                    {"Name": "Tier", "Value": random.choice(["Free", "Standard", "Premium"])},
                ],
                "Value": random.uniform(10, 500),
                "Unit": "None",
            },
            {   # high-resolution (1-second) metric
                "MetricName": "PaymentLatency",
                "Value": random.uniform(50, 900),
                "Unit": "Milliseconds",
                "StorageResolution": 1,
            },
        ],
    )
    time.sleep(2)
```

### Verify & reflect
- Console → Metrics → `Lab4/Shop`. Count the metric streams: with 3 Tier values you have exactly **3** `CheckoutValue` streams. Now imagine `CustomerId` as a dimension with 50k users — that's 50k × $0.30 = **$15,000/month**. This is cardinality made tangible.

---

## Lab 5 — EMF from Lambda (the Modern Standard)

**Goal:** emit metrics from Lambda with **zero API latency** using Embedded Metric Format, keeping unique IDs queryable for free.

### Steps

**1. Create the function** (Python 3.12, default execution role is enough — EMF only needs log permissions):
```python
import json, time

def handler(event, context):
    emf = {
        "_aws": {
            "Timestamp": int(time.time() * 1000),
            "CloudWatchMetrics": [{
                "Namespace": "Lab5/Payments",
                "Dimensions": [["Environment"]],
                "Metrics": [{"Name": "TransactionProcessingTime", "Unit": "Milliseconds"}]
            }]
        },
        "Environment": "Production",
        "TransactionProcessingTime": 245,
        "OrderId": "ord-99831",       # NOT a dimension -> free, queryable via Logs Insights
        "CustomerId": "cust-552"
    }
    print(json.dumps(emf))            # that's it — print to stdout
    return {"statusCode": 200}
```

**2. Invoke a few times:**
```bash
for i in {1..5}; do
  aws lambda invoke --function-name lab5-emf --payload '{}' /dev/null
done
```

### Verify
- Metrics → `Lab5/Payments` → `TransactionProcessingTime` appears within ~1 minute, no `PutMetricData` call made.
- Logs Insights on `/aws/lambda/lab5-emf`:
```sql
fields @timestamp, TransactionProcessingTime, OrderId, CustomerId
| filter CustomerId = "cust-552"
```
You just queried a per-customer transaction **without paying for a per-customer metric**.

---

## Lab 6 — EKS Container Insights + Control-Plane Logs

**Goal:** full-stack Kubernetes observability. **Prereqs:** an EKS cluster (eksctl or console) with a node group.

### Steps

```bash
# 1. Control-plane logs
aws eks update-cluster-config --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","controllerManager","scheduler"],"enabled":true}]}'

# 2. Cap their retention immediately (cost control)
aws logs put-retention-policy --log-group-name /aws/eks/my-cluster/cluster --retention-in-days 30

# 3. IAM: attach CloudWatchAgentServerPolicy to the NODE role, then install the add-on
aws eks create-addon --cluster-name my-cluster --addon-name amazon-cloudwatch-observability

# 4. Verify the DaemonSets
kubectl get pods -n amazon-cloudwatch    # cloudwatch-agent-* and fluent-bit-* Running

# 5. Deploy a victim workload with a deliberately tiny memory limit
kubectl create deployment stress --image=polinux/stress \
  -- stress --vm 1 --vm-bytes 300M --vm-hang 0
kubectl set resources deployment stress --limits=memory=128Mi
```

### Verify
- Console → CloudWatch → Container Insights → your cluster: node/pod/container drill-down.
- Watch `pod_number_of_container_restarts` climb and `container_memory_oom_killed` fire as the kernel kills the over-limit container — the exact signals from the README's alert table.
- (Optional) add the Fluent Bit namespace-exclusion filter from the README §7.4 and confirm `kube-system` logs stop ingesting.

### Cleanup
Delete the deployment and, if the cluster was lab-only, the cluster — EKS bills hourly.

---

## Lab 7 — The Golden Trio Dashboard

**Goal:** one dashboard, three panels: Traffic, Errors, Latency-breakdown — the pattern that instantly answers "is it my code or the gateway?"

**Prereqs:** any API Gateway + Lambda pair (Lab 5's function behind a HTTP API works).

### Steps

Use the full `put-dashboard` JSON from the [cheatsheet §6](commands-cheatsheet.md#6-dashboards), substituting your API name and function name. Then drive traffic:

```bash
for i in {1..50}; do curl -s https://<api-id>.execute-api.ap-south-1.amazonaws.com/ > /dev/null; done
```

### Verify & read the panels like an SRE
- `IntegrationLatency` and `Duration` rising **together** → Lambda code is the bottleneck.
- Only `Latency` rising → API Gateway config/authorizers are the drag.
- Add a 4th widget: metric-math error rate `(errors/invocations)*100`.

---

## Lab 8 — Synthetics Canary (Outside-In Monitoring)

**Goal:** probe your endpoint every 5 minutes from AWS's side — catch outages before users do.

### Steps (console-first; canary scripting is visual)

1. CloudWatch → Synthetics → **Create canary** → blueprint **Heartbeat monitoring**.
2. URL: your API/website endpoint. Schedule: `rate(5 minutes)`.
3. Let it create the IAM role + S3 artifacts bucket.
4. After 2–3 runs, stop your endpoint (or point the canary at a fake URL) and watch it fail.
5. Create an alarm on the canary's `SuccessPercent` metric (`< 100` for 1 period → SNS).

### Verify
Failed runs store screenshots + HAR files in S3 — inspect exactly what the "user" saw.

### Cleanup
Stop + delete the canary and its artifacts bucket (runs bill per execution).

---

## Lab 9 — Anomaly Detection & Composite Alarms

**Goal:** smarter alerting — ML-based bands for traffic-shaped metrics, and composite logic to stop 3 AM false pages.

### Steps

```bash
# 1. Anomaly detector + alarm on API Gateway request count
aws cloudwatch put-anomaly-detector \
  --single-metric-anomaly-detector '{"Namespace":"AWS/ApiGateway","MetricName":"Count","Dimensions":[{"Name":"ApiName","Value":"<your-api>"}],"Stat":"Sum"}'

aws cloudwatch put-metric-alarm \
  --alarm-name "Lab9-AnomalousTraffic" \
  --comparison-operator GreaterThanUpperThreshold \
  --evaluation-periods 3 \
  --metrics '[
    {"Id":"m1","MetricStat":{"Metric":{"Namespace":"AWS/ApiGateway","MetricName":"Count","Dimensions":[{"Name":"ApiName","Value":"<your-api>"}]},"Period":300,"Stat":"Sum"},"ReturnData":true},
    {"Id":"ad1","Expression":"ANOMALY_DETECTION_BAND(m1, 2)","ReturnData":true}]' \
  --threshold-metric-id ad1 \
  --alarm-actions arn:aws:sns:ap-south-1:<ACCOUNT_ID>:ops-alerts

# 2. Composite: page ONLY when traffic anomaly AND errors coincide
aws cloudwatch put-composite-alarm \
  --alarm-name "Lab9-RealIncident" \
  --alarm-rule 'ALARM("Lab9-AnomalousTraffic") AND ALARM("Lab3-AppErrors")' \
  --alarm-actions arn:aws:sns:ap-south-1:<ACCOUNT_ID>:ops-alerts
```

### Verify
- The anomaly model needs some history — after a few hours the console shows the gray "expected band" around the metric.
- Force-test the composite: `set-alarm-state` both children to `ALARM` and watch the parent fire once, not twice.

---

## Lab 10 — X-Ray Trace-to-Log Correlation (Capstone)

**Goal:** click a slow trace → see the exact log lines from that request. The MTTR-killer.

### Steps

**1. Enable Active Tracing on your Lambda:**
```bash
aws lambda update-function-configuration \
  --function-name lab5-emf --tracing-config Mode=Active
```
(Ensure the execution role has `AWSXRayDaemonWriteAccess`.)

**2. Use Powertools so every log line carries the trace ID:**
```python
from aws_lambda_powertools import Logger, Tracer
import time, random

logger = Logger(service="orders")      # auto-injects xray_trace_id into JSON logs
tracer = Tracer(service="orders")

@tracer.capture_lambda_handler
@logger.inject_lambda_context
def handler(event, context):
    logger.info("processing order")
    delay = random.uniform(0.1, 1.5)
    time.sleep(delay)                   # simulated slow dependency
    if delay > 1.2:
        logger.error("upstream payment API slow", extra={"delay_s": delay})
    return {"statusCode": 200}
```
(Add the Powertools Lambda layer for your region, or bundle the pip package.)

**3. Enable API Gateway tracing** (if fronted by API GW): stage → *X-Ray Tracing: enabled* — now the trace spans GW → Lambda.

**4. Drive traffic:**
```bash
for i in {1..30}; do aws lambda invoke --function-name lab5-emf --payload '{}' /dev/null; done
```

### Verify
1. CloudWatch → X-Ray traces → filter `responsetime > 1`.
2. Open a slow trace → segment timeline shows where the time went.
3. **Logs tab** → the exact log rows for that trace ID appear, including your `"upstream payment API slow"` error.
4. Reverse direction — from Logs Insights:
```sql
fields @timestamp, @message
| filter xray_trace_id = "<trace-id-from-console>"
```

**Checklist from the README §10.5 applies here:** NTP-synced clocks, sampling aligned with log verbosity.

---

## Where to Go Next

- Re-run labs 5 + 10 on **ECS Fargate** with the ADOT collector.
- Add **Application Signals SLOs** on top of Lab 10 ("99% of invocations < 500 ms / 7 days").
- Wire Lab 3's alarm to a **Lambda auto-remediation** function instead of email.
