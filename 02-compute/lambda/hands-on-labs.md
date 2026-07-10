# 🧪 Hands-On Labs

Practical exercises covering both Lambda Architecture (the data pattern) and AWS Lambda (the compute service). Do them in order — each builds on the concepts from [`README.md`](./README.md).

---

## Lab 1: Build a Mini Lambda Architecture Pipeline (Local Python)

**Objective:** Implement a tiny end-to-end Lambda Architecture — master dataset, batch layer, speed layer, serving layer — entirely on your local machine.

**Prerequisites**

```bash
pip install pandas pandas-streaming
```

### Step 1 — Simulate the raw data source

```python
import datetime

# Our immutable master dataset (simulated log file)
master_dataset = [
    {"timestamp": "2026-07-06 10:00:00", "amount": 100},
    {"timestamp": "2026-07-06 10:05:00", "amount": 150},
    {"timestamp": "2026-07-06 10:10:00", "amount": 200},
]

# Real-time data streaming in right now (not yet processed by batch)
realtime_stream = [
    {"timestamp": "2026-07-06 10:12:00", "amount": 50},
    {"timestamp": "2026-07-06 10:13:00", "amount": 300},
]
```

### Step 2 — Batch Layer (historical aggregates)

```python
import pandas as pd

def run_batch_layer(dataset):
    df = pd.DataFrame(dataset)
    # Simulating a heavy batch computation (e.g., total historical sales)
    batch_view = {"total_sales": df["amount"].sum(), "record_count": len(df)}
    return batch_view

# Run batch
historical_view = run_batch_layer(master_dataset)
print(f"Batch View Computed: {historical_view}")
```

### Step 3 — Speed Layer (instant stream aggregate)

```python
def run_speed_layer(stream):
    df_stream = pd.DataFrame(stream)
    # Fast, low-latency aggregate of recent stream only
    realtime_view = {"total_sales": df_stream["amount"].sum(), "record_count": len(df_stream)}
    return realtime_view

# Run speed layer
current_realtime_view = run_speed_layer(realtime_stream)
print(f"Speed View Computed: {current_realtime_view}")
```

### Step 4 — Serving Layer (merge both views)

```python
def query_serving_layer(batch_view, realtime_view):
    # Merge the views
    total_sales = batch_view["total_sales"] + realtime_view["total_sales"]
    total_records = batch_view["record_count"] + realtime_view["record_count"]

    return {
        "Unified_Total_Sales": total_sales,
        "Unified_Total_Records": total_records
    }

# Final output seen by the user/dashboard
final_result = query_serving_layer(historical_view, current_realtime_view)
print("\n--- Serving Layer Query Result ---")
print(final_result)
```

**Expected Output**

```
Batch View Computed: {'total_sales': 450, 'record_count': 3}
Speed View Computed: {'total_sales': 350, 'record_count': 2}

--- Serving Layer Query Result ---
{'Unified_Total_Sales': 800, 'Unified_Total_Records': 5}
```

### Try it yourself
- Add a 4th entry to `realtime_stream` and re-run only the speed + serving layers (no need to touch the batch layer) — this is the *separation of concerns* benefit in action.
- "Promote" the realtime records into `master_dataset` and rerun the batch layer — this simulates a real batch cycle absorbing the speed layer's data.
- Introduce a bug in `run_batch_layer`, "fix" it, and rerun from the (immutable) master dataset to see fault tolerance in practice.

---

## Lab 2: Deploy Your First AWS Lambda Function via SAM

**Objective:** Take the configuration concepts from Part 3 of the README and deploy a real function.

**Prerequisites:** AWS CLI configured (`aws configure`), SAM CLI installed, an AWS account.

```bash
# 1. Scaffold a project
sam init
# Choose: AWS Quick Start Templates → Hello World Example → Python 3.13 → arm64

# 2. Replace the generated template.yaml with the production template
#    (see commands-cheatsheet.md → "Full SAM Template Reference")

# 3. Build
sam build

# 4. Deploy (guided — answers get saved to samconfig.toml)
sam deploy --guided

# 5. Invoke the deployed function
aws lambda invoke \
  --function-name ProcessDataFunction \
  --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  response.json
cat response.json

# 6. Verify configuration matches what you set
aws lambda get-function-configuration --function-name ProcessDataFunction
```

**Checkpoints to verify in the console or via CLI:**
- Architecture = `arm64`
- Memory = 1536 MB
- Timeout = 30 sec
- Ephemeral storage = 1024 MB
- SnapStart = `ApplyOn: PublishedVersions`

**Teardown:** `sam delete`

---

## Lab 3: Observe Cold Start vs Warm Start in Practice

**Objective:** See the latency difference from Part 2 of the README with your own eyes in CloudWatch Logs.

```bash
# 1. Invoke immediately after deploy (cold)
aws lambda invoke --function-name ProcessDataFunction --payload '{}' \
  --cli-binary-format raw-in-base64-out out1.json

# 2. Invoke again right away (warm)
aws lambda invoke --function-name ProcessDataFunction --payload '{}' \
  --cli-binary-format raw-in-base64-out out2.json

# 3. Pull the REPORT lines from logs — compare "Init Duration" (only appears on cold starts)
aws logs filter-log-events \
  --log-group-name /aws/lambda/ProcessDataFunction \
  --filter-pattern "REPORT"
```

- Compare the two REPORT lines: the first invoke should show an `Init Duration`; the second should not (or should be much faster overall).
- Wait 15+ minutes without invoking, then invoke again — you should see a fresh `Init Duration` appear, confirming the container went idle and was recycled.
- Enable SnapStart (Lab 2's template already has it) and repeat — compare the `Init Duration` values before/after.

---

## Lab 4: Configure Concurrency Controls

**Objective:** Feel the difference between Reserved and Provisioned concurrency.

```bash
# 1. Cap concurrency at 5 to protect a downstream DB
aws lambda put-function-concurrency \
  --function-name ProcessDataFunction \
  --reserved-concurrent-executions 5

# 2. Fire 20 concurrent invokes and watch some get throttled
for i in $(seq 1 20); do
  aws lambda invoke --function-name ProcessDataFunction --payload '{}' \
    --cli-binary-format raw-in-base64-out "out_$i.json" &
done
wait

# 3. Check for TooManyRequestsException / throttle errors in CloudWatch
aws logs filter-log-events \
  --log-group-name /aws/lambda/ProcessDataFunction \
  --filter-pattern "Throttl"

# 4. Now eliminate cold starts on 2 always-ready instances
aws lambda publish-version --function-name ProcessDataFunction
aws lambda put-provisioned-concurrency-config \
  --function-name ProcessDataFunction \
  --qualifier 1 \
  --provisioned-concurrent-executions 2
```

Compare Init Duration for the first two invokes after enabling provisioned concurrency — they should be near zero.

---

## Lab 5 (Challenge): Ephemeral Storage for Large File Processing

**Objective:** Practical use of the `/tmp` configuration option.

1. Increase ephemeral storage: `EphemeralStorage: Size: 2048` in your template.
2. Write a handler that generates or downloads a large temp file (e.g. a multi-MB PDF or dataset) into `/tmp`.
3. Redeploy and invoke; confirm via logs that the function completes without a "No space left on device" error.
4. Reduce storage back to 512 MB and reproduce the failure to see the limit in action.

---

## Lab 6 (Stretch Goal): From Lambda Architecture to Kappa Architecture

**Objective:** Conceptually refactor Lab 1 to remove the dual-codebase problem.

1. Take your Lab 1 pipeline and **delete the batch layer entirely**.
2. Replay the *entire* `master_dataset` through `run_speed_layer` (treating history as just "a longer stream").
3. Merge that single unified view directly — no separate serving-layer merge step needed.
4. Discuss/write down: what did you gain (one codebase, simpler ops)? What did you give up (the batch layer's easy full-accuracy recompute guarantee)?

This is a discussion/design exercise, not a full implementation — a good candidate for your own write-up once you're comfortable with Lab 1.
