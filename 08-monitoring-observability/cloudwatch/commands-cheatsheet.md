# CloudWatch — CLI Commands Cheatsheet

All commands use AWS CLI v2. Replace placeholder values (`<...>`) and regions as needed.
Default region can be set once: `aws configure set region ap-south-1`

---

## Table of Contents

1. [Metrics — Publish & Read](#1-metrics--publish--read)
2. [Alarms](#2-alarms)
3. [Logs — Groups, Streams, Events](#3-logs--groups-streams-events)
4. [Logs Insights Queries](#4-logs-insights-queries)
5. [Metric Filters & Subscription Filters](#5-metric-filters--subscription-filters)
6. [Dashboards](#6-dashboards)
7. [Unified CloudWatch Agent](#7-unified-cloudwatch-agent)
8. [EKS / Container Insights](#8-eks--container-insights)
9. [Synthetics, Anomaly Detection & Metric Streams](#9-synthetics-anomaly-detection--metric-streams)
10. [X-Ray](#10-x-ray)
11. [EventBridge (CloudWatch Events)](#11-eventbridge)

---

## 1. Metrics — Publish & Read

```bash
# List all namespaces/metrics available
aws cloudwatch list-metrics

# List metrics for one namespace
aws cloudwatch list-metrics --namespace "AWS/EC2"

# List a specific metric across dimensions
aws cloudwatch list-metrics --namespace "AWS/EC2" --metric-name CPUUtilization

# Publish a simple custom metric
aws cloudwatch put-metric-data \
  --namespace "ECommerceApp/Production" \
  --metric-name SuccessfulLogins \
  --value 1 \
  --unit Count

# Publish a custom metric WITH dimensions
aws cloudwatch put-metric-data \
  --namespace "ECommerceApp/Production" \
  --metric-name ApiLatency \
  --dimensions Environment=Prod,Tier=Premium \
  --value 245 \
  --unit Milliseconds

# Publish a HIGH-RESOLUTION (1-second) metric
aws cloudwatch put-metric-data \
  --namespace "Trading/Realtime" \
  --metric-name OrderProcessingTime \
  --value 12.5 \
  --unit Milliseconds \
  --storage-resolution 1

# Publish pre-aggregated statistics (min/max/sum/count in one call)
aws cloudwatch put-metric-data \
  --namespace "ECommerceApp/Production" \
  --metric-name RequestLatency \
  --statistic-values SampleCount=100,Sum=12000,Minimum=50,Maximum=900 \
  --unit Milliseconds

# Read metric statistics (classic API)
aws cloudwatch get-metric-statistics \
  --namespace "AWS/EC2" \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --start-time 2026-07-28T00:00:00Z \
  --end-time 2026-07-28T23:59:59Z \
  --period 300 \
  --statistics Average Maximum

# Read metrics with metric math (modern API — supports expressions)
aws cloudwatch get-metric-data \
  --start-time 2026-07-28T00:00:00Z \
  --end-time 2026-07-28T12:00:00Z \
  --metric-data-queries '[
    {"Id":"errors","MetricStat":{"Metric":{"Namespace":"AWS/Lambda","MetricName":"Errors","Dimensions":[{"Name":"FunctionName","Value":"my-fn"}]},"Period":300,"Stat":"Sum"}},
    {"Id":"invocations","MetricStat":{"Metric":{"Namespace":"AWS/Lambda","MetricName":"Invocations","Dimensions":[{"Name":"FunctionName","Value":"my-fn"}]},"Period":300,"Stat":"Sum"}},
    {"Id":"errorRate","Expression":"(errors/invocations)*100","Label":"Error Rate %"}
  ]'
```

---

## 2. Alarms

```bash
# Create a static-threshold alarm (CPU > 80% for 2 x 5-min periods)
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-WebServer" \
  --alarm-description "CPU above 80% for 10 minutes" \
  --namespace "AWS/EC2" \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:111122223333:ops-alerts \
  --ok-actions arn:aws:sns:ap-south-1:111122223333:ops-alerts

# Create an ANOMALY DETECTION alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "AnomalousRequestCount" \
  --comparison-operator GreaterThanUpperThreshold \
  --evaluation-periods 3 \
  --metrics '[
    {"Id":"m1","MetricStat":{"Metric":{"Namespace":"AWS/ApiGateway","MetricName":"Count","Dimensions":[{"Name":"ApiName","Value":"orders-api"}]},"Period":300,"Stat":"Sum"},"ReturnData":true},
    {"Id":"ad1","Expression":"ANOMALY_DETECTION_BAND(m1, 2)","Label":"Expected band","ReturnData":true}
  ]' \
  --threshold-metric-id ad1 \
  --alarm-actions arn:aws:sns:ap-south-1:111122223333:ops-alerts

# Create a COMPOSITE alarm (fire only when BOTH children are in ALARM)
aws cloudwatch put-composite-alarm \
  --alarm-name "Critical-App-Degradation" \
  --alarm-rule "ALARM(\"HighCPU-WebServer\") AND ALARM(\"HighLatency-API\")" \
  --alarm-actions arn:aws:sns:ap-south-1:111122223333:pagerduty

# EC2 auto-recover on hardware failure
aws cloudwatch put-metric-alarm \
  --alarm-name "AutoRecover-i-0123" \
  --namespace "AWS/EC2" \
  --metric-name StatusCheckFailed_System \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --statistic Maximum --period 60 --evaluation-periods 2 \
  --threshold 0 --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:automate:ap-south-1:ec2:recover

# List / inspect / test / clean up
aws cloudwatch describe-alarms
aws cloudwatch describe-alarms --state-value ALARM
aws cloudwatch describe-alarm-history --alarm-name "HighCPU-WebServer"
aws cloudwatch set-alarm-state --alarm-name "HighCPU-WebServer" \
  --state-value ALARM --state-reason "Testing SNS wiring"       # force-test an alarm
aws cloudwatch disable-alarm-actions --alarm-names "HighCPU-WebServer"
aws cloudwatch enable-alarm-actions  --alarm-names "HighCPU-WebServer"
aws cloudwatch delete-alarms --alarm-names "HighCPU-WebServer"
```

---

## 3. Logs — Groups, Streams, Events

```bash
# Create a log group + set retention (do BOTH, always)
aws logs create-log-group --log-group-name /myapp/production
aws logs put-retention-policy --log-group-name /myapp/production --retention-in-days 30

# Create with Infrequent Access class (cheap ingestion)
aws logs create-log-group --log-group-name /myapp/vpc-flow-logs --log-group-class INFREQUENT_ACCESS

# List groups / streams
aws logs describe-log-groups
aws logs describe-log-groups --log-group-name-prefix /myapp
aws logs describe-log-streams --log-group-name /myapp/production --order-by LastEventTime --descending

# Write a test event manually
aws logs create-log-stream --log-group-name /myapp/production --log-stream-name manual-test
aws logs put-log-events \
  --log-group-name /myapp/production \
  --log-stream-name manual-test \
  --log-events timestamp=$(date +%s000),message="ERROR test event from CLI"

# Read events
aws logs get-log-events --log-group-name /myapp/production --log-stream-name manual-test

# Tail logs LIVE (extremely useful)
aws logs tail /myapp/production --follow
aws logs tail /myapp/production --since 1h --filter-pattern "ERROR"

# Filter events across ALL streams in a group
aws logs filter-log-events \
  --log-group-name /myapp/production \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s000)

# Export a log group to S3 (batch archive)
aws logs create-export-task \
  --log-group-name /myapp/production \
  --from $(date -d '30 days ago' +%s000) \
  --to $(date +%s000) \
  --destination my-log-archive-bucket \
  --destination-prefix myapp-export

# Encrypt a log group with KMS
aws logs associate-kms-key --log-group-name /myapp/production \
  --kms-key-id arn:aws:kms:ap-south-1:111122223333:key/<key-id>

# Delete
aws logs delete-log-group --log-group-name /myapp/production
```

---

## 4. Logs Insights Queries

```bash
# Start a query
aws logs start-query \
  --log-group-name /myapp/production \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'

# Fetch results (use the queryId returned above)
aws logs get-query-results --query-id <query-id>
```

**Query language patterns worth memorizing:**

```sql
-- Count errors per 5-minute bucket
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as errors by bin(5m)

-- Top 10 slowest Lambda invocations (built-in @duration)
filter @type = "REPORT"
| stats max(@duration) as maxMs by @requestId
| sort maxMs desc | limit 10

-- Parse a field out of unstructured text
parse @message "TraceID: *" as TraceId
| display @timestamp, TraceId, @message

-- Find HTTP 404s in web logs
fields @timestamp, @message
| filter @message like /404/
| stats count(*) by bin(1h)

-- Query EMF metadata fields directly (e.g., per-customer lookup — the free-cardinality trick)
fields @timestamp, TransactionProcessingTime, OrderId, CustomerId
| filter CustomerId = "cust-552"
| sort @timestamp desc
```

---

## 5. Metric Filters & Subscription Filters

```bash
# Metric filter: count ERROR lines as a metric
aws logs put-metric-filter \
  --log-group-name /myapp/production \
  --filter-name ErrorCount \
  --filter-pattern "ERROR" \
  --metric-transformations \
    metricName=AppErrorCount,metricNamespace=MyApp/Logs,metricValue=1,defaultValue=0

# Metric filter on structured JSON logs
aws logs put-metric-filter \
  --log-group-name /myapp/production \
  --filter-name HighLatency \
  --filter-pattern '{ $.latencyMs > 1000 }' \
  --metric-transformations \
    metricName=SlowRequests,metricNamespace=MyApp/Logs,metricValue=1

# List / test / delete metric filters
aws logs describe-metric-filters --log-group-name /myapp/production
aws logs test-metric-filter --filter-pattern "ERROR" \
  --log-event-messages "INFO all good" "ERROR db timeout"
aws logs delete-metric-filter --log-group-name /myapp/production --filter-name ErrorCount

# Subscription filter: stream logs in real time to Lambda
aws logs put-subscription-filter \
  --log-group-name /myapp/production \
  --filter-name ToLambda \
  --filter-pattern "" \
  --destination-arn arn:aws:lambda:ap-south-1:111122223333:function:log-processor
```

---

## 6. Dashboards

```bash
# Create/update a dashboard
aws cloudwatch put-dashboard \
  --dashboard-name "Prod-API-Golden-Trio" \
  --dashboard-body '{
    "widgets": [
      {"type":"metric","x":0,"y":0,"width":8,"height":6,
       "properties":{"title":"Traffic","region":"ap-south-1",
         "metrics":[["AWS/ApiGateway","Count","ApiName","orders-api"],
                    ["AWS/Lambda","Invocations","FunctionName","orders-fn"]],
         "stat":"Sum","period":300}},
      {"type":"metric","x":8,"y":0,"width":8,"height":6,
       "properties":{"title":"Errors","region":"ap-south-1",
         "metrics":[["AWS/ApiGateway","5XXError","ApiName","orders-api"],
                    ["AWS/Lambda","Errors","FunctionName","orders-fn"],
                    ["AWS/Lambda","Throttles","FunctionName","orders-fn"]],
         "stat":"Sum","period":300}},
      {"type":"metric","x":16,"y":0,"width":8,"height":6,
       "properties":{"title":"Latency Breakdown","region":"ap-south-1",
         "metrics":[["AWS/ApiGateway","Latency","ApiName","orders-api"],
                    ["AWS/ApiGateway","IntegrationLatency","ApiName","orders-api"],
                    ["AWS/Lambda","Duration","FunctionName","orders-fn"]],
         "stat":"Average","period":300}}
    ]}'

# List / read / delete
aws cloudwatch list-dashboards
aws cloudwatch get-dashboard --dashboard-name "Prod-API-Golden-Trio"
aws cloudwatch delete-dashboards --dashboard-names "Prod-API-Golden-Trio"
```

---

## 7. Unified CloudWatch Agent

```bash
# Install on Amazon Linux / Ubuntu (EC2)
sudo yum install -y amazon-cloudwatch-agent          # Amazon Linux
sudo apt-get install -y amazon-cloudwatch-agent      # Ubuntu (or download .deb)

# Interactive config wizard (generates JSON config)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Start the agent with a local config file
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s

# Start with config stored in SSM Parameter Store (fleet-wide pattern)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c ssm:AmazonCloudWatch-linux -s

# Status / stop
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a stop

# Agent log file (first stop for troubleshooting)
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

**Minimal agent config for memory + disk (the metrics EC2 lacks by default):**
```json
{
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": { "InstanceId": "${aws:InstanceId}" },
    "metrics_collected": {
      "mem":  { "measurement": ["mem_used_percent"] },
      "disk": { "measurement": ["used_percent"], "resources": ["/"] }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          { "file_path": "/var/log/myapp/app.log",
            "log_group_name": "/myapp/production",
            "log_stream_name": "{instance_id}" }
        ]
      }
    }
  }
}
```

---

## 8. EKS / Container Insights

```bash
# Enable control plane logs on an EKS cluster
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Install the CloudWatch Observability add-on (Container Insights + App Signals)
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability

# Check add-on status
aws eks describe-addon --cluster-name my-cluster --addon-name amazon-cloudwatch-observability

# Verify agent + Fluent Bit pods are running
kubectl get pods -n amazon-cloudwatch

# Set retention on the (expensive) control-plane log group
aws logs put-retention-policy \
  --log-group-name /aws/eks/my-cluster/cluster --retention-in-days 30
```

---

## 9. Synthetics, Anomaly Detection & Metric Streams

```bash
# List canaries / start / stop
aws synthetics describe-canaries
aws synthetics start-canary --name my-heartbeat-canary
aws synthetics stop-canary  --name my-heartbeat-canary

# Create an anomaly detector on a metric
aws cloudwatch put-anomaly-detector \
  --single-metric-anomaly-detector '{
    "Namespace":"AWS/ApiGateway","MetricName":"Count",
    "Dimensions":[{"Name":"ApiName","Value":"orders-api"}],"Stat":"Sum"}'

aws cloudwatch describe-anomaly-detectors

# Create a metric stream to Firehose (export to S3/Datadog)
aws cloudwatch put-metric-stream \
  --name all-metrics-to-lake \
  --firehose-arn arn:aws:firehose:ap-south-1:111122223333:deliverystream/metrics-stream \
  --role-arn arn:aws:iam::111122223333:role/CWMetricStreamRole \
  --output-format json

aws cloudwatch list-metric-streams
aws cloudwatch stop-metric-stream  --name all-metrics-to-lake
aws cloudwatch start-metric-stream --names all-metrics-to-lake
```

---

## 10. X-Ray

```bash
# Get service graph for a time window
aws xray get-service-graph \
  --start-time $(date -d '1 hour ago' +%s) --end-time $(date +%s)

# Get trace summaries (find slow/faulted traces)
aws xray get-trace-summaries \
  --start-time $(date -d '1 hour ago' +%s) --end-time $(date +%s) \
  --filter-expression 'responsetime > 1 OR error'

# Fetch full traces by ID
aws xray batch-get-traces --trace-ids <trace-id>

# Manage sampling rules (e.g., 100% of errors, 5% of successes)
aws xray get-sampling-rules
aws xray create-sampling-rule --cli-input-json file://sampling-rule.json
```

---

## 11. EventBridge

```bash
# Rule matching an event pattern (EC2 state change)
aws events put-rule \
  --name ec2-stopped-rule \
  --event-pattern '{"source":["aws.ec2"],"detail-type":["EC2 Instance State-change Notification"],"detail":{"state":["stopped"]}}'

# Scheduled (cron) rule
aws events put-rule --name nightly-report --schedule-expression "cron(0 2 * * ? *)"

# Attach a target
aws events put-targets --rule ec2-stopped-rule \
  --targets "Id"="1","Arn"="arn:aws:sns:ap-south-1:111122223333:ops-alerts"

aws events list-rules
aws events delete-rule --name ec2-stopped-rule --force
```
