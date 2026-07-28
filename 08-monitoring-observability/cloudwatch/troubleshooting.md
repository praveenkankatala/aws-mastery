# CloudWatch — Troubleshooting Guide

Common issues, their real error messages, root causes, and fixes — organized by area.

---

## Table of Contents

1. [Metrics Problems](#1-metrics-problems)
2. [Alarm Problems](#2-alarm-problems)
3. [CloudWatch Agent Problems](#3-cloudwatch-agent-problems)
4. [Logs Problems](#4-logs-problems)
5. [Logs Insights Problems](#5-logs-insights-problems)
6. [EKS / Container Insights Problems](#6-eks--container-insights-problems)
7. [Custom Metrics / EMF Problems](#7-custom-metrics--emf-problems)
8. [X-Ray / Tracing Problems](#8-x-ray--tracing-problems)
9. [Cost Surprises](#9-cost-surprises)

---

## 1. Metrics Problems

### "My EC2 memory / disk-space metrics don't exist"
- **Symptom:** `AWS/EC2` namespace has CPU and network but no memory or disk %.
- **Cause:** these are OS-level metrics; the hypervisor cannot see them. This is by design, not a bug.
- **Fix:** install the **Unified CloudWatch Agent** on the instance; metrics arrive in the `CWAgent` namespace, not `AWS/EC2`. (People often look in the wrong namespace after installing.)

### "EC2 metrics only update every 5 minutes"
- **Cause:** Basic Monitoring is the default (5-minute periods).
- **Fix:** enable **Detailed Monitoring** for 1-minute granularity:
  ```bash
  aws ec2 monitor-instances --instance-ids i-0123456789abcdef0
  ```
  (Detailed monitoring is chargeable.)

### "My metric disappeared from the console"
- **Cause:** metrics without new data points stop appearing in *list* views after ~2 weeks, and data ages out per the retention ladder (1-min data: 15 days; 5-min: 63 days; 1-hour: 15 months).
- **Fix:** nothing is "deleted" early — query older aggregates with a wider period via `get-metric-data`. If you need raw high-res data long-term, export via Metric Streams to S3.

### "S3 request metrics are all missing"
- **Cause:** S3 **request metrics** (AllRequests, 4xxErrors, latency…) are **disabled by default**; only daily storage metrics are free/automatic.
- **Fix:** S3 console → bucket → Metrics → *Create filter* to enable 1-minute request metrics (per bucket or prefix).

### `InvalidParameterValue: The value ... is not a valid dimension`
- **Cause:** malformed `--dimensions` syntax; the CLI wants `Name=...,Value=...` pairs for `get-metric-statistics` but `Key=Value` shorthand for `put-metric-data`.
- **Fix:** `put-metric-data --dimensions Environment=Prod,Tier=Premium` vs `get-metric-statistics --dimensions Name=InstanceId,Value=i-...`

---

## 2. Alarm Problems

### Alarm stuck in `INSUFFICIENT_DATA`
The single most common CloudWatch complaint. Causes, in order of likelihood:
1. **Dimension mismatch** — the alarm's dimensions must match the metric's dimensions *exactly* (including case). An alarm on `CPUUtilization` with a typo'd InstanceId watches a metric that doesn't exist.
2. **Sparse metric** — metric filters emit nothing during quiet periods. Fix: `--treat-missing-data notBreaching` (or set `defaultValue=0` on the metric filter).
3. **Period vs data frequency mismatch** — alarming with `--period 60` on a metric published every 5 minutes = 4 empty evaluation windows out of 5.
4. **New alarm** — it simply hasn't collected enough evaluation periods yet; wait `period × evaluation-periods`.

### "Alarm fired but I got no email"
1. **SNS subscription never confirmed** — the #1 cause. Check: `aws sns list-subscriptions-by-topic --topic-arn <arn>` — status must not be `PendingConfirmation`.
2. Email went to spam.
3. Alarm actions disabled: check `ActionsEnabled` in `describe-alarms`; re-enable with `enable-alarm-actions`.
4. Wrong topic ARN region — SNS topic must be in the **same region** as the alarm.

### "Alarm flaps constantly (OK ↔ ALARM)"
- **Cause:** threshold too close to normal operating range, or single evaluation period on a noisy metric.
- **Fix:** raise `--evaluation-periods` (e.g., 3 of 3), use `--datapoints-to-alarm` for M-of-N logic ("3 breaches out of 5 periods"), or switch to an anomaly-detection alarm. Wrap related alarms in a **composite alarm** so one page fires, not five.

### "Anomaly detection band looks wrong / alarms on normal traffic"
- **Cause:** the ML model has too little history (needs up to ~2 weeks for stable seasonal bands), or a recent legitimate pattern change (product launch).
- **Fix:** wait for training; widen the band (the `2` in `ANOMALY_DETECTION_BAND(m1, 2)` is the standard-deviation multiplier — try 3); exclude known windows from training in the anomaly-detector configuration.

### "EC2 recover action didn't work"
- **Cause:** recover only applies to `StatusCheckFailed_System` (AWS hardware), not `StatusCheckFailed_Instance` (your OS); and only supported instance types/EBS-only instances qualify.
- **Fix:** use `arn:aws:automate:<region>:ec2:reboot` for instance-level failures; verify instance-type support.

---

## 3. CloudWatch Agent Problems

**Golden rule:** the agent's own log answers 90% of cases:
`/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`

### `AccessDeniedException` / `is not authorized to perform: logs:PutLogEvents`
- **Cause:** instance role lacks permissions.
- **Fix:** attach **`CloudWatchAgentServerPolicy`** to the instance profile. If using SSM Parameter Store configs, also `AmazonSSMReadOnlyAccess`.

### Agent runs but no metrics appear
1. Config file never loaded — running `-a fetch-config ... -s` is mandatory after every config change; editing the JSON alone does nothing.
2. Looking in the wrong namespace — agent metrics go to `CWAgent` (or your custom `namespace` value).
3. JSON config syntax error — check agent log for `Fail to fetch/parse json config`.
4. **No outbound path** — instance in a private subnet with no NAT gateway and no CloudWatch **VPC endpoints** (`com.amazonaws.<region>.monitoring` and `.logs`). Add endpoints or NAT.

### `unable to detect credentials` on-premises
- **Cause:** outside EC2 there's no instance metadata service.
- **Fix:** run the agent in `onPremise` mode (`-m onPremise`) with a credentials profile in `/root/.aws/credentials` referenced by `common-config.toml`.

### Log file not shipping
- Path glob doesn't match (`file_path` must resolve to actual files), file permissions block the agent user, or the log group in another region than expected (`region` override in config).

---

## 4. Logs Problems

### `ResourceNotFoundException: The specified log group does not exist`
- **Cause:** group not created yet, or wrong region in your CLI call.
- **Fix:** `aws logs create-log-group ...` first; confirm `--region`.

### `DataAlreadyAcceptedException` / `InvalidSequenceTokenException` (scripted `put-log-events`)
- **Cause:** legacy sequence-token handling when multiple writers hit one stream.
- **Fix:** newer CLI/SDK versions no longer require sequence tokens; upgrade. Best practice regardless: **one writer per log stream** (e.g., stream per instance-id).

### "Lambda function produces no logs"
- **Cause:** execution role missing `logs:CreateLogGroup/CreateLogStream/PutLogEvents`.
- **Fix:** attach `AWSLambdaBasicExecutionRole`.

### `ThrottlingException` on log ingestion
- **Cause:** exceeding per-stream/per-account `PutLogEvents` quotas.
- **Fix:** spread across more streams, batch events, or front with Kinesis Firehose.

### Export-to-S3 task fails
- **Cause:** S3 bucket policy doesn't grant the CloudWatch Logs service principal `s3:PutObject`, or bucket in a different region than the log group.
- **Fix:** add the documented bucket policy for `logs.<region>.amazonaws.com`; same-region bucket.

---

## 5. Logs Insights Problems

### "Query returned no results but I know the logs exist"
1. **Time range** — Insights defaults to a short window; your events may be outside it.
2. Wrong log group(s) selected.
3. `filter` is case-sensitive: `like /error/` won't match `ERROR`. Use `like /(?i)error/` for case-insensitive regex.

### "Query keeps timing out / scans too much"
- **Fix:** narrow the time range, add `filter` early in the pipeline (filters push down), `limit` results, and split monster queries per log group. For repeated heavy analysis, export to S3 + Athena instead.

### "Can't query my JSON fields"
- **Cause:** log lines aren't valid single-line JSON, so auto-discovery of fields fails.
- **Fix:** emit structured single-line JSON (Powertools/EMF does this); otherwise use `parse` to extract fields manually.

---

## 6. EKS / Container Insights Problems

### No data in Container Insights after installing the add-on
1. **Node role** missing `CloudWatchAgentServerPolicy` (the DaemonSet pods inherit node credentials unless using IRSA).
2. Pods crashlooping: `kubectl get pods -n amazon-cloudwatch` → `kubectl logs <pod>` for the actual error.
3. Private cluster without VPC endpoints for `logs` + `monitoring`.

### Control-plane log groups empty
- **Cause:** logging types not enabled on the cluster — it's off by default.
- **Fix:** `aws eks update-cluster-config --logging ...` (see cheatsheet §8); logs can take ~15 minutes to start flowing.

### Pod stuck `Pending` and metrics don't say why
- **Cause:** metrics show *state*, not *reason*. The reason lives in Kubernetes **Events** (`FailedScheduling: insufficient memory`).
- **Fix:** `kubectl describe pod <pod>` and stream cluster events into your log pipeline (README §7.4 best practice #3) so `ErrImagePull` / `FailedScheduling` are searchable.

### `OOMKilled` containers
- **Cause:** container exceeded its manifest memory **limit**; kernel killed it (`container_memory_oom_killed > 0`).
- **Fix:** raise the limit or fix the leak; set *requests* ≈ observed steady-state, *limits* with sensible headroom.

### Latency spikes with low CPU usage — the throttling trap
- **Cause:** `container_cpu_cfs_throttled_periods` rising — the container hits its CPU **limit** and is artificially slowed even though node CPU looks fine.
- **Fix:** raise/remove the CPU limit (many shops set requests but no limits for latency-sensitive services).

### Container Insights bill exploding
- **Cause:** verbose containers × Fluent Bit shipping every namespace.
- **Fix:** namespace-exclusion filter (README §7.4), log-group retention caps, Logs-IA for debug-grade streams.

---

## 7. Custom Metrics / EMF Problems

### EMF metrics never appear
1. JSON not printed as a **single line** to stdout, or the `_aws` block malformed (validate the schema — `Timestamp` in **milliseconds**, `Dimensions` is an array of arrays).
2. Printed to stderr in a runtime that routes it differently, or logs not reaching CloudWatch Logs at all (fix Lambda basic execution role first).
3. Metrics appear ~1 min after log ingestion — not instantly.

### "My Lambda got slower after adding metrics"
- **Cause:** synchronous `PutMetricData` inside the handler — adds 50–100 ms per invocation and can throttle.
- **Fix:** switch to EMF (print JSON) — zero added latency.

### `LimitExceededException` on `PutMetricData`
- **Cause:** API throttling from high-frequency direct publishing.
- **Fix:** batch up to 1,000 values / 150 datapoints per call, aggregate client-side with `StatisticValues`, or move to EMF.

### Bill shock from custom metrics
- **Cause:** high-cardinality dimension (user ID, request ID, pod name...). Every unique dimension-value combo = one billed metric at $0.30/month.
- **Fix:** dimensions only for low-cardinality categories (env, tier, region); unique IDs go in EMF body fields → query via Logs Insights for free. Audit: `aws cloudwatch list-metrics --namespace <ns> | grep -c MetricName`.

---

## 8. X-Ray / Tracing Problems

### No traces at all
1. Tracing not enabled: Lambda `Mode=Active`, API GW stage X-Ray flag, ECS task with X-Ray daemon/ADOT sidecar.
2. Role missing `AWSXRayDaemonWriteAccess`.
3. **Sampling** — default rule records 1 req/sec + 5%; low-traffic test apps may legitimately show few traces. Temporarily set 100% sampling while testing.

### Trace timeline exists but the Logs tab is empty
- **Cause:** log lines don't contain the trace ID, or the log group isn't associated with the traced service.
- **Fix:** inject the ID (Powertools `Logger`, or the custom logging filter from README §10.2); confirm the exact string appears in raw logs.

### Traces and logs misaligned in time / won't stitch
- **Cause:** server clock drift.
- **Fix:** enforce NTP (`chronyd` on Amazon Linux) — more than a few seconds of drift breaks chronological correlation in the UI.

### Broken/partial traces across services
- **Cause:** an intermediate hop drops the `X-Amzn-Trace-Id` header (proxies, custom HTTP clients not instrumented).
- **Fix:** instrument all clients (X-Ray SDK / OTel auto-instrumentation) and whitelist the header through proxies.

---

## 9. Cost Surprises

| Symptom | Root cause | Fix |
|---|---|---|
| Huge "DataProcessing-Bytes" line item | Log ingestion (charged per GB ingested, not just stored) | Filter at source (Fluent Bit), drop debug logs, Logs-IA class |
| Storage grows forever | Log groups default to **Never Expire** | `put-retention-policy` on every group; automate via a compliance Lambda |
| Custom metrics line item exploding | Cardinality (see §7) | Low-cardinality dimensions + EMF body fields |
| High `GetMetricData` API costs | Third-party tool polling metrics | Replace polling with **Metric Streams** → Firehose |
| Dashboard charges | > 3 dashboards ($3/dashboard/month beyond free tier) | Consolidate; use automatic dashboards where possible |
| Alarm charges creep | High-resolution + composite alarms bill more than standard | Reserve high-res alarms for genuinely sub-minute needs |

---

## Quick Diagnostic Flowchart

```
Metric/alarm not behaving?
 ├── Data exists?  -> list-metrics / get-metric-data for the EXACT namespace+dimensions
 │      └── No -> producer problem (agent config, IAM, EMF schema, S3 request metrics off)
 ├── Data exists but alarm INSUFFICIENT_DATA?
 │      └── dimensions mismatch | period mismatch | treat-missing-data
 ├── Alarm ALARM but no notification?
 │      └── SNS confirmed? same region? actions enabled?
 └── Everything fires but too noisy?
        └── M-of-N datapoints | composite alarms | anomaly detection
```
