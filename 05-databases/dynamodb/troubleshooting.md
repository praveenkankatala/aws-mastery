# Amazon DynamoDB — Troubleshooting Guide

> Real error messages, what actually causes them, and how to fix them. Organized so you can Ctrl-F the exception name you're staring at.

---

## Table of Contents

- [How to Diagnose Anything](#how-to-diagnose-anything)
- [Quick Error Index](#quick-error-index)
- [1. Table Creation & Schema Errors](#1-table-creation--schema-errors)
- [2. Throttling & Capacity Errors](#2-throttling--capacity-errors)
- [3. Validation & Expression Errors](#3-validation--expression-errors)
- [4. Conditional Write Failures](#4-conditional-write-failures)
- [5. Transaction Errors](#5-transaction-errors)
- [6. Index Problems](#6-index-problems)
- [7. Query & Scan Problems](#7-query--scan-problems)
- [8. Consistency & Missing Data](#8-consistency--missing-data)
- [9. Batch Operation Problems](#9-batch-operation-problems)
- [10. Access & Permission Errors](#10-access--permission-errors)
- [11. Streams & Lambda Problems](#11-streams--lambda-problems)
- [12. TTL Problems](#12-ttl-problems)
- [13. Global Tables Problems](#13-global-tables-problems)
- [14. Backup, Restore, Import & Export Problems](#14-backup-restore-import--export-problems)
- [15. DAX Problems](#15-dax-problems)
- [16. SDK & Client Problems](#16-sdk--client-problems)
- [17. CLI-Specific Problems](#17-cli-specific-problems)
- [18. DynamoDB Local Problems](#18-dynamodb-local-problems)
- [19. Performance Problems](#19-performance-problems)
- [20. Cost Surprises](#20-cost-surprises)
- [21. Data Type Gotchas](#21-data-type-gotchas)
- [22. Diagnostic Command Toolkit](#22-diagnostic-command-toolkit)
- [23. Incident Runbooks](#23-incident-runbooks)

---

## How to Diagnose Anything

Work through this before diving into a specific error.

```
1. WHAT is the exact exception name and message?
   → Don't paraphrase. "It's throwing an error" is not a diagnosis.

2. WHERE does it come from?
   → 4xx  = your request is wrong (fix the code)
   → 5xx  = AWS-side (retry with backoff)
   → 429 / ProvisionedThroughputExceeded = capacity (fix design or capacity)

3. WHEN did it start?
   → After a deploy?          → your change
   → After a traffic increase? → capacity or hot partition
   → Intermittently forever?   → eventual consistency or retry gap

4. HOW OFTEN?
   → Every request  → configuration or permissions
   → Some requests  → hot key, specific data, or race condition
   → Under load only → capacity

5. WHAT does CloudWatch say?
   → ThrottledRequests, SystemErrors, UserErrors, ConsumedCapacity,
     SuccessfulRequestLatency, and Contributor Insights top keys
```

**The single most useful debugging flag:**

```bash
--return-consumed-capacity INDEXES
```

Add it to any request. If a query you thought was cheap reports 400 RCU, you've found your bug.

**The second most useful:**

```bash
--return-values-on-condition-check-failure ALL_OLD
```

When a conditional write fails, this shows you the item that blocked it — no second read needed.

---

## Quick Error Index

| Error / symptom | Section |
|---|---|
| `ResourceNotFoundException` | [1](#1-table-creation--schema-errors), [10](#10-access--permission-errors) |
| `ResourceInUseException` | [1](#1-table-creation--schema-errors) |
| `ValidationException: ... not defined in AttributeDefinitions` | [1](#1-table-creation--schema-errors) |
| `LimitExceededException` | [1](#1-table-creation--schema-errors) |
| `ProvisionedThroughputExceededException` | [2](#2-throttling--capacity-errors) |
| `ThrottlingException` | [2](#2-throttling--capacity-errors) |
| `RequestLimitExceeded` | [2](#2-throttling--capacity-errors) |
| `ValidationException: Attribute name is a reserved keyword` | [3](#3-validation--expression-errors) |
| `ValidationException: ExpressionAttributeValues ... not used` | [3](#3-validation--expression-errors) |
| `ValidationException: Item size ... exceeded` | [3](#3-validation--expression-errors) |
| `ValidationException: The provided key element does not match the schema` | [3](#3-validation--expression-errors) |
| `ValidationException: Two document paths overlap` | [3](#3-validation--expression-errors) |
| `ConditionalCheckFailedException` | [4](#4-conditional-write-failures) |
| `TransactionCanceledException` | [5](#5-transaction-errors) |
| `TransactionConflictException` | [5](#5-transaction-errors) |
| `IdempotentParameterMismatchException` | [5](#5-transaction-errors) |
| `ValidationException: Consistent reads are not supported on GSIs` | [6](#6-index-problems) |
| Item missing from GSI | [6](#6-index-problems) |
| `ItemCollectionSizeLimitExceededException` | [6](#6-index-problems) |
| `ValidationException: Query key condition not supported` | [7](#7-query--scan-problems) |
| Query returns empty page but has `LastEvaluatedKey` | [7](#7-query--scan-problems) |
| Read-after-write returns stale data | [8](#8-consistency--missing-data) |
| `UnprocessedItems` / `UnprocessedKeys` | [9](#9-batch-operation-problems) |
| `ValidationException: Provided list of item keys contains duplicates` | [9](#9-batch-operation-problems) |
| `AccessDeniedException` | [10](#10-access--permission-errors) |
| `UnrecognizedClientException` / `InvalidSignatureException` | [10](#10-access--permission-errors), [16](#16-sdk--client-problems) |
| Lambda not triggered by stream | [11](#11-streams--lambda-problems) |
| `Iterator age` climbing | [11](#11-streams--lambda-problems) |
| TTL items never delete | [12](#12-ttl-problems) |
| `ReplicationLatency` high | [13](#13-global-tables-problems) |
| `TableAlreadyExistsException` on import | [14](#14-backup-restore-import--export-problems) |
| `PointInTimeRecoveryUnavailableException` | [14](#14-backup-restore-import--export-problems) |
| Can't connect to DAX | [15](#15-dax-problems) |
| `NoRegionError` / `NoCredentialsError` | [16](#16-sdk--client-problems) |
| `Float types are not supported` (boto3) | [16](#16-sdk--client-problems) |
| `Invalid base64` on CLI v2 | [17](#17-cli-specific-problems) |
| High p99 latency | [19](#19-performance-problems) |
| Unexpected bill | [20](#20-cost-surprises) |
| Empty string rejected | [21](#21-data-type-gotchas) |
| Numbers sorting wrong | [21](#21-data-type-gotchas) |

---

## 1. Table Creation & Schema Errors

### `ValidationException: One or more parameter values were invalid: Some AttributeDefinitions are not used in KeySchema`

**Cause:** you declared an attribute in `--attribute-definitions` that isn't used as a key in the table, an LSI, or a GSI.

**Why people hit it:** they think `AttributeDefinitions` is a schema declaration. It isn't. DynamoDB is schemaless — this parameter exists *only* to type the key attributes.

```bash
# ❌ WRONG — Email is not a key anywhere
--attribute-definitions \
    AttributeName=UserId,AttributeType=S \
    AttributeName=Email,AttributeType=S \
--key-schema AttributeName=UserId,KeyType=HASH

# ✅ RIGHT
--attribute-definitions AttributeName=UserId,AttributeType=S \
--key-schema AttributeName=UserId,KeyType=HASH
```

**Rule:** the set of attributes in `AttributeDefinitions` must equal *exactly* the set of key attributes across the table and all indexes — no more, no less.

---

### `ValidationException: Invalid KeySchema: Some index key attribute is not defined in AttributeDefinitions`

**Cause:** the mirror image — you used an attribute in an index key without declaring it.

```bash
# Adding a GSI on Status requires Status in AttributeDefinitions,
# AND the table's own key attributes if the GSI uses them as a range key
aws dynamodb update-table --table-name Orders \
  --attribute-definitions \
      AttributeName=Status,AttributeType=S \
      AttributeName=OrderDate,AttributeType=S \
  --global-secondary-index-updates '[{"Create":{...}}]'
```

---

### `ResourceInUseException: Table already exists: Orders`

**Cause:** the table exists, or is still being created/deleted.

```bash
# Check
aws dynamodb describe-table --table-name Orders --query 'Table.TableStatus'

# If DELETING, wait it out
aws dynamodb wait table-not-exists --table-name Orders

# Make your scripts idempotent
aws dynamodb describe-table --table-name Orders >/dev/null 2>&1 \
  || aws dynamodb create-table --table-name Orders ...
```

---

### `ResourceNotFoundException: Requested resource not found`

**Five causes, in order of likelihood:**

1. **Wrong Region.** By far the most common.
   ```bash
   aws configure get region
   aws dynamodb list-tables --region us-east-1
   aws dynamodb list-tables --region eu-west-1
   ```
2. **Typo or wrong case.** Table names are case-sensitive: `Orders` ≠ `orders`.
3. **Wrong account/profile.** `aws sts get-caller-identity`
4. **Table still `CREATING`.** Add `aws dynamodb wait table-exists`.
5. **Endpoint override left in code** — e.g. `endpoint_url="http://localhost:8000"` still set in a deployed app.

---

### `LimitExceededException: Subscriber limit exceeded`

**Cause:** too many concurrent control-plane operations (creating/updating/deleting tables or indexes), or you've hit the tables-per-Region quota.

```bash
# Check account limits
aws dynamodb describe-limits
aws service-quotas list-service-quotas --service-code dynamodb --output table
```

**Fix:** serialize your table operations, add backoff between them, or request a quota increase. Only one GSI can be created or deleted at a time per table.

---

### `ValidationException: Cannot create more than 5 local secondary indexes per table`

Hard limit. LSIs cannot be added later either. If you need a sixth alternate sort order, use a GSI.

---

### `ValidationException: Number of attributes in KeySchema does not exactly match number of attributes defined in AttributeDefinitions`

**Cause:** a mismatch between the two lists — usually a copy-paste error where one has a stale attribute.

Print both and diff them mentally:

```bash
aws dynamodb describe-table --table-name Orders \
  --query '{Defs:Table.AttributeDefinitions,Keys:Table.KeySchema}'
```

---

### Table stuck in `CREATING` or `UPDATING`

Normal for large GSI backfills — these can take hours. Check progress:

```bash
aws dynamodb describe-table --table-name Orders --query '{
  Table:Table.TableStatus,
  GSI:Table.GlobalSecondaryIndexes[].{Name:IndexName,Status:IndexStatus,Backfilling:Backfilling,Items:ItemCount}}'
```

If `Backfilling: true`, it's working — just wait. If it's been stuck with no progress for many hours, open a support case.

---

### `ValidationException: Table is not currently in a state that allows this operation`

You're issuing an update while another update is in flight. DynamoDB allows only one table-level modification at a time.

```bash
aws dynamodb wait table-exists --table-name Orders   # waits for ACTIVE
```

---

### Can't delete a table

```
ValidationException: Resource cannot be deleted, deletion protection is enabled
```

```bash
aws dynamodb update-table --table-name Orders --no-deletion-protection-enabled
sleep 10
aws dynamodb delete-table --table-name Orders
```

If it's a **global table replica**, you must remove the replica first, or delete replicas in the right order.

---

## 2. Throttling & Capacity Errors

### `ProvisionedThroughputExceededException`

> *"You exceeded your maximum allowed provisioned throughput for a table or for one or more global secondary indexes."*

This is the #1 DynamoDB error. There are **five distinct root causes** and they need different fixes.

---

#### Cause A — genuinely under-provisioned

**Diagnose:**

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB --metric-name ConsumedWriteCapacityUnits \
  --dimensions Name=TableName,Value=Orders \
  --start-time $(date -u -d '3 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Sum Maximum --output table
```

If consumed ≈ provisioned, you simply need more.

**Fix:**
```bash
aws dynamodb update-table --table-name Orders \
  --provisioned-throughput ReadCapacityUnits=200,WriteCapacityUnits=100
# or switch to on-demand
aws dynamodb update-table --table-name Orders --billing-mode PAY_PER_REQUEST
```

---

#### Cause B — hot partition (consumed is LOW but you're still throttled)

**This is the confusing one.** CloudWatch shows 12% utilization and you're being rejected. That means your traffic is concentrated on one partition key value.

**Diagnose with Contributor Insights:**

```bash
aws dynamodb update-contributor-insights \
  --table-name Orders --contributor-insights-action ENABLE
# wait a few minutes, then view in the CloudWatch console:
# CloudWatch → Contributor Insights → DynamoDB rules
```

It will name your hot key directly.

**Fix — pick based on your data:**

| Situation | Fix |
|---|---|
| Time-series key (`2026-08-04`) | Write sharding: `2026-08-04#<0-9>` |
| One dominant tenant | Shard that tenant's key only |
| A single very hot read key | Put DAX or an app cache in front |
| Low-cardinality key (`Status`, `Country`) | **Redesign** — this key was never viable |

**Write sharding, random suffix:**

```python
import random
shard = random.randint(0, 9)
item["PK"] = f"{date}#{shard}"

# Read: query all 10 shards in parallel and merge
```

**Write sharding, deterministic suffix** (better — reads hit one shard):

```python
shard = hash(order_id) % 10
item["PK"] = f"{date}#{shard}"
# Read for a known order_id: compute the same shard, one query
```

---

#### Cause C — a GSI is the bottleneck

**Symptom:** writes to the base table throttle even though the base table has capacity.

DynamoDB will not let an index fall arbitrarily behind, so an under-provisioned GSI back-pressures the table.

**Diagnose:**

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB --metric-name ThrottledRequests \
  --dimensions Name=TableName,Value=Orders Name=GlobalSecondaryIndexName,Value=StatusIndex \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Sum
```

**Fix:** raise the GSI's capacity, add auto scaling to the GSI (people always forget this), or check whether the GSI itself has a hot key — a GSI on `Status` with four values has only four partition keys.

---

#### Cause D — a burst that outran auto scaling

Auto scaling is reactive and takes minutes.

**Fix:**
- Use **scheduled scaling** for known events (batch windows, daily jobs).
- **Pre-warm** before launches:
  ```bash
  aws dynamodb update-table --table-name Orders \
    --warm-throughput ReadUnitsPerSecond=50000,WriteUnitsPerSecond=25000
  ```
- Raise the auto scaling **minimum** so the floor is high enough.
- Or switch to on-demand.

---

#### Cause E — the client isn't retrying properly

Throttling is *expected* and normal at the margins. A well-configured client absorbs it.

```python
# Python
from botocore.config import Config
cfg = Config(retries={"max_attempts": 10, "mode": "adaptive"})
ddb = boto3.resource("dynamodb", config=cfg)
```

```javascript
// JS SDK v3
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
const client = new DynamoDBClient({ maxAttempts: 10, retryMode: "adaptive" });
```

```java
// Java SDK v2
DynamoDbClient.builder()
  .overrideConfiguration(c -> c.retryPolicy(RetryPolicy.builder()
      .numRetries(10).build()))
  .build();
```

> ⚠️ `mode="adaptive"` adds client-side rate limiting on top of retries. It is the right default for DynamoDB.

---

### `ThrottlingException` / `RequestLimitExceeded`

Different from `ProvisionedThroughputExceededException`:

| Error | Meaning |
|---|---|
| `ProvisionedThroughputExceededException` | Table/index capacity exceeded |
| `ThrottlingException` | Control-plane API rate limit (too many DescribeTable etc.) |
| `RequestLimitExceeded` | **Account-level** throughput quota reached |

For `RequestLimitExceeded`:

```bash
aws dynamodb describe-limits
aws service-quotas request-service-quota-increase \
  --service-code dynamodb --quota-code <code> --desired-value <n>
```

For `ThrottlingException` on control-plane calls: **stop calling `DescribeTable` in your request path.** Cache table metadata at startup.

---

### On-demand table is still throttling

On-demand instantly serves any previously reached peak, and can roughly **double** beyond that within about 30 minutes. A 10× instantaneous spike from a cold table can still throttle.

**Fix:**
- Pre-warm with `--warm-throughput` ahead of the event.
- Ramp traffic gradually in load tests so the table learns your peak.
- Check you haven't set `MaxReadRequestUnits` / `MaxWriteRequestUnits` too low:
  ```bash
  aws dynamodb describe-table --table-name Orders --query 'Table.OnDemandThroughput'
  ```

---

## 3. Validation & Expression Errors

### `ValidationException: Invalid ... expression: Attribute name is a reserved keyword; reserved keyword: Status`

**Cause:** ~573 words are reserved. Common offenders: `Name`, `Status`, `Size`, `Value`, `Data`, `Timestamp`, `Year`, `Date`, `Count`, `Type`, `Key`, `Source`, `State`, `Order`, `Group`, `User`, `Role`.

```bash
# ❌
--filter-expression "Status = :s"

# ✅
--filter-expression "#st = :s" \
--expression-attribute-names '{"#st":"Status"}'
```

**Prevention:** alias every attribute name with `#`. It costs nothing and removes an entire class of bug.

---

### `ValidationException: ExpressionAttributeValues contains invalid key: Syntax error; key: ":val"`

You forgot the colon, or used the wrong prefix.

```
Names  → #alias  → ExpressionAttributeNames
Values → :alias  → ExpressionAttributeValues
```

---

### `ValidationException: Value provided in ExpressionAttributeValues unused in expressions`

You declared a placeholder you didn't reference. DynamoDB rejects unused placeholders — usually a leftover from editing an expression.

Same applies to `ExpressionAttributeNames`.

---

### `ValidationException: The provided expression refers to an attribute that does not exist in the item`

Occurs when an `UpdateExpression` does arithmetic on a missing attribute:

```bash
# ❌ fails if Total doesn't exist yet
--update-expression "SET Total = Total + :x"

# ✅ safe
--update-expression "SET Total = if_not_exists(Total, :zero) + :x" \
--expression-attribute-values '{":zero":{"N":"0"},":x":{"N":"10"}}'

# ✅ or use ADD, which treats a missing number as 0
--update-expression "ADD Total :x"
```

---

### `ValidationException: The provided key element does not match the schema`

**Causes:**
1. Missing the sort key on a composite-key table (GetItem/UpdateItem/DeleteItem need **both**).
2. Wrong type — sending `{"N":"1042"}` when the key is `S`.
3. Extra attributes in `--key` — the key must contain *only* the key attributes.
4. Using base-table keys when querying a GSI, or vice versa.

```bash
aws dynamodb describe-table --table-name Orders --query 'Table.KeySchema'
```

---

### `ValidationException: Item size has exceeded the maximum allowed size of 400KB`

**Fixes:**

| Approach | How |
|---|---|
| **Offload to S3** | Store the blob in S3, keep `s3://bucket/key` in the item |
| **Compress** | gzip + base64 the large attribute (often 5–10× reduction) |
| **Vertical partitioning** | Split into multiple items in the same item collection (`SK = "PART#1"`, `"PART#2"`) |
| **Drop redundancy** | Long attribute names count toward the 400 KB — shorten them |

Check sizes before writing:

```python
import json
size = len(json.dumps(item).encode("utf-8"))
if size > 380_000:
    raise ValueError(f"Item too large: {size} bytes")
```

---

### `ValidationException: Two document paths overlap with each other`

You referenced the same path twice in one `UpdateExpression`:

```bash
# ❌
--update-expression "SET Address = :a, Address.City = :c"

# ✅ — set the whole map, or set individual fields, not both
--update-expression "SET Address.City = :c, Address.Zip = :z"
```

---

### `ValidationException: Expression size has exceeded the maximum allowed size`

Expressions are capped at 4 KB. Usually caused by generating a giant `IN (...)` clause.

**Fix:** use `BatchGetItem` for many keys, or restructure so a `Query` on a partition replaces the list.

---

### `ValidationException: Number overflow. Attempting to store a number with magnitude larger than supported range`

DynamoDB numbers support 38 significant digits, roughly 1E-130 to 9.9999E+125. Anything larger must be stored as a String.

---

### `ValidationException: Attribute value cannot be an empty set`

Sets must have at least one member. To remove the last member, `REMOVE` the whole attribute instead of `DELETE`-ing the final element.

---

## 4. Conditional Write Failures

### `ConditionalCheckFailedException`

**This is usually not a bug — it's the feature working.** Your condition evaluated to false.

**Diagnose:** ask DynamoDB what the item actually looked like.

```bash
aws dynamodb update-item --table-name Orders \
  --key '{"UserId":{"S":"U-1"},"OrderDate":{"S":"2026-08-04"}}' \
  --update-expression "SET #s = :new" \
  --condition-expression "#s = :old" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":new":{"S":"SHIPPED"},":old":{"S":"PENDING"}}' \
  --return-values-on-condition-check-failure ALL_OLD
```

**Common real causes:**

| Cause | Fix |
|---|---|
| Item genuinely in a different state | Handle as normal business logic (return 409) |
| Race — another writer won | Retry with fresh read (optimistic locking loop) |
| Item doesn't exist | Use `attribute_exists()` explicitly and return 404 |
| Version mismatch | Re-read, re-apply, retry — bounded number of attempts |
| Wrong assumption about `attribute_not_exists` on composite keys | It checks the **specific PK+SK item**, not the partition |

**Retry pattern:**

```python
for attempt in range(5):
    item = table.get_item(Key=key, ConsistentRead=True)["Item"]
    try:
        table.update_item(
            Key=key,
            UpdateExpression="SET #d = :d, Version = Version + :one",
            ConditionExpression="Version = :v",
            ExpressionAttributeNames={"#d": "Data"},
            ExpressionAttributeValues={":d": new_data, ":one": 1, ":v": item["Version"]},
        )
        break
    except ClientError as e:
        if e.response["Error"]["Code"] != "ConditionalCheckFailedException":
            raise
        time.sleep(0.05 * (2 ** attempt))
else:
    raise RuntimeError("too much contention")
```

> 💰 **Note:** a failed conditional write **still consumes WCU**. A hot retry loop is expensive as well as slow.

---

### High `ConditionalCheckFailedRequests` in CloudWatch

A sustained spike means contention. Options:

- Reduce the number of writers touching the same item.
- Replace read-modify-write with an **atomic counter** (`ADD`) where possible — no condition needed.
- Shard the contended item (e.g. 10 counter items summed on read).
- Add jittered backoff so retries don't synchronize.

---

## 5. Transaction Errors

### `TransactionCanceledException`

The whole transaction was rolled back. **Read `CancellationReasons`** — one entry per action, in order.

```python
try:
    client.transact_write_items(TransactItems=items)
except ClientError as e:
    if e.response["Error"]["Code"] == "TransactionCanceledException":
        for i, r in enumerate(e.response["CancellationReasons"]):
            print(f"action[{i}]: {r.get('Code')} - {r.get('Message')}")
```

| Reason code | Meaning | What to do |
|---|---|---|
| `None` | This action was fine | — |
| `ConditionalCheckFailed` | Your business condition failed | Business logic — return 409 |
| `ItemCollectionSizeLimitExceeded` | LSI item collection > 10 GB | Redesign the key |
| `TransactionConflict` | Another transaction/write touched the same item | Retry with backoff |
| `ProvisionedThroughputExceeded` | Not enough capacity (remember: 2×) | Add capacity |
| `ThrottlingError` | Table throttled | Back off |
| `ValidationError` | Malformed request | Fix the code |

---

### `TransactionConflictException` / frequent `TransactionConflict` reasons

Two transactions competed for the same item.

**Fixes:**
- Retry with exponential backoff **and jitter**.
- Reduce transaction scope — don't include items that don't need atomicity.
- Split hot items (shard a counter rather than transacting on it).
- Ask whether you need a transaction at all: a single conditional `UpdateItem` is already atomic and doesn't conflict this way.

---

### `ValidationException: Transaction request cannot include multiple operations on one item`

The same primary key appears twice in one transaction. Not allowed, even across different action types.

**Fix:** merge them into a single `Update` with a combined expression.

---

### `IdempotentParameterMismatchException`

You reused a `ClientRequestToken` with **different** transaction content within the 10-minute window.

**Fix:** generate a new token whenever the payload changes. Derive the token from the business operation (e.g. `f"transfer-{transfer_id}"`), not from a random value regenerated on retry — the point is that retries reuse it.

---

### Transactions are slower / more expensive than expected

Expected behaviour:
- **2× capacity** per item, reads and writes.
- Two-phase commit adds latency.
- More conflict retries under contention.

**Don't use transactions for:** bulk loads (`BatchWriteItem`), anything one conditional update handles, or high-throughput hot paths.

---

## 6. Index Problems

### `ValidationException: Consistent reads are not supported on global secondary indexes`

Architectural, not configurable. GSIs are updated asynchronously.

**If you need strong consistency on an alternate key:**
- Use an **LSI** (same partition key, alternate sort key — supports `ConsistentRead`).
- Or query the GSI to find the key, then `GetItem` the base table with `ConsistentRead=true`.

---

### Item written but missing from the GSI

**Cause 1 — eventual consistency.** GSI propagation is typically well under a second but is not instant. If you write then immediately query the index, you may miss it.

**Fix:** don't read your own write from a GSI. Use the base table for read-after-write.

**Cause 2 — the item lacks an index key attribute.** This is the sparse-index behaviour: **an item only appears in a GSI if it has *all* the index key attributes.**

```bash
# Item has no Status attribute → will never appear in StatusIndex
aws dynamodb get-item --table-name Orders \
  --key '{"UserId":{"S":"U-1"},"OrderDate":{"S":"2026-08-04"}}' \
  --query 'Item.Status'
```

**Cause 3 — type mismatch.** The GSI key is declared `S` but the item stores a Number. The item is silently excluded.

**Cause 4 — backfill still running.**
```bash
aws dynamodb describe-table --table-name Orders \
  --query 'Table.GlobalSecondaryIndexes[].{Name:IndexName,Status:IndexStatus,Backfilling:Backfilling}'
```

---

### GSI query returns items with missing attributes

**Cause:** the projection doesn't include them. There is **no automatic fallback to the base table.**

```bash
aws dynamodb describe-table --table-name Orders \
  --query 'Table.GlobalSecondaryIndexes[].{Name:IndexName,Projection:Projection}'
```

**Fixes:**
- Recreate the index with `INCLUDE` covering the attributes you need (you cannot modify a projection in place — you must delete and recreate the index).
- Or do a follow-up `BatchGetItem` on the base table.

---

### `ItemCollectionSizeLimitExceededException`

**Cause:** the table has an **LSI**, and one item collection (all items sharing one partition key) exceeded **10 GB**.

**This limit exists only when LSIs are present.**

**Fixes:**
- Change the partition key so collections are smaller (add a shard suffix or a time bucket).
- Remove the LSI and use a GSI instead (requires recreating the table).
- Archive old items out of the collection.

**Monitor proactively:**

```bash
aws dynamodb put-item --table-name Orders --item file://item.json \
  --return-item-collection-metrics SIZE
```

---

### Writes throttled but the base table has plenty of capacity

Classic GSI back-pressure. See [Cause C](#cause-c--a-gsi-is-the-bottleneck) above.

---

### Can't add an LSI to an existing table

You can't. Ever. LSIs are creation-time only.

**Options:**
1. Create a GSI instead (works for most cases).
2. Create a new table with the LSI, then migrate: export to S3 → import, or a Lambda/Glue copy job.

---

### GSI creation is taking forever

Backfilling a large table can take many hours. It's online — the table stays available. Watch `OnlineIndexPercentageProgress` in CloudWatch. On provisioned tables, give the index generous write capacity during backfill, then scale it down.

---

## 7. Query & Scan Problems

### `ValidationException: Query key condition not supported`

**Causes:**

| Mistake | Fix |
|---|---|
| Range operator on the **partition key** | Partition key must use `=` only |
| Non-key attribute in `KeyConditionExpression` | Move it to `FilterExpression`, or add an index |
| `OR` between key conditions | Not supported — run separate queries |
| `contains()` or `ends_with()` on the sort key | Only `begins_with()` is supported for keys |
| Using base-table keys against a GSI | Use the GSI's own key attributes |

```bash
# ❌
--key-condition-expression "UserId BETWEEN :a AND :b"
--key-condition-expression "UserId = :u AND Status = :s"     # Status isn't a key

# ✅
--key-condition-expression "UserId = :u AND OrderDate BETWEEN :a AND :b"
--key-condition-expression "UserId = :u" --filter-expression "#st = :s"
```

---

### Query returns fewer items than expected

**Cause 1 — the 1 MB page limit.** Every Query/Scan returns at most 1 MB. You must paginate.

**Cause 2 — `Limit` applies before the filter.** `--limit 10` means "read 10 items", not "return 10 matching items". After filtering you might get 2.

**Cause 3 — you're not looping on `LastEvaluatedKey`.**

```python
items, kwargs = [], {...}
while True:
    r = table.query(**kwargs)
    items.extend(r["Items"])
    if "LastEvaluatedKey" not in r:
        break
    kwargs["ExclusiveStartKey"] = r["LastEvaluatedKey"]
```

---

### Query returns an empty list but has `LastEvaluatedKey`

**This is normal and catches everyone.** With a `FilterExpression`, DynamoDB reads a 1 MB page, filters it, and returns whatever survived — possibly nothing — along with a cursor.

**Never** treat an empty page as "no results". Loop until `LastEvaluatedKey` is absent.

---

### Scan is extremely slow / times out

Expected — Scan reads the entire table.

**Fixes in order of preference:**
1. **Add a GSI** and Query instead. This is almost always the right answer.
2. Use **parallel scan** if you genuinely need the whole table:
   ```bash
   for i in 0 1 2 3; do
     aws dynamodb scan --table-name Orders --total-segments 4 --segment $i &
   done; wait
   ```
3. Use **export to S3 + Athena** for analytics — zero RCU consumed.
4. Add `--limit` and paginate to avoid client timeouts.

---

### `ScannedCount` is far higher than `Count`

You're reading a lot of data and throwing most of it away. **This is a cost bug, not a correctness bug.**

```bash
--return-consumed-capacity TOTAL \
--query '{Returned:Count,Scanned:ScannedCount,RCU:ConsumedCapacity.CapacityUnits}'
```

**Fix:** move the filter condition into the key schema or a GSI so DynamoDB reads only what you need.

---

### Results aren't in the order I expected

- Sorting only happens **within a partition**, by sort key. Across partitions there is no order.
- Sort keys sort **lexicographically for strings**: `"10" < "9"`. Zero-pad numbers stored as strings.
- Use `--no-scan-index-forward` for descending.
- **Scan has no ordering at all** — never rely on it.

---

## 8. Consistency & Missing Data

### I wrote an item and immediately read it back — it's not there

**Cause:** eventually consistent reads are the default.

```bash
aws dynamodb get-item --table-name Orders --key '...' --consistent-read
```

For GSIs, `ConsistentRead` is not available — read the base table instead.

---

### Item "disappeared"

Checklist:

| Possibility | Check |
|---|---|
| TTL deleted it | `aws dynamodb describe-time-to-live --table-name X` — is TTL on? Did the item have an `ExpiresAt` in the past? |
| A `PutItem` overwrote it | `PutItem` replaces the whole item. Check your write path. |
| Wrong key | Sort key mismatch returns nothing, silently. |
| Wrong Region / table | `aws sts get-caller-identity`, `aws configure get region` |
| Conditional delete succeeded | Check application logs |
| Global table last-writer-wins overwrote it | Another Region wrote the same item |

**Investigate with PITR:**

```bash
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Orders --target-table-name Orders-Investigate \
  --restore-date-time 2026-08-04T09:00:00Z
```

Then query the restored copy without touching production.

---

### Attributes vanished after an update

You used `PutItem` instead of `UpdateItem`. `PutItem` replaces the entire item — omitted attributes are deleted.

```bash
# ❌ deletes everything except what you supply
aws dynamodb put-item --item '{"PK":{"S":"X"},"Status":{"S":"NEW"}}'

# ✅ changes only Status
aws dynamodb update-item --key '{"PK":{"S":"X"}}' \
  --update-expression "SET #s = :v" ...
```

---

### Counter values are wrong / lower than expected

**Cause:** read-modify-write races.

```python
# ❌ lost updates under concurrency
item = table.get_item(Key=k)["Item"]
table.put_item(Item={**item, "Count": item["Count"] + 1})

# ✅ atomic
table.update_item(Key=k, UpdateExpression="ADD #c :one",
                  ExpressionAttributeNames={"#c":"Count"},
                  ExpressionAttributeValues={":one": 1})
```

**Note:** atomic counters are *not idempotent*. A client retry after a network timeout can double-count. If exact counts matter, use a conditional update with a version, or a transaction with `ClientRequestToken`.

---

## 9. Batch Operation Problems

### `UnprocessedItems` / `UnprocessedKeys` returned

**Not an error — a partial success.** DynamoDB throttled some of your batch. **If you ignore this, you silently lose writes.**

```python
import time, random

def batch_write_all(client, table, items):
    requests = [{"PutRequest": {"Item": i}} for i in items]
    for i in range(0, len(requests), 25):
        chunk = requests[i:i+25]
        unprocessed = {table: chunk}
        attempt = 0
        while unprocessed:
            resp = client.batch_write_item(RequestItems=unprocessed)
            unprocessed = resp.get("UnprocessedItems", {})
            if unprocessed:
                attempt += 1
                if attempt > 10:
                    raise RuntimeError("persistent throttling")
                time.sleep(min(2 ** attempt * 0.05, 20) * (0.5 + random.random()))
```

**Or just use the high-level helper**, which handles this for you:

```python
with table.batch_writer() as batch:
    for item in items:
        batch.put_item(Item=item)
```

---

### `ValidationException: Provided list of item keys contains duplicates`

`BatchGetItem` and `BatchWriteItem` reject duplicate keys within a single request.

```python
seen, unique = set(), []
for i in items:
    k = (i["PK"], i["SK"])
    if k not in seen:
        seen.add(k); unique.append(i)
```

---

### `ValidationException: Too many items requested for the BatchWriteItem call`

Hard limits: **25 items** for `BatchWriteItem`, **100 items** for `BatchGetItem`, 16 MB either way. Chunk your input.

---

### BatchWriteItem doesn't support updates or conditions

Correct — only `PutRequest` and `DeleteRequest`, with no `ConditionExpression`. For conditional or partial writes you need individual `UpdateItem` calls or `TransactWriteItems`.

---

### Batch operations are slower than expected

Batches are parallel server-side but capped at 25/100 items. For bulk loading:

- **Best:** `import-table` from S3 — consumes zero write capacity and is dramatically cheaper.
- **Second:** multiple threads each issuing `BatchWriteItem`.
- **Never:** a serial loop of `PutItem` for millions of rows.

---

## 10. Access & Permission Errors

### `AccessDeniedException: User ... is not authorized to perform: dynamodb:Query on resource ...`

**Cause 1 — missing index permission.** This is the #1 cause.

```json
"Resource": [
  "arn:aws:dynamodb:us-east-1:123456789012:table/Orders",
  "arn:aws:dynamodb:us-east-1:123456789012:table/Orders/index/*"
]
```

Without the `/index/*` line, table access works and index queries fail.

**Cause 2 — an explicit `Deny`** in an SCP, permissions boundary, or the role's own policy. Explicit deny always wins.

**Cause 3 — wrong action name.** `Query` and `Scan` are separate. `BatchGetItem` is not covered by `GetItem`. `TransactWriteItems` needs `dynamodb:PutItem`/`UpdateItem`/`DeleteItem` on the target items **plus** `dynamodb:ConditionCheckItem` for ConditionCheck actions.

**Cause 4 — condition keys blocking you**, e.g. `dynamodb:LeadingKeys` restricting which partition keys you may touch.

**Diagnose:**

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/AppRole \
  --action-names dynamodb:Query dynamodb:GetItem dynamodb:Scan \
  --resource-arns \
     arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
     arn:aws:dynamodb:us-east-1:123456789012:table/Orders/index/StatusIndex \
  --output table
```

---

### `AccessDeniedException` mentioning KMS

The table uses a customer managed key and your principal lacks KMS permissions.

```json
{
  "Effect": "Allow",
  "Action": ["kms:Decrypt", "kms:GenerateDataKey", "kms:DescribeKey"],
  "Resource": "arn:aws:kms:us-east-1:123456789012:key/<key-id>"
}
```

You also need to be allowed in the **KMS key policy**, not just your IAM policy.

---

### `UnrecognizedClientException: The security token included in the request is invalid`

- Expired temporary credentials (STS session ended) — refresh.
- Wrong or rotated access keys.
- `AWS_SESSION_TOKEN` missing when using temporary credentials.
- Credentials from a different partition (e.g. GovCloud keys against commercial endpoints).

```bash
aws sts get-caller-identity
env | grep AWS_
```

---

### `InvalidSignatureException: Signature expired`

Your machine's clock is skewed by more than 5 minutes.

```bash
sudo ntpdate -s time.nist.gov      # or: sudo chronyc makestep
timedatectl status
```

---

### VPC endpoint configured but calls still fail

- **Gateway endpoint:** confirm the endpoint is attached to the subnet's **route table**, and that the endpoint policy allows your actions.
- **Interface endpoint:** confirm the security group allows HTTPS (443) from your resources and that private DNS is enabled.
- If you added a `aws:SourceVpce` deny condition, verify the endpoint ID matches exactly.

```bash
aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.us-east-1.dynamodb
```

---

## 11. Streams & Lambda Problems

### Lambda isn't being invoked

**Checklist in order:**

```bash
# 1. Is the stream enabled?
aws dynamodb describe-table --table-name Orders --query 'Table.StreamSpecification'

# 2. Does the mapping exist and is it Enabled?
aws lambda list-event-source-mappings --function-name MyFn \
  --query 'EventSourceMappings[].{State:State,Reason:StateTransitionReason,LastResult:LastProcessingResult}'

# 3. Does the role have stream permissions?
aws iam list-attached-role-policies --role-name MyFnRole
```

The role needs `dynamodb:DescribeStream`, `GetRecords`, `GetShardIterator`, `ListStreams` — all included in the managed policy `AWSLambdaDynamoDBExecutionRole`.

**Other causes:**
- `--starting-position LATEST` means it only sees changes made *after* the mapping was created. Use `TRIM_HORIZON` to process existing records.
- A `--filter-criteria` pattern is excluding your events. Test the pattern carefully — filter syntax is strict.
- The mapping is in `Creating` state — wait for `Enabled`.
- Recreating a table generates a **new stream ARN**; the old mapping is dead.

---

### `IteratorAge` climbing / consumer falling behind

**Meaning:** records are being produced faster than they're consumed. At 24 hours, records are lost forever.

**Causes and fixes:**

| Cause | Fix |
|---|---|
| Lambda too slow | Increase memory (which increases CPU), optimize the handler |
| Batch size too small | Raise `--batch-size` (up to 1000 for DynamoDB streams) |
| Not enough parallelism | Raise `--parallelization-factor` (up to 10 per shard) |
| Poison-pill record blocking the shard | Enable `--bisect-batch-on-function-error` and set `--maximum-retry-attempts` |
| Downstream service slow | Buffer to SQS and process asynchronously |
| Lambda concurrency limit | Raise reserved concurrency |

```bash
aws lambda update-event-source-mapping --uuid <uuid> \
  --batch-size 500 --parallelization-factor 8 \
  --maximum-retry-attempts 3 --bisect-batch-on-function-error \
  --maximum-record-age-in-seconds 3600 \
  --destination-config '{"OnFailure":{"Destination":"arn:aws:sqs:...:dlq"}}'
```

---

### The same batch is processed over and over

A record in the batch keeps failing, and without a retry cap Lambda retries until the record expires (24 hours), blocking the shard entirely.

**Fix:** always set `--maximum-retry-attempts`, `--maximum-record-age-in-seconds`, `--bisect-batch-on-function-error`, and an on-failure destination.

---

### Duplicate processing

Stream delivery to consumers is **at-least-once**. Duplicates are normal, not a bug.

**Fix: make the consumer idempotent.**

```python
def handler(event, context):
    for r in event["Records"]:
        dedupe_key = r["eventID"]     # stable per change record
        if already_processed(dedupe_key):
            continue
        process(r)
        mark_processed(dedupe_key)    # e.g. a DynamoDB item with TTL
```

---

### Records arrive out of order

Ordering is guaranteed **per partition key only**. Across different keys there's no ordering.

If you need global ordering, you need a different design — sequence numbers in the item, or a single partition key (which reintroduces a hot partition).

---

### Can't change `StreamViewType`

You must disable the stream, wait, then re-enable with the new view type. **This resets the stream** and creates a new ARN — existing consumers lose their position and any unread records.

```bash
aws dynamodb update-table --table-name Orders --stream-specification StreamEnabled=false
sleep 60
aws dynamodb update-table --table-name Orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

Plan for the gap.

---

### Can't tell TTL deletions from application deletes

Both arrive as `eventName: REMOVE`. Check `userIdentity`:

```python
ui = record.get("userIdentity", {})
is_ttl = ui.get("type") == "Service" and ui.get("principalId") == "dynamodb.amazonaws.com"
```

---

## 12. TTL Problems

### Items never get deleted

Work down this list — one of these is always the cause:

| # | Check | Fix |
|---|---|---|
| 1 | Is TTL enabled? | `aws dynamodb describe-time-to-live --table-name X` |
| 2 | Does the attribute name match **exactly**? | Case-sensitive. `expiresAt` ≠ `ExpiresAt` |
| 3 | Is it a **Number** type? | `{"N":"1785835200"}` ✅ · `{"S":"1785835200"}` ❌ silently ignored |
| 4 | Is it **seconds**, not milliseconds? | `1785835200` ✅ · `1785835200000` ❌ = year 58,547 |
| 5 | Is the timestamp actually in the past? | `date -d @<value>` |
| 6 | Has enough time passed? | Deletion happens within ~48 h of expiry, sometimes longer |

**Audit script:**

```python
import boto3, time
t = boto3.resource("dynamodb").Table("Sessions")
now = int(time.time())
bad = []
for item in t.scan()["Items"]:
    v = item.get("ExpiresAt")
    if v is None:
        bad.append((item["SessionId"], "missing"))
    elif not isinstance(v, (int, float)) and not str(v).isdigit():
        bad.append((item["SessionId"], "not numeric"))
    elif int(v) > now + 10**10:
        bad.append((item["SessionId"], "milliseconds not seconds"))
print(bad[:20])
```

---

### Expired items still show up in queries

**Expected.** TTL deletion is asynchronous — expired items remain readable until the background process removes them.

**You must filter on read:**

```bash
NOW=$(date +%s)
aws dynamodb query --table-name Sessions \
  --key-condition-expression "SessionId = :s" \
  --filter-expression "ExpiresAt > :now" \
  --expression-attribute-values "{\":s\":{\"S\":\"S-1\"},\":now\":{\"N\":\"$NOW\"}}"
```

Skipping this filter in a session store is a genuine security bug — you'll honour expired sessions.

---

### TTL deleted items I wanted to keep

**Fixes going forward:**
- Remove the TTL attribute from items you want to keep permanently (items without it never expire).
- Archive on expiry: Streams (`OLD_IMAGE`) → Lambda → S3.

**Recovery:** restore from PITR to a point before the deletion.

---

### TTL deletions are slow or bursty

TTL runs as a background process with no SLA on exact timing and no capacity consumption. It's best-effort. If you need deterministic deletion timing, delete explicitly (and pay the WCU).

---

## 13. Global Tables Problems

### Can't create a replica

| Error | Cause | Fix |
|---|---|---|
| Streams not enabled | Global tables require `NEW_AND_OLD_IMAGES` | Enable the stream first |
| Table not empty (older versions) | v2017.11.29 required empty tables | Use version 2019.11.21 |
| Name/schema mismatch | Replicas must be identical | Same table name and key schema |
| Region not supported | Especially for MRSC | Check current Region support |

```bash
aws dynamodb update-table --table-name Orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

---

### High `ReplicationLatency` / growing `PendingReplicationCount`

**Causes:**
- Replica Region is under-provisioned — replicated writes consume **rWCU in every replica Region**.
- Sudden burst of writes in the source Region.
- Cross-Region network events.

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB --metric-name ReplicationLatency \
  --dimensions Name=TableName,Value=Orders \
              Name=ReceivingRegion,Value=eu-west-1 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average Maximum
```

**Fix:** increase capacity in the replica Regions (including their GSIs), and alarm on `ReplicationLatency`.

---

### Data lost — a write disappeared in a multi-Region setup

**Cause:** MREC uses **last-writer-wins** conflict resolution by timestamp. Concurrent writes to the same item in two Regions mean one silently wins.

**Fixes:**
- **Region-scoped partition keys** — route each entity's writes to one Region.
- **Application-level conflict resolution** — version vectors, or append-only items merged on read.
- **MRSC global tables** — synchronous replication, no conflicts. Requires exactly three Regions and must be chosen at creation.

---

### Can't switch a global table between MREC and MRSC

You can't. Consistency mode is fixed at creation. Migrating means creating a new global table and moving data (export/import or a dual-write cutover).

---

### Replica shows a different item count

Normal — replica metrics update independently and asynchronously. A persistent large gap alongside high `PendingReplicationCount` indicates a real replication problem.

---

### Can't delete a table that's part of a global table

Remove the replicas first:

```bash
aws dynamodb update-table --table-name Orders \
  --replica-updates '[{"Delete":{"RegionName":"eu-west-1"}}]'
# wait for it to complete, then delete the last one
```

---

## 14. Backup, Restore, Import & Export Problems

### `PointInTimeRecoveryUnavailableException`

PITR isn't enabled, or you're asking for a time outside the recovery window.

```bash
aws dynamodb describe-continuous-backups --table-name Orders \
  --query 'ContinuousBackupsDescription.PointInTimeRecoveryDescription'
```

That output gives `EarliestRestorableDateTime` and `LatestRestorableDateTime`. Your requested time must fall between them.

Also: PITR needs a few minutes after being enabled before any restore point exists.

---

### `ValidationException: Cannot restore to the same table name`

Restores **always create a new table**. You cannot restore over an existing one.

**Cutover pattern:**
1. Restore to `Orders-Restored`.
2. Verify the data.
3. Point the application at the new table (config change), **or** rename by: delete `Orders`, restore again to the name `Orders`.

Plan this before an incident — improvising it under pressure goes badly.

---

### Restored table is missing settings

**By design.** A restore does **not** carry over:

- ❌ Point-in-Time Recovery (must be re-enabled — easy to forget)
- ❌ TTL configuration
- ❌ Stream settings
- ❌ Auto Scaling policies
- ❌ CloudWatch alarms
- ❌ Resource-based policies
- ❌ Tags (unless you pass the flag)
- ✅ LSIs (always restored)
- ⚙️ GSIs (optional — skipping them is much faster)

**Post-restore checklist script:**

```bash
T=Orders-Restored
aws dynamodb update-continuous-backups --table-name $T \
  --point-in-time-recovery-specification "PointInTimeRecoveryEnabled=true,RecoveryPeriodInDays=14"
aws dynamodb update-time-to-live --table-name $T \
  --time-to-live-specification "Enabled=true,AttributeName=ExpiresAt"
aws dynamodb update-table --table-name $T \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
aws dynamodb update-table --table-name $T --deletion-protection-enabled
# re-register auto scaling and re-create alarms here
```

---

### Restore is taking hours

Expected for large tables. To speed it up, skip GSIs and rebuild them afterwards:

```bash
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Orders --target-table-name Orders-Restored \
  --use-latest-restorable-time \
  --global-secondary-index-override '[]'
```

---

### Export fails

| Error | Fix |
|---|---|
| PITR not enabled | Enable it — exports read from the continuous backup |
| S3 access denied | Bucket policy must allow `dynamodb.amazonaws.com` to `PutObject` |
| Bucket in another Region | Allowed, but slower and incurs transfer cost |
| Export time outside PITR window | Pick a time within the window |

```bash
aws dynamodb describe-export --export-arn <arn> \
  --query 'ExportDescription.{Status:ExportStatus,Failure:FailureMessage}'
```

---

### `TableAlreadyExistsException` on import

`import-table` **always creates a new table**. The target must not exist.

**Fix:** import to a temporary name, then migrate, or delete the existing table first.

---

### Import completed but rows are missing

```bash
aws dynamodb describe-import --import-arn <arn> \
  --query 'ImportTableDescription.{Processed:ProcessedItemCount,Errors:ErrorCount,Log:CloudWatchLogGroupArn}'
```

Malformed rows are skipped and logged to CloudWatch Logs. Common causes: missing key columns, wrong CSV delimiter, type mismatches (a non-numeric value in a Number key), duplicate keys (last one wins), or wrong compression setting.

---

## 15. DAX Problems

### Can't connect to the DAX cluster

DAX is **VPC-only**. You cannot reach it from your laptop or from a Lambda that isn't VPC-attached.

**Checklist:**

```bash
aws dax describe-clusters --cluster-names my-dax \
  --query 'Clusters[0].{Status:Status,Endpoint:ClusterDiscoveryEndpoint,SG:SecurityGroups}'
```

1. Client is in the **same VPC** (or peered/connected).
2. Security group allows **TCP 8111** from the client's security group.
3. Using the **cluster discovery endpoint**, not a node address.
4. Cluster status is `available`.
5. Using the **DAX client library** (`amazon-dax-client`), not plain boto3 pointed at the endpoint.
6. Lambda has VPC configuration with subnets and the right security group.

---

### DAX returns stale data

**Expected behaviour.**

- **Item cache**: updated on write-through, expires by `record-ttl-millis` (default 5 min).
- **Query cache**: **not invalidated by writes at all** — it only expires by `query-ttl-millis`.

So a Query result can be stale even after you've written through DAX.

**Fixes:**
- Lower `query-ttl-millis`.
- Use `ConsistentRead=True` for reads that must be current (bypasses DAX entirely).
- Route reads that need freshness directly to DynamoDB.

---

### DAX isn't improving latency

- **Cache misses** — check the `ItemCacheHits`/`ItemCacheMisses` CloudWatch metrics. Low hit rate means your access pattern isn't repetitive enough for caching to help.
- **Write-heavy workload** — DAX adds a hop on writes and caches nothing useful.
- **Cold cache** — measure after warm-up.
- **Using `ConsistentRead`** — that bypasses the cache.
- **Undersized nodes** — cache eviction due to memory pressure.

---

### DAX cluster cost is unexpectedly high

DAX bills **per node-hour, always on** — this is the one DynamoDB-adjacent service that costs money while idle. A 3-node cluster in a dev account left running for a month is a real bill.

```bash
aws dax describe-clusters --query 'Clusters[].{Name:ClusterName,Nodes:TotalNodes,Type:NodeType}'
aws dax delete-cluster --cluster-name my-dax
```

---

## 16. SDK & Client Problems

### `botocore.exceptions.NoRegionError: You must specify a region`

```bash
aws configure set region us-east-1
export AWS_DEFAULT_REGION=us-east-1
```

```python
boto3.resource("dynamodb", region_name="us-east-1")
```

---

### `NoCredentialsError` / `PartialCredentialsError`

Credentials aren't being found. The chain is: environment variables → shared credentials file → IAM role (EC2/ECS/Lambda).

```bash
aws sts get-caller-identity
env | grep AWS_
cat ~/.aws/credentials
```

In Lambda/ECS/EC2, use the execution role — never hard-code keys.

---

### `TypeError: Float types are not supported. Use Decimal instead` (boto3)

DynamoDB numbers map to Python `Decimal`, not `float`.

```python
from decimal import Decimal
import json

# Writing
item = {"Price": Decimal("149.99")}
# From arbitrary JSON:
item = json.loads(payload, parse_float=Decimal)

# Reading — convert back for JSON serialization
def to_native(o):
    if isinstance(o, Decimal):
        return int(o) if o % 1 == 0 else float(o)
    raise TypeError
json.dumps(items, default=to_native)
```

---

### `Inexact` / `Rounded` Decimal errors

```python
import decimal, boto3
ctx = decimal.Context(prec=38, traps=[])
# Or simply construct Decimals from strings, never from floats:
Decimal("0.1")     # ✅
Decimal(0.1)       # ❌ carries binary float imprecision
```

---

### Connection pool exhausted / `Connection pool is full` warnings

```python
from botocore.config import Config
cfg = Config(max_pool_connections=50,
             retries={"max_attempts": 10, "mode": "adaptive"},
             connect_timeout=1, read_timeout=3)
ddb = boto3.resource("dynamodb", config=cfg)
```

---

### Lambda cold starts are slow / high latency on first invoke

**Create the client outside the handler** so it's reused across invocations:

```python
import boto3
table = boto3.resource("dynamodb").Table("Orders")   # module scope ✅

def lambda_handler(event, context):
    return table.get_item(Key=...)                    # reuses the connection
```

For Node.js, set `AWS_NODEJS_CONNECTION_REUSE_ENABLED=1` (default in SDK v3).

---

### Requests hang and eventually time out

Default socket timeouts are long. Set aggressive ones for a low-latency database:

```python
Config(connect_timeout=1, read_timeout=3, retries={"max_attempts": 5, "mode": "adaptive"})
```

Also check: is the Lambda in a VPC without a DynamoDB endpoint or NAT gateway? That's the classic silent hang.

---

### `EndpointConnectionError: Could not connect to the endpoint URL`

- No internet route from the subnet (VPC Lambda without NAT or a gateway endpoint).
- Wrong Region in the endpoint hostname.
- A stale `endpoint_url` override pointing at DynamoDB Local.
- Corporate proxy interference — check `HTTP_PROXY`/`HTTPS_PROXY`.

---

## 17. CLI-Specific Problems

### `Invalid base64` errors on CLI v2

CLI v2 changed the default binary format.

```bash
aws configure set cli_binary_format raw-in-base64-out
# or per command
aws dynamodb put-item --cli-binary-format raw-in-base64-out ...
```

---

### JSON quoting nightmares in the shell

Use files instead of inline JSON:

```bash
cat > /tmp/item.json << 'EOF'
{"UserId": {"S": "U-1042"}, "Total": {"N": "149.99"}}
EOF
aws dynamodb put-item --table-name Orders --item file:///tmp/item.json
```

Note `<< 'EOF'` with quotes — this prevents the shell from expanding `$` inside the heredoc.

---

### `Parameter validation failed: Invalid type for parameter`

Almost always a missing type descriptor. The CLI needs low-level DynamoDB JSON:

```bash
# ❌
--item '{"UserId":"U-1042","Total":149.99}'

# ✅
--item '{"UserId":{"S":"U-1042"},"Total":{"N":"149.99"}}'
```

Note that Numbers are quoted strings in the wire format.

---

### The CLI opens a pager and hangs

```bash
aws dynamodb scan --table-name Orders --no-cli-pager
# or permanently
export AWS_PAGER=""
```

---

### Results are truncated

The CLI auto-paginates, but `--max-items` stops it early and returns a `NextToken`.

```bash
aws dynamodb scan --table-name Orders --page-size 100        # paginate internally
aws dynamodb scan --table-name Orders --starting-token <NextToken>
```

---

### `--query` returns null

JMESPath is case-sensitive and DynamoDB responses are nested with type descriptors:

```bash
# ❌
--query 'Item.Status'

# ✅
--query 'Item.Status.S'
```

---

## 18. DynamoDB Local Problems

### `Cannot do operations on a non-existent table` even though I created it

DynamoDB Local partitions tables by credentials and Region **unless** you use `-sharedDb`. If your create and query used different fake credentials, you're looking at two different databases.

```bash
docker run -p 8000:8000 amazon/dynamodb-local \
  -jar DynamoDBLocal.jar -sharedDb
```

---

### Data disappears on restart

DynamoDB Local is in-memory by default.

```bash
docker run -p 8000:8000 -v "$PWD/ddb-data:/home/dynamodblocal/data" \
  amazon/dynamodb-local -jar DynamoDBLocal.jar -sharedDb -dbPath ./data
```

---

### Behaviour differs from real DynamoDB

DynamoDB Local is an emulator. Known differences:
- **No real throttling** — capacity settings are accepted but not enforced meaningfully.
- **Immediate consistency everywhere** — GSIs update synchronously, so eventual-consistency bugs hide.
- **TTL isn't automatic** (in some versions) — items don't expire on their own.
- **No Streams-to-Lambda triggers**, no DAX, no global tables, no PITR.

**Consequence:** always run integration tests against a real table in a sandbox account before shipping.

---

### Credentials errors against local

It still requires *something*, even though it ignores the values:

```bash
export AWS_ACCESS_KEY_ID=local
export AWS_SECRET_ACCESS_KEY=local
export AWS_DEFAULT_REGION=us-east-1
```

---

## 19. Performance Problems

### High p99 latency

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB --metric-name SuccessfulRequestLatency \
  --dimensions Name=TableName,Value=Orders Name=Operation,Value=Query \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Average Maximum
```

| Cause | Fix |
|---|---|
| Large items | Reduce item size; project fewer attributes |
| Large result sets | Lower `Limit`, paginate |
| Scans in the request path | Add a GSI |
| Strongly consistent reads | Use eventual consistency where acceptable |
| Client in a different Region | Move the client, or use global tables |
| Cold Lambda + client created inside the handler | Move client creation to module scope |
| Retries from throttling | Fix capacity — retry latency shows up as p99 |
| No connection reuse | Enable keep-alive |
| Transactions | Inherently higher latency (two-phase) |

> **Important:** the `SuccessfulRequestLatency` metric measures *server-side* time only. If your application sees 200 ms and CloudWatch shows 5 ms, the problem is in your network, client, or retries — not DynamoDB.

---

### Throughput is capped no matter how much capacity I add

Almost certainly a hot partition. A single partition tops out around 3,000 RCU / 1,000 WCU regardless of table-level capacity.

Enable Contributor Insights and see [§2 Cause B](#cause-b--hot-partition-consumed-is-low-but-youre-still-throttled).

---

### Bulk load is slow

| Approach | Relative speed | Cost |
|---|---|---|
| Serial `PutItem` | Slowest | Highest |
| Threaded `BatchWriteItem` | Fast | Normal WCU |
| **`import-table` from S3** | **Fastest** | **No write capacity consumed** |

For an existing large table, `export-table-to-point-in-time` → `import-table` is the cheapest and fastest copy mechanism by a wide margin.

---

### Query performance degrades as the table grows

If a single **item collection** (one partition key) keeps growing, queries over it read more data.

**Fixes:**
- Add a time bucket to the partition key: `USER#1042#2026-08`.
- Use a more selective sort key condition.
- Archive old items to S3 with TTL + Streams.

---

## 20. Cost Surprises

### The bill jumped and I don't know why

**Diagnose:**

```bash
# Group cost by table (requires cost allocation tags to be activated)
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon DynamoDB"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE
```

**The usual suspects:**

| Symptom | Cause |
|---|---|
| Read charges dominate | A `Scan` got into the request path |
| Write charges dominate | Too many GSIs, or `ALL` projections |
| Storage dominates | Large items or no TTL — consider Standard-IA table class |
| Steady cost with no traffic | Provisioned capacity on an idle table, or a DAX cluster |
| Sudden 2× jump | Someone enabled a new GSI |
| Cross-Region transfer | Global table replication |
| Backup line item | PITR + many on-demand backups |

---

### On-demand is much more expensive than expected

On-demand costs roughly 5–7× provisioned per unit of throughput. It wins when traffic is spiky or idle; it loses badly on steady high load.

**Fix:** if utilization is consistently above ~40–50% of a stable peak, switch to provisioned with auto scaling, and buy Reserved Capacity for the baseline.

---

### GSIs are doubling my write bill

Every GSI that an item qualifies for is an additional write.

```
1 write to base table + 3 GSIs (all matching) = 4 WCU
```

**Fixes:**
- Delete unused GSIs (check `ConsumedReadCapacityUnits` per index — a GSI with zero reads is pure cost).
- Change `ALL` projections to `INCLUDE` with only the attributes you need.
- Use **sparse indexes** so most items never enter the index at all.

---

### Storage cost is high

- Compress large attributes (gzip + base64).
- Offload blobs to S3.
- Enable TTL to age out old data.
- Switch to the **Standard-IA table class** if storage is more than about half the bill.
- Shorten attribute names — they're stored on every item.

---

### A runaway job burned through my budget

**Prevention:**

```bash
# Hard ceiling on an on-demand table
aws dynamodb update-table --table-name Orders \
  --on-demand-throughput MaxReadRequestUnits=5000,MaxWriteRequestUnits=2000

# Budget alert
aws budgets create-budget --account-id $ACCOUNT --budget file://budget.json
```

Also: deny `dynamodb:Scan` in production IAM roles. It's the single highest-leverage cost control.

---

## 21. Data Type Gotchas

### `ValidationException: One or more parameter values are not valid. The AttributeValue for a key attribute cannot contain an empty string value`

Empty strings are allowed for **non-key** attributes but not for partition keys, sort keys, or index key attributes.

**Fix:** omit the attribute, use a sentinel value (`"NONE"`), or store `NULL`.

---

### Numbers sort in the wrong order

If the sort key is a **String**, sorting is lexicographic: `"10" < "2" < "9"`.

**Fixes:**
- Store as a **Number** type if you need numeric ordering and don't need composite keys.
- Zero-pad strings to a fixed width: `"0002"`, `"0010"`.
- For prices in composite keys: `PRICE#00008999`.

---

### Dates sort incorrectly

Use ISO-8601 (`2026-08-04T09:15:00Z`) — it's lexicographically sortable by design. Formats like `04/08/2026` or `Aug 4 2026` are not.

---

### Booleans stored as strings

`{"S":"true"}` is a string; `{"BOOL":true}` is a boolean. Filters like `attribute_type(Paid, :b)` and comparisons behave differently. Be consistent.

---

### Set vs List confusion

| | Set (`SS`/`NS`/`BS`) | List (`L`) |
|---|---|---|
| Order preserved | ❌ | ✅ |
| Duplicates allowed | ❌ | ✅ |
| Mixed types | ❌ | ✅ |
| Can be empty | ❌ | ✅ |
| Atomic add/remove | ✅ (`ADD`/`DELETE`) | Partial (`list_append`, indexed `REMOVE`) |

---

### Nested map updates fail

You cannot create intermediate levels implicitly:

```bash
# ❌ fails if Address doesn't exist
--update-expression "SET Address.City = :c"

# ✅ create the map first, or set the whole map
--update-expression "SET Address = if_not_exists(Address, :empty)"
--expression-attribute-values '{":empty":{"M":{}}}'
# then a second update for the field
```

---

### Binary attributes come back as base64

Expected — binary is base64-encoded on the wire. The SDKs decode it for you; the CLI does not.

```bash
aws dynamodb get-item ... --query 'Item.Blob.B' --output text | base64 -d
```

---

## 22. Diagnostic Command Toolkit

### Full table health check

```bash
#!/usr/bin/env bash
T=${1:?usage: healthcheck.sh <table-name>}

echo "=== CONFIGURATION ==="
aws dynamodb describe-table --table-name "$T" --query 'Table.{
  Status:TableStatus, Billing:BillingModeSummary.BillingMode,
  Items:ItemCount, SizeBytes:TableSizeBytes,
  Encryption:SSEDescription.Status,
  DeletionProtection:DeletionProtectionEnabled,
  Class:TableClassSummary.TableClass,
  Stream:StreamSpecification, Warm:WarmThroughput}'

echo "=== INDEXES ==="
aws dynamodb describe-table --table-name "$T" --query 'Table.GlobalSecondaryIndexes[].{
  Name:IndexName, Status:IndexStatus, Backfilling:Backfilling,
  Projection:Projection.ProjectionType, Items:ItemCount}' --output table

echo "=== PITR ==="
aws dynamodb describe-continuous-backups --table-name "$T" \
  --query 'ContinuousBackupsDescription.PointInTimeRecoveryDescription'

echo "=== TTL ==="
aws dynamodb describe-time-to-live --table-name "$T" --query 'TimeToLiveDescription'

echo "=== REPLICAS ==="
aws dynamodb describe-table --table-name "$T" \
  --query 'Table.Replicas[].{Region:RegionName,Status:ReplicaStatus}' --output table

echo "=== THROTTLES (24h) ==="
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB --metric-name ThrottledRequests \
  --dimensions Name=TableName,Value="$T" \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 3600 --statistics Sum \
  --query 'Datapoints[?Sum>`0`].{Time:Timestamp,Count:Sum}' --output table

echo "=== ERRORS (24h) ==="
for M in SystemErrors UserErrors ConditionalCheckFailedRequests; do
  V=$(aws cloudwatch get-metric-statistics \
        --namespace AWS/DynamoDB --metric-name $M \
        --dimensions Name=TableName,Value="$T" \
        --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
        --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
        --period 86400 --statistics Sum \
        --query 'Datapoints[0].Sum' --output text)
  echo "  $M: ${V:-0}"
done
```

### Find the real cost of a query

```bash
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u" \
  --expression-attribute-values '{":u":{"S":"U-1042"}}' \
  --return-consumed-capacity INDEXES \
  --query '{Returned:Count, Scanned:ScannedCount,
            TableRCU:ConsumedCapacity.Table.CapacityUnits,
            IndexRCU:ConsumedCapacity.GlobalSecondaryIndexes}'
```

### Audit every table for safety settings

```bash
for t in $(aws dynamodb list-tables --query 'TableNames[]' --output text); do
  pitr=$(aws dynamodb describe-continuous-backups --table-name "$t" \
    --query 'ContinuousBackupsDescription.PointInTimeRecoveryDescription.PointInTimeRecoveryStatus' \
    --output text 2>/dev/null)
  prot=$(aws dynamodb describe-table --table-name "$t" \
    --query 'Table.DeletionProtectionEnabled' --output text)
  printf "%-40s PITR=%-10s DeleteProtection=%s\n" "$t" "${pitr:-?}" "$prot"
done
```

### Trace a single item's history via PITR

```bash
for H in 01 06 12 18; do
  aws dynamodb restore-table-to-point-in-time \
    --source-table-name Orders \
    --target-table-name "Orders-T$H" \
    --restore-date-time "2026-08-04T${H}:00:00Z" \
    --global-secondary-index-override '[]'
done
# then GetItem the same key from each and diff
```

---

## 23. Incident Runbooks

### 🚨 Runbook: Table is throttling in production

```
1. CONFIRM
   aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB \
     --metric-name ThrottledRequests --dimensions Name=TableName,Value=<T> ...

2. LOCALIZE — table or index?
   Check the GlobalSecondaryIndexName dimension too.

3. CLASSIFY
   Consumed ≈ Provisioned?  → under-provisioned → go to 4a
   Consumed << Provisioned? → hot partition     → go to 4b

4a. IMMEDIATE (under-provisioned)
   aws dynamodb update-table --table-name <T> --billing-mode PAY_PER_REQUEST
   (or raise provisioned capacity 2-3x)

4b. IMMEDIATE (hot partition)
   Enable Contributor Insights, identify the hot key.
   Short term: cache that key in the app, or rate-limit the offending caller.
   Medium term: write sharding or a key redesign.

5. VERIFY
   Watch ThrottledRequests return to zero. Confirm client error rates drop.

6. FOLLOW UP
   - Add/adjust auto scaling on the table AND every GSI
   - Confirm clients use retries mode=adaptive
   - Add a ThrottledRequests alarm if it was missing
   - Post-incident: was this a capacity issue or a modeling issue?
```

---

### 🚨 Runbook: Accidental data deletion

```
1. STOP THE BLEEDING
   Disable the job/deploy that's deleting. Revoke its IAM permissions if needed:
     aws iam put-role-policy --role-name <R> --policy-name EmergencyDeny \
       --policy-document '{"Version":"2012-10-17","Statement":[{
         "Effect":"Deny","Action":["dynamodb:DeleteItem","dynamodb:BatchWriteItem"],
         "Resource":"*"}]}'

2. ESTABLISH THE TIMELINE
   When did deletion start? CloudTrail / application logs / stream records.

3. CHECK RECOVERY OPTIONS
   aws dynamodb describe-continuous-backups --table-name <T>
   aws dynamodb list-backups --table-name <T>

4. RESTORE TO A SIDE TABLE (never over production)
   aws dynamodb restore-table-to-point-in-time \
     --source-table-name <T> --target-table-name <T>-Recovery \
     --restore-date-time <before-the-incident>

5. VERIFY the recovered data before touching production.

6. MERGE OR CUT OVER
   Merge: script the missing items back with conditional writes
          (attribute_not_exists) so you don't clobber newer data.
   Cutover: point the app at <T>-Recovery, then rename.

7. RE-APPLY LOST SETTINGS on the restored table:
   PITR, TTL, streams, auto scaling, alarms, deletion protection, tags.

8. PREVENT
   - Deletion protection ON
   - Deny bulk delete actions in production roles
   - Require a second approval for destructive jobs
```

---

### 🚨 Runbook: Stream consumer is falling behind

```
1. MEASURE
   CloudWatch → Lambda → IteratorAge for the function.
   Approaching 24 hours = imminent permanent data loss.

2. IMMEDIATE RELIEF
   aws lambda update-event-source-mapping --uuid <uuid> \
     --parallelization-factor 10 --batch-size 500

   Increase Lambda memory (more CPU):
   aws lambda update-function-configuration --function-name <F> --memory-size 1024

3. CHECK FOR A POISON PILL
   aws logs filter-log-events --log-group-name /aws/lambda/<F> \
     --filter-pattern "ERROR" --start-time <ms>

   If one record keeps failing:
     aws lambda update-event-source-mapping --uuid <uuid> \
       --bisect-batch-on-function-error --maximum-retry-attempts 3 \
       --destination-config '{"OnFailure":{"Destination":"<dlq-arn>"}}'

4. IF STILL BEHIND
   Simplify the handler: write raw records to SQS/S3 and process asynchronously.

5. FOLLOW UP
   - Alarm on IteratorAge > 1 hour
   - Make the consumer idempotent
   - Load-test the consumer at 2x peak write rate
```

---

### 🚨 Runbook: Unexpected DynamoDB bill

```
1. IDENTIFY THE DIMENSION
   Cost Explorer → filter Amazon DynamoDB → group by USAGE_TYPE
   ReadCapacityUnit-Hrs / WriteCapacityUnit-Hrs / TimedStorage / etc.

2. IDENTIFY THE TABLE
   Group by the TableName cost allocation tag (activate it if you haven't).

3. COMMON ROOT CAUSES
   Reads high    → a Scan reached production. Search the code for scan(.
   Writes high   → new GSI, or ALL projections, or a retry storm
   Storage high  → no TTL, large items, wrong table class
   Flat baseline → idle provisioned table, or a forgotten DAX cluster

4. IMMEDIATE CONTROLS
   aws dynamodb update-table --table-name <T> \
     --on-demand-throughput MaxReadRequestUnits=<n>,MaxWriteRequestUnits=<n>
   Add "Deny dynamodb:Scan" to production roles.

5. STRUCTURAL FIX
   Replace the Scan with a GSI query. Trim projections. Enable TTL.
   Consider Standard-IA. Buy Reserved Capacity for stable baselines.
```

---

## Still Stuck?

**Collect this before asking for help** — it turns a vague question into a solvable one:

```bash
# 1. Exact error (with request ID if available)
# 2. Table configuration
aws dynamodb describe-table --table-name <T> > table-config.json
# 3. Recent metrics
aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB \
  --metric-name ThrottledRequests --dimensions Name=TableName,Value=<T> \
  --start-time $(date -u -d '6 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 300 --statistics Sum
# 4. The exact request that fails (with --return-consumed-capacity INDEXES)
# 5. SDK/CLI version and Region
aws --version
```

**Where to go:**
- [AWS re:Post — DynamoDB](https://repost.aws/tags/TAQeK5mgAsSIS_LcDQGSN0Bg)
- [DynamoDB Developer Guide — Troubleshooting](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)
- [AWS Support](https://console.aws.amazon.com/support/) (Business tier and above for technical support)
- [AWS Health Dashboard](https://health.aws.amazon.com/health/status) — rule out a service event first

---

➡️ **[README.md](./README.md)** — concepts and architecture
➡️ **[commands-cheatsheet.md](./commands-cheatsheet.md)** — command reference
➡️ **[hands-on-labs.md](./hands-on-labs.md)** — practice labs
