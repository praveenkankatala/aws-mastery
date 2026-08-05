# Amazon DynamoDB — Hands-On Labs

> Twelve labs that build on each other. Start at Lab 1 with an empty account and finish with a deployed serverless API on a single-table design. Every command is runnable as written.

---

## Before You Start

### Requirements

```bash
aws --version                # aws-cli/2.x
python3 --version            # 3.9+
pip install boto3
aws sts get-caller-identity  # confirm credentials
```

### Set your working variables

```bash
export REGION=us-east-1
export ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
aws configure set region $REGION
aws configure set cli_binary_format raw-in-base64-out
```

### Cost & cleanup

Every lab uses **on-demand billing** and tiny datasets, so total cost is pennies — likely $0 inside the Free Tier. **Lab 9 (DAX) is the exception**: DAX nodes bill per hour whether you use them or not. Each lab ends with a **🧹 Cleanup** section. Do them.

### Lab index

| # | Lab | Time | Teaches |
|---|---|---|---|
| 1 | [Your First Table & CRUD](#lab-1--your-first-table--crud) | 20 min | Tables, items, GetItem, data types |
| 2 | [Composite Keys & Query](#lab-2--composite-keys--query) | 25 min | Sort keys, Query vs Scan, cost proof |
| 3 | [Expressions & Conditional Writes](#lab-3--expressions--conditional-writes) | 30 min | Filters, conditions, optimistic locking |
| 4 | [Secondary Indexes](#lab-4--secondary-indexes-gsi--lsi) | 35 min | GSI, LSI, projections, sparse indexes |
| 5 | [Capacity, Throttling & Auto Scaling](#lab-5--capacity-throttling--auto-scaling) | 40 min | RCU/WCU, deliberate throttling, scaling |
| 6 | [Transactions](#lab-6--transactions) | 30 min | ACID, ConditionCheck, unique constraints |
| 7 | [Streams + Lambda](#lab-7--streams--lambda) | 45 min | Event-driven architecture, CDC |
| 8 | [TTL & Session Store](#lab-8--ttl--session-store) | 25 min | Automatic expiry, session patterns |
| 9 | [DAX Caching](#lab-9--dax-caching-optional-costs-money) | 40 min | Microsecond reads, write-through cache |
| 10 | [Single-Table Design](#lab-10--single-table-design) | 60 min | Real modeling, GSI overloading |
| 11 | [Backup, Restore & Export](#lab-11--backup-restore--export) | 35 min | PITR, disaster recovery, S3 export |
| 12 | [Deploy a Serverless API](#lab-12--deploy-a-serverless-api) | 60 min | End-to-end production deployment |

---

## Lab 1 — Your First Table & CRUD

**Goal:** create a table, write items, read them back, and understand what a DynamoDB item actually looks like on the wire.

### 1.1 Create the table

```bash
aws dynamodb create-table \
  --table-name Lab-Products \
  --attribute-definitions AttributeName=Sku,AttributeType=S \
  --key-schema AttributeName=Sku,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --tags Key=Lab,Value=1

aws dynamodb wait table-exists --table-name Lab-Products
echo "Table is ACTIVE"
```

### 1.2 Inspect what you built

```bash
aws dynamodb describe-table --table-name Lab-Products \
  --query 'Table.{Name:TableName,Status:TableStatus,Keys:KeySchema,Billing:BillingModeSummary.BillingMode}'
```

**Notice:** the table has no columns. Only `Sku` is defined, because only key attributes need declaring.

### 1.3 Write items with every data type

```bash
aws dynamodb put-item --table-name Lab-Products --item '{
  "Sku":         {"S": "SKU-1001"},
  "Name":        {"S": "Mechanical Keyboard"},
  "Price":       {"N": "8999"},
  "InStock":     {"BOOL": true},
  "Tags":        {"SS": ["electronics","input","rgb"]},
  "Ratings":     {"NS": ["5","4","5","3"]},
  "Dimensions":  {"M": {"Length":{"N":"35"},"Width":{"N":"13"},"Unit":{"S":"cm"}}},
  "Images":      {"L": [{"S":"img1.jpg"},{"S":"img2.jpg"}]},
  "Discontinued":{"NULL": true}
}'
```

Add two more:

```bash
aws dynamodb put-item --table-name Lab-Products --item '{
  "Sku":{"S":"SKU-1002"},"Name":{"S":"Wireless Mouse"},
  "Price":{"N":"2499"},"InStock":{"BOOL":true},
  "Tags":{"SS":["electronics","input"]}}'

aws dynamodb put-item --table-name Lab-Products --item '{
  "Sku":{"S":"SKU-1003"},"Name":{"S":"USB-C Hub"},
  "Price":{"N":"3499"},"InStock":{"BOOL":false},
  "Tags":{"SS":["electronics","accessory"]}}'
```

### 1.4 Read them back

```bash
# Full item
aws dynamodb get-item --table-name Lab-Products \
  --key '{"Sku":{"S":"SKU-1001"}}'

# Strongly consistent, with the cost shown
aws dynamodb get-item --table-name Lab-Products \
  --key '{"Sku":{"S":"SKU-1001"}}' \
  --consistent-read --return-consumed-capacity TOTAL \
  --query '{Item:Item.Name.S,RCU:ConsumedCapacity.CapacityUnits}'
```

**Compare the RCU** with and without `--consistent-read`. You should see `1.0` vs `0.5`.

### 1.5 Projections and nested paths

```bash
aws dynamodb get-item --table-name Lab-Products \
  --key '{"Sku":{"S":"SKU-1001"}}' \
  --projection-expression "#n, Price, Dimensions.Length" \
  --expression-attribute-names '{"#n":"Name"}'
```

**Why `#n`?** `Name` is a reserved word. Try it without the alias and read the error — it's a rite of passage.

### 1.6 The overwrite trap

```bash
# This DELETES every attribute except the two you supply
aws dynamodb put-item --table-name Lab-Products \
  --item '{"Sku":{"S":"SKU-1002"},"Name":{"S":"Wireless Mouse v2"}}'

aws dynamodb get-item --table-name Lab-Products --key '{"Sku":{"S":"SKU-1002"}}'
```

`Price`, `InStock`, and `Tags` are gone. **PutItem replaces the whole item.** Use `UpdateItem` (Lab 3) when you want to change one attribute.

### 1.7 Delete

```bash
aws dynamodb delete-item --table-name Lab-Products \
  --key '{"Sku":{"S":"SKU-1003"}}' --return-values ALL_OLD

# Deleting something that doesn't exist succeeds silently
aws dynamodb delete-item --table-name Lab-Products --key '{"Sku":{"S":"NOPE"}}'
echo "Exit code: $?"     # 0
```

### ✅ Checkpoint

You should be able to explain: why only `Sku` was declared; the difference between `S` and `N`; why `PutItem` deleted attributes; why strongly consistent reads cost double.

### 🧹 Cleanup

```bash
aws dynamodb delete-table --table-name Lab-Products
```

---

## Lab 2 — Composite Keys & Query

**Goal:** understand item collections, and see with your own eyes why Query beats Scan.

### 2.1 Create a table with a sort key

```bash
aws dynamodb create-table \
  --table-name Lab-Orders \
  --attribute-definitions \
      AttributeName=CustomerId,AttributeType=S \
      AttributeName=OrderDate,AttributeType=S \
  --key-schema \
      AttributeName=CustomerId,KeyType=HASH \
      AttributeName=OrderDate,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --tags Key=Lab,Value=2

aws dynamodb wait table-exists --table-name Lab-Orders
```

### 2.2 Load sample data

```bash
cat > /tmp/seed-orders.py << 'EOF'
import boto3, random, datetime
t = boto3.resource("dynamodb").Table("Lab-Orders")
statuses = ["PENDING", "SHIPPED", "DELIVERED", "CANCELLED"]
with t.batch_writer() as b:
    for c in range(1, 21):                       # 20 customers
        for d in range(1, 26):                   # 25 orders each = 500 items
            date = datetime.date(2026, random.randint(1, 8), random.randint(1, 28))
            b.put_item(Item={
                "CustomerId": f"CUST-{c:04d}",
                "OrderDate":  f"{date.isoformat()}#{d:03d}",
                "Status":     random.choice(statuses),
                "Total":      round(random.uniform(10, 999), 2),
                "Region":     random.choice(["APAC", "EMEA", "AMER"]),
            })
print("Loaded 500 orders")
EOF
python3 /tmp/seed-orders.py
```

### 2.3 Query one customer's item collection

```bash
aws dynamodb query --table-name Lab-Orders \
  --key-condition-expression "CustomerId = :c" \
  --expression-attribute-values '{":c":{"S":"CUST-0001"}}' \
  --return-consumed-capacity TOTAL \
  --query '{Count:Count,RCU:ConsumedCapacity.CapacityUnits}'
```

### 2.4 Sort key range conditions

```bash
# Orders from a specific month
aws dynamodb query --table-name Lab-Orders \
  --key-condition-expression "CustomerId = :c AND begins_with(OrderDate, :m)" \
  --expression-attribute-values '{":c":{"S":"CUST-0001"},":m":{"S":"2026-03"}}' \
  --query 'Count'

# A date range
aws dynamodb query --table-name Lab-Orders \
  --key-condition-expression "CustomerId = :c AND OrderDate BETWEEN :a AND :b" \
  --expression-attribute-values '{
      ":c":{"S":"CUST-0001"},":a":{"S":"2026-01"},":b":{"S":"2026-04"}}' \
  --query 'Count'

# Most recent 5
aws dynamodb query --table-name Lab-Orders \
  --key-condition-expression "CustomerId = :c" \
  --expression-attribute-values '{":c":{"S":"CUST-0001"}}' \
  --no-scan-index-forward --limit 5 \
  --query 'Items[].OrderDate.S'
```

### 2.5 The experiment that makes it click

Both of these return "CUST-0001's PENDING orders". Compare the cost.

```bash
echo "=== QUERY (targets one partition) ==="
aws dynamodb query --table-name Lab-Orders \
  --key-condition-expression "CustomerId = :c" \
  --filter-expression "#s = :st" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":c":{"S":"CUST-0001"},":st":{"S":"PENDING"}}' \
  --return-consumed-capacity TOTAL \
  --query '{Returned:Count,Scanned:ScannedCount,RCU:ConsumedCapacity.CapacityUnits}'

echo "=== SCAN (reads the whole table) ==="
aws dynamodb scan --table-name Lab-Orders \
  --filter-expression "CustomerId = :c AND #s = :st" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":c":{"S":"CUST-0001"},":st":{"S":"PENDING"}}' \
  --return-consumed-capacity TOTAL \
  --query '{Returned:Count,Scanned:ScannedCount,RCU:ConsumedCapacity.CapacityUnits}'
```

**Read the `Scanned` field.** The Query scanned ~25 items. The Scan scanned 500. Same answer, 20× the cost. At 5 million items that ratio is the difference between a working product and an outage.

### 2.6 Prove that filters don't save money

```bash
# No filter
aws dynamodb query --table-name Lab-Orders \
  --key-condition-expression "CustomerId = :c" \
  --expression-attribute-values '{":c":{"S":"CUST-0002"}}' \
  --return-consumed-capacity TOTAL --query 'ConsumedCapacity.CapacityUnits'

# With a filter that removes most results
aws dynamodb query --table-name Lab-Orders \
  --key-condition-expression "CustomerId = :c" \
  --filter-expression "Total > :t" \
  --expression-attribute-values '{":c":{"S":"CUST-0002"},":t":{"N":"900"}}' \
  --return-consumed-capacity TOTAL --query 'ConsumedCapacity.CapacityUnits'
```

**Identical RCU.** The filter runs after DynamoDB has already read — and billed you for — the data.

### 2.7 Pagination in practice

```bash
cat > /tmp/paginate.py << 'EOF'
import boto3
from boto3.dynamodb.conditions import Key
t = boto3.resource("dynamodb").Table("Lab-Orders")

kwargs = {"KeyConditionExpression": Key("CustomerId").eq("CUST-0003"), "Limit": 7}
page, total = 0, 0
while True:
    r = t.query(**kwargs)
    page += 1; total += len(r["Items"])
    print(f"page {page}: {len(r['Items'])} items")
    if "LastEvaluatedKey" not in r:
        break
    kwargs["ExclusiveStartKey"] = r["LastEvaluatedKey"]
print(f"total: {total} across {page} pages")
EOF
python3 /tmp/paginate.py
```

### ✅ Checkpoint

Explain: what an item collection is; why `begins_with` works on the sort key but not the partition key; why `ScannedCount` matters more than `Count` for cost.

### 🧹 Cleanup

Keep `Lab-Orders` — Labs 3, 4, and 5 use it.

---

## Lab 3 — Expressions & Conditional Writes

**Goal:** master UpdateItem and use conditions to build safe concurrent logic.

*Uses `Lab-Orders` from Lab 2.*

### 3.1 Surgical updates

```bash
aws dynamodb put-item --table-name Lab-Orders --item '{
  "CustomerId":{"S":"CUST-9999"},"OrderDate":{"S":"2026-08-04#001"},
  "Status":{"S":"PENDING"},"Total":{"N":"250.00"},
  "Version":{"N":"1"},"Tags":{"SS":["new"]}}'

# Change only Status — everything else survives
aws dynamodb update-item --table-name Lab-Orders \
  --key '{"CustomerId":{"S":"CUST-9999"},"OrderDate":{"S":"2026-08-04#001"}}' \
  --update-expression "SET #s = :st" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":st":{"S":"CONFIRMED"}}' \
  --return-values ALL_NEW
```

### 3.2 All four clauses at once

```bash
aws dynamodb update-item --table-name Lab-Orders \
  --key '{"CustomerId":{"S":"CUST-9999"},"OrderDate":{"S":"2026-08-04#001"}}' \
  --update-expression "SET Total = Total + :fee, CreatedAt = if_not_exists(CreatedAt, :now) ADD Revisions :one, Tags :t REMOVE OldField" \
  --expression-attribute-values '{
      ":fee":{"N":"15"},":now":{"S":"2026-08-04T10:00:00Z"},
      ":one":{"N":"1"},":t":{"SS":["priority"]}}' \
  --return-values ALL_NEW
```

Run it **twice**. Watch: `Total` increases each time, `Revisions` increments, but `CreatedAt` stays at its first value thanks to `if_not_exists`.

### 3.3 Atomic counters under concurrency

```bash
cat > /tmp/counter.py << 'EOF'
import boto3, concurrent.futures
t = boto3.resource("dynamodb").Table("Lab-Orders")
key = {"CustomerId": "CUST-9999", "OrderDate": "2026-08-04#001"}

def bump(_):
    t.update_item(Key=key,
                  UpdateExpression="ADD ViewCount :one",
                  ExpressionAttributeValues={":one": 1})

with concurrent.futures.ThreadPoolExecutor(max_workers=20) as ex:
    list(ex.map(bump, range(200)))

print("ViewCount =", t.get_item(Key=key, ConsistentRead=True)["Item"]["ViewCount"])
EOF
python3 /tmp/counter.py
```

**Expect exactly 200.** No reads, no locks, no lost updates. A read-modify-write loop would have lost dozens.

### 3.4 Conditional writes — a state machine

```bash
# Allowed: CONFIRMED → SHIPPED
aws dynamodb update-item --table-name Lab-Orders \
  --key '{"CustomerId":{"S":"CUST-9999"},"OrderDate":{"S":"2026-08-04#001"}}' \
  --update-expression "SET #s = :new" \
  --condition-expression "#s = :expected" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":new":{"S":"SHIPPED"},":expected":{"S":"CONFIRMED"}}'

# Blocked: you can't confirm an order that's already shipped
aws dynamodb update-item --table-name Lab-Orders \
  --key '{"CustomerId":{"S":"CUST-9999"},"OrderDate":{"S":"2026-08-04#001"}}' \
  --update-expression "SET #s = :new" \
  --condition-expression "#s = :expected" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":new":{"S":"CONFIRMED"},":expected":{"S":"PENDING"}}' \
  --return-values-on-condition-check-failure ALL_OLD
```

The second fails with `ConditionalCheckFailedException` **and returns the offending item** — you learn *why* it failed without a second read.

### 3.5 Optimistic locking

```bash
cat > /tmp/optimistic.py << 'EOF'
import boto3
from botocore.exceptions import ClientError
t = boto3.resource("dynamodb").Table("Lab-Orders")
key = {"CustomerId": "CUST-9999", "OrderDate": "2026-08-04#001"}

def update_with_lock(new_total, retries=3):
    for attempt in range(retries):
        item = t.get_item(Key=key, ConsistentRead=True)["Item"]
        ver = item["Version"]
        try:
            t.update_item(
                Key=key,
                UpdateExpression="SET Total = :t, Version = Version + :one",
                ConditionExpression="Version = :ver",
                ExpressionAttributeValues={":t": new_total, ":one": 1, ":ver": ver},
            )
            print(f"OK on attempt {attempt+1} (was version {ver})")
            return
        except ClientError as e:
            if e.response["Error"]["Code"] != "ConditionalCheckFailedException":
                raise
            print(f"conflict on attempt {attempt+1}, retrying")
    raise RuntimeError("gave up after retries")

update_with_lock(500)
EOF
python3 /tmp/optimistic.py
```

### 3.6 Insert-only writes

```bash
# Succeeds — new key
aws dynamodb put-item --table-name Lab-Orders \
  --item '{"CustomerId":{"S":"CUST-8888"},"OrderDate":{"S":"2026-08-04#001"},"Total":{"N":"10"}}' \
  --condition-expression "attribute_not_exists(CustomerId)"

# Fails — would overwrite
aws dynamodb put-item --table-name Lab-Orders \
  --item '{"CustomerId":{"S":"CUST-8888"},"OrderDate":{"S":"2026-08-04#001"},"Total":{"N":"20"}}' \
  --condition-expression "attribute_not_exists(CustomerId)"
```

> 💡 On a composite key, `attribute_not_exists(CustomerId)` is evaluated against the **specific item** identified by PK+SK — it means "this exact item doesn't exist", not "this partition key is unused."

### ✅ Checkpoint

You should be able to write an update that increments a counter, sets a default only once, removes a field, and refuses to run if the item is in the wrong state — all in one call.

### 🧹 Cleanup

Keep `Lab-Orders`.

---

## Lab 4 — Secondary Indexes (GSI & LSI)

**Goal:** add indexes to answer queries the base key can't, and see the sparse-index trick.

### 4.1 Add a GSI to the existing table

```bash
aws dynamodb update-table --table-name Lab-Orders \
  --attribute-definitions \
      AttributeName=Status,AttributeType=S \
      AttributeName=OrderDate,AttributeType=S \
  --global-secondary-index-updates '[{
    "Create": {
      "IndexName": "StatusIndex",
      "KeySchema": [
        {"AttributeName":"Status","KeyType":"HASH"},
        {"AttributeName":"OrderDate","KeyType":"RANGE"}],
      "Projection": {"ProjectionType":"INCLUDE","NonKeyAttributes":["CustomerId","Total"]}
    }}]'
```

### 4.2 Watch the backfill

```bash
watch -n 5 "aws dynamodb describe-table --table-name Lab-Orders \
  --query 'Table.GlobalSecondaryIndexes[].{Name:IndexName,Status:IndexStatus,Backfilling:Backfilling}' \
  --output table"
```

Press Ctrl-C when `IndexStatus` reads `ACTIVE`. The table stayed fully available the whole time.

### 4.3 Query the GSI

```bash
# All SHIPPED orders across every customer
aws dynamodb query --table-name Lab-Orders \
  --index-name StatusIndex \
  --key-condition-expression "#s = :st" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":st":{"S":"SHIPPED"}}' \
  --return-consumed-capacity INDEXES \
  --query '{Count:Count,RCU:ConsumedCapacity.CapacityUnits}'
```

This was **impossible** on the base table without a full Scan.

### 4.4 Confirm GSIs reject strong reads

```bash
aws dynamodb query --table-name Lab-Orders \
  --index-name StatusIndex --consistent-read \
  --key-condition-expression "#s = :st" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":st":{"S":"SHIPPED"}}'
```

Expected: `ValidationException: Consistent reads are not supported on global secondary indexes`. This is a hard architectural rule, not a setting.

### 4.5 See what the projection left out

```bash
aws dynamodb query --table-name Lab-Orders \
  --index-name StatusIndex \
  --key-condition-expression "#s = :st" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":st":{"S":"SHIPPED"}}' \
  --limit 1 --query 'Items[0]'
```

`Region` is missing — it wasn't in `NonKeyAttributes`. To get it you'd need a second `GetItem` against the base table. **This is the projection trade-off, made visible.**

### 4.6 Build a sparse index

```bash
# 1. Create an index on a flag attribute most items don't have
aws dynamodb update-table --table-name Lab-Orders \
  --attribute-definitions AttributeName=NeedsReview,AttributeType=S \
  --global-secondary-index-updates '[{
    "Create": {
      "IndexName": "ReviewQueue",
      "KeySchema": [{"AttributeName":"NeedsReview","KeyType":"HASH"}],
      "Projection": {"ProjectionType":"KEYS_ONLY"}
    }}]'

aws dynamodb wait table-exists --table-name Lab-Orders
sleep 30
```

```bash
# 2. Flag three items
for i in 0001 0002 0003; do
  DATE=$(aws dynamodb query --table-name Lab-Orders \
    --key-condition-expression "CustomerId = :c" \
    --expression-attribute-values "{\":c\":{\"S\":\"CUST-$i\"}}" \
    --limit 1 --query 'Items[0].OrderDate.S' --output text)
  aws dynamodb update-item --table-name Lab-Orders \
    --key "{\"CustomerId\":{\"S\":\"CUST-$i\"},\"OrderDate\":{\"S\":\"$DATE\"}}" \
    --update-expression "SET NeedsReview = :y" \
    --expression-attribute-values '{":y":{"S":"YES"}}'
done

# 3. The review queue — 3 items out of 500+, nearly free to read
aws dynamodb query --table-name Lab-Orders --index-name ReviewQueue \
  --key-condition-expression "NeedsReview = :y" \
  --expression-attribute-values '{":y":{"S":"YES"}}' \
  --return-consumed-capacity INDEXES \
  --query '{Count:Count,RCU:ConsumedCapacity.CapacityUnits}'
```

```bash
# 4. Clear a flag and watch the item leave the index
aws dynamodb update-item --table-name Lab-Orders \
  --key "{\"CustomerId\":{\"S\":\"CUST-0001\"},\"OrderDate\":{\"S\":\"$(aws dynamodb query --table-name Lab-Orders --key-condition-expression 'CustomerId = :c' --expression-attribute-values '{":c":{"S":"CUST-0001"}}' --limit 1 --query 'Items[0].OrderDate.S' --output text)\"}}" \
  --update-expression "REMOVE NeedsReview"

sleep 3
aws dynamodb query --table-name Lab-Orders --index-name ReviewQueue \
  --key-condition-expression "NeedsReview = :y" \
  --expression-attribute-values '{":y":{"S":"YES"}}' --query 'Count'
```

Now 2. **Removing the attribute removed the index entry.** That's how you build a work queue with zero scanning.

### 4.7 LSI — must be created with the table

```bash
aws dynamodb create-table \
  --table-name Lab-OrdersLSI \
  --attribute-definitions \
      AttributeName=CustomerId,AttributeType=S \
      AttributeName=OrderDate,AttributeType=S \
      AttributeName=Total,AttributeType=N \
  --key-schema \
      AttributeName=CustomerId,KeyType=HASH \
      AttributeName=OrderDate,KeyType=RANGE \
  --local-secondary-indexes '[{
      "IndexName":"TotalIndex",
      "KeySchema":[
        {"AttributeName":"CustomerId","KeyType":"HASH"},
        {"AttributeName":"Total","KeyType":"RANGE"}],
      "Projection":{"ProjectionType":"ALL"}}]' \
  --billing-mode PAY_PER_REQUEST

aws dynamodb wait table-exists --table-name Lab-OrdersLSI

# Load a few items
for i in 1 2 3 4 5; do
  aws dynamodb put-item --table-name Lab-OrdersLSI --item \
    "{\"CustomerId\":{\"S\":\"C-1\"},\"OrderDate\":{\"S\":\"2026-08-0$i\"},\"Total\":{\"N\":\"$((i*137))\"}}"
done

# LSI supports strongly consistent reads — unlike a GSI
aws dynamodb query --table-name Lab-OrdersLSI --index-name TotalIndex \
  --key-condition-expression "CustomerId = :c AND Total > :t" \
  --expression-attribute-values '{":c":{"S":"C-1"},":t":{"N":"200"}}' \
  --consistent-read \
  --query 'Items[].{Date:OrderDate.S,Total:Total.N}' --output table
```

### 4.8 Try to add an LSI later (you can't)

```bash
aws dynamodb update-table --table-name Lab-OrdersLSI \
  --local-secondary-index-updates '[]' 2>&1 | head -3
```

There is no such parameter. **LSIs are creation-time only** — this is the single most important LSI fact.

### ✅ Checkpoint

Explain: when to choose GSI over LSI; why a sparse index is cheap; what a `KEYS_ONLY` projection forces you to do; why GSIs can't be strongly consistent.

### 🧹 Cleanup

```bash
aws dynamodb delete-table --table-name Lab-OrdersLSI
# keep Lab-Orders for Lab 5
```

---

## Lab 5 — Capacity, Throttling & Auto Scaling

**Goal:** deliberately cause throttling, then fix it. Nothing teaches capacity like seeing it break.

### 5.1 Create a tiny provisioned table

```bash
aws dynamodb create-table \
  --table-name Lab-Capacity \
  --attribute-definitions AttributeName=Id,AttributeType=S \
  --key-schema AttributeName=Id,KeyType=HASH \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1

aws dynamodb wait table-exists --table-name Lab-Capacity
```

### 5.2 Break it on purpose

```bash
cat > /tmp/throttle.py << 'EOF'
import boto3, concurrent.futures, time
from botocore.config import Config
from botocore.exceptions import ClientError

# Disable retries so we SEE the throttles instead of the SDK hiding them
cfg = Config(retries={"max_attempts": 0})
c = boto3.client("dynamodb", config=cfg)

ok = throttled = 0
def write(i):
    global ok, throttled
    try:
        c.put_item(TableName="Lab-Capacity",
                   Item={"Id": {"S": f"item-{i}"}, "Payload": {"S": "x" * 500}})
        return "ok"
    except ClientError as e:
        return e.response["Error"]["Code"]

start = time.time()
with concurrent.futures.ThreadPoolExecutor(max_workers=50) as ex:
    results = list(ex.map(write, range(400)))

from collections import Counter
print(f"{time.time()-start:.1f}s")
for code, n in Counter(results).items():
    print(f"  {code}: {n}")
EOF
python3 /tmp/throttle.py
```

You should see a healthy pile of `ProvisionedThroughputExceededException`. The first few dozen succeed on burst capacity, then reality arrives.

### 5.3 Confirm it in CloudWatch

```bash
sleep 90
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB --metric-name ThrottledRequests \
  --dimensions Name=TableName,Value=Lab-Capacity \
  --start-time $(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Sum \
  --query 'Datapoints[].{Time:Timestamp,Throttles:Sum}' --output table
```

### 5.4 Fix 1 — let the SDK retry properly

```bash
cat > /tmp/retry.py << 'EOF'
import boto3, concurrent.futures, time
from botocore.config import Config
cfg = Config(retries={"max_attempts": 10, "mode": "adaptive"})
c = boto3.client("dynamodb", config=cfg)

def write(i):
    c.put_item(TableName="Lab-Capacity",
               Item={"Id": {"S": f"retry-{i}"}, "Payload": {"S": "x"*500}})

start = time.time()
with concurrent.futures.ThreadPoolExecutor(max_workers=20) as ex:
    list(ex.map(write, range(200)))
print(f"All 200 succeeded in {time.time()-start:.1f}s (adaptive retries absorbed the throttling)")
EOF
python3 /tmp/retry.py
```

Slower, but **zero failures**. `mode="adaptive"` slows the client down in response to throttling signals. This one config line prevents a large share of real production incidents.

### 5.5 Fix 2 — auto scaling

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb --resource-id "table/Lab-Capacity" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --min-capacity 1 --max-capacity 100

aws application-autoscaling put-scaling-policy \
  --service-namespace dynamodb --resource-id "table/Lab-Capacity" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --policy-name lab-write-scaling --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification":{"PredefinedMetricType":"DynamoDBWriteCapacityUtilization"},
    "ScaleOutCooldown": 60, "ScaleInCooldown": 300}'

aws application-autoscaling describe-scaling-policies \
  --service-namespace dynamodb --output table
```

Re-run `/tmp/throttle.py` and check the provisioned capacity after a few minutes:

```bash
aws dynamodb describe-table --table-name Lab-Capacity \
  --query 'Table.ProvisionedThroughput'
```

**Observe the delay.** Auto scaling reacts in minutes, so the *first* burst still throttled. This is exactly why it doesn't protect flash sales.

### 5.6 Fix 3 — on-demand

```bash
aws dynamodb update-table --table-name Lab-Capacity --billing-mode PAY_PER_REQUEST
aws dynamodb wait table-exists --table-name Lab-Capacity
python3 /tmp/throttle.py
```

No throttling. This is why on-demand is the right default for unknown traffic — and note you can only switch modes **once per 24 hours**.

### 5.7 Add a cost ceiling

```bash
aws dynamodb update-table --table-name Lab-Capacity \
  --on-demand-throughput MaxReadRequestUnits=1000,MaxWriteRequestUnits=500

aws dynamodb describe-table --table-name Lab-Capacity --query 'Table.OnDemandThroughput'
```

### 5.8 Check warm throughput

```bash
aws dynamodb describe-table --table-name Lab-Capacity --query 'Table.WarmThroughput'
```

This is the rate the table's current partitions can absorb *instantly*. Before a launch, raise it:

```bash
aws dynamodb update-table --table-name Lab-Capacity \
  --warm-throughput ReadUnitsPerSecond=12000,WriteUnitsPerSecond=6000
```

### ✅ Checkpoint

Explain: why the first requests succeeded before throttling started; why auto scaling didn't prevent the initial burst; what `mode="adaptive"` does; when to use on-demand vs provisioned.

### 🧹 Cleanup

```bash
aws application-autoscaling deregister-scalable-target \
  --service-namespace dynamodb --resource-id "table/Lab-Capacity" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits"
aws dynamodb delete-table --table-name Lab-Capacity
```

---

## Lab 6 — Transactions

**Goal:** implement a money transfer and a unique-email constraint, and see a transaction roll back.

### 6.1 Set up

```bash
aws dynamodb create-table --table-name Lab-Accounts \
  --attribute-definitions AttributeName=AccountId,AttributeType=S \
  --key-schema AttributeName=AccountId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

aws dynamodb create-table --table-name Lab-Ledger \
  --attribute-definitions AttributeName=TxnId,AttributeType=S \
  --key-schema AttributeName=TxnId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

aws dynamodb wait table-exists --table-name Lab-Accounts
aws dynamodb wait table-exists --table-name Lab-Ledger

aws dynamodb put-item --table-name Lab-Accounts --item \
  '{"AccountId":{"S":"A-1"},"Balance":{"N":"1000"},"Status":{"S":"ACTIVE"}}'
aws dynamodb put-item --table-name Lab-Accounts --item \
  '{"AccountId":{"S":"A-2"},"Balance":{"N":"500"},"Status":{"S":"ACTIVE"}}'
aws dynamodb put-item --table-name Lab-Accounts --item \
  '{"AccountId":{"S":"A-3"},"Balance":{"N":"50"},"Status":{"S":"FROZEN"}}'
```

### 6.2 A successful transfer

```bash
cat > /tmp/txn-ok.json << 'EOF'
{
  "TransactItems": [
    {"Update": {"TableName":"Lab-Accounts","Key":{"AccountId":{"S":"A-1"}},
      "UpdateExpression":"SET Balance = Balance - :amt",
      "ConditionExpression":"Balance >= :amt AND #s = :active",
      "ExpressionAttributeNames":{"#s":"Status"},
      "ExpressionAttributeValues":{":amt":{"N":"300"},":active":{"S":"ACTIVE"}}}},
    {"Update": {"TableName":"Lab-Accounts","Key":{"AccountId":{"S":"A-2"}},
      "UpdateExpression":"SET Balance = Balance + :amt",
      "ExpressionAttributeValues":{":amt":{"N":"300"}}}},
    {"Put": {"TableName":"Lab-Ledger",
      "Item":{"TxnId":{"S":"T-001"},"From":{"S":"A-1"},"To":{"S":"A-2"},"Amount":{"N":"300"}},
      "ConditionExpression":"attribute_not_exists(TxnId)"}}
  ]
}
EOF

aws dynamodb transact-write-items --cli-input-json file:///tmp/txn-ok.json \
  --client-request-token "transfer-001" --return-consumed-capacity TOTAL
```

```bash
aws dynamodb scan --table-name Lab-Accounts \
  --query 'Items[].{Acct:AccountId.S,Bal:Balance.N}' --output table
```

A-1 = 700, A-2 = 800.

### 6.3 A transaction that rolls back

```bash
cat > /tmp/txn-fail.json << 'EOF'
{
  "TransactItems": [
    {"Update": {"TableName":"Lab-Accounts","Key":{"AccountId":{"S":"A-1"}},
      "UpdateExpression":"SET Balance = Balance - :amt",
      "ConditionExpression":"Balance >= :amt",
      "ExpressionAttributeValues":{":amt":{"N":"99999"}}}},
    {"Update": {"TableName":"Lab-Accounts","Key":{"AccountId":{"S":"A-2"}},
      "UpdateExpression":"SET Balance = Balance + :amt",
      "ExpressionAttributeValues":{":amt":{"N":"99999"}}}}
  ]
}
EOF

aws dynamodb transact-write-items --cli-input-json file:///tmp/txn-fail.json
```

```bash
aws dynamodb scan --table-name Lab-Accounts \
  --query 'Items[].{Acct:AccountId.S,Bal:Balance.N}' --output table
```

**Unchanged.** A-2 was never credited even though its own action had no condition. All or nothing.

### 6.4 Read the cancellation reasons

```bash
cat > /tmp/reasons.py << 'EOF'
import boto3, json
from botocore.exceptions import ClientError
c = boto3.client("dynamodb")
try:
    c.transact_write_items(TransactItems=[
        {"Update":{"TableName":"Lab-Accounts","Key":{"AccountId":{"S":"A-1"}},
                   "UpdateExpression":"SET Balance = Balance - :a",
                   "ConditionExpression":"Balance >= :a",
                   "ExpressionAttributeValues":{":a":{"N":"99999"}}}},
        {"ConditionCheck":{"TableName":"Lab-Accounts","Key":{"AccountId":{"S":"A-3"}},
                   "ConditionExpression":"#s = :active",
                   "ExpressionAttributeNames":{"#s":"Status"},
                   "ExpressionAttributeValues":{":active":{"S":"ACTIVE"}}}},
    ])
except ClientError as e:
    print("Code:", e.response["Error"]["Code"])
    for i, r in enumerate(e.response.get("CancellationReasons", [])):
        print(f"  action[{i}]: {r.get('Code')}")
EOF
python3 /tmp/reasons.py
```

Output shows `ConditionalCheckFailed` for **both** actions — A-1 lacks funds *and* A-3 is frozen. The array positions map to your `TransactItems` order. This is how you debug a failed transaction.

### 6.5 ConditionCheck — validate without modifying

```bash
cat > /tmp/txn-check.json << 'EOF'
{
  "TransactItems": [
    {"ConditionCheck": {"TableName":"Lab-Accounts","Key":{"AccountId":{"S":"A-2"}},
      "ConditionExpression":"#s = :active",
      "ExpressionAttributeNames":{"#s":"Status"},
      "ExpressionAttributeValues":{":active":{"S":"ACTIVE"}}}},
    {"Put": {"TableName":"Lab-Ledger",
      "Item":{"TxnId":{"S":"T-002"},"Note":{"S":"verified recipient is active"}}}}
  ]
}
EOF
aws dynamodb transact-write-items --cli-input-json file:///tmp/txn-check.json
```

### 6.6 The unique-email pattern

```bash
aws dynamodb create-table --table-name Lab-Users \
  --attribute-definitions AttributeName=PK,AttributeType=S \
  --key-schema AttributeName=PK,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
aws dynamodb wait table-exists --table-name Lab-Users

register() {
  aws dynamodb transact-write-items --transact-items "[
    {\"Put\":{\"TableName\":\"Lab-Users\",
      \"Item\":{\"PK\":{\"S\":\"USER#$1\"},\"Email\":{\"S\":\"$2\"}},
      \"ConditionExpression\":\"attribute_not_exists(PK)\"}},
    {\"Put\":{\"TableName\":\"Lab-Users\",
      \"Item\":{\"PK\":{\"S\":\"EMAIL#$2\"},\"UserId\":{\"S\":\"$1\"}},
      \"ConditionExpression\":\"attribute_not_exists(PK)\"}}]"
}

register alice alice@example.com     # succeeds
register bob   alice@example.com     # fails — email taken
register bob   bob@example.com       # succeeds
```

DynamoDB has no `UNIQUE` constraint. **This transaction is how you build one.**

### 6.7 Idempotency

```bash
# Run this twice — the balance changes only once
for i in 1 2; do
  echo "attempt $i:"
  aws dynamodb transact-write-items \
    --transact-items '[{"Update":{"TableName":"Lab-Accounts",
      "Key":{"AccountId":{"S":"A-1"}},
      "UpdateExpression":"SET Balance = Balance - :a",
      "ExpressionAttributeValues":{":a":{"N":"50"}}}}]' \
    --client-request-token "idem-demo-fixed-token"
  aws dynamodb get-item --table-name Lab-Accounts \
    --key '{"AccountId":{"S":"A-1"}}' --query 'Item.Balance.N' --output text
done
```

The same `ClientRequestToken` within 10 minutes makes the retry a no-op. **Every retryable financial operation should carry one.**

### ✅ Checkpoint

Explain: what happened to A-2 when the transaction failed; how to read `CancellationReasons`; why `ClientRequestToken` matters; how to enforce uniqueness on a non-key attribute.

### 🧹 Cleanup

```bash
for t in Lab-Accounts Lab-Ledger Lab-Users; do aws dynamodb delete-table --table-name $t; done
```

---

## Lab 7 — Streams + Lambda

**Goal:** build a real event-driven pipeline: item changes automatically trigger downstream logic.

### 7.1 Table with streams enabled

```bash
aws dynamodb create-table --table-name Lab-Inventory \
  --attribute-definitions AttributeName=Sku,AttributeType=S \
  --key-schema AttributeName=Sku,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

aws dynamodb wait table-exists --table-name Lab-Inventory

STREAM_ARN=$(aws dynamodb describe-table --table-name Lab-Inventory \
  --query 'Table.LatestStreamArn' --output text)
echo "Stream: $STREAM_ARN"
```

### 7.2 IAM role for the Lambda

```bash
cat > /tmp/trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{
  "Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},
  "Action":"sts:AssumeRole"}]}
EOF

aws iam create-role --role-name LabStreamRole \
  --assume-role-policy-document file:///tmp/trust.json

aws iam attach-role-policy --role-name LabStreamRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaDynamoDBExecutionRole
aws iam attach-role-policy --role-name LabStreamRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

sleep 15   # IAM propagation
```

### 7.3 The function

```bash
mkdir -p /tmp/fn && cat > /tmp/fn/lambda_function.py << 'EOF'
import json

def lambda_handler(event, context):
    for rec in event["Records"]:
        name = rec["eventName"]                    # INSERT | MODIFY | REMOVE
        ddb  = rec["dynamodb"]
        key  = ddb["Keys"]["Sku"]["S"]
        old  = ddb.get("OldImage", {})
        new  = ddb.get("NewImage", {})

        if name == "INSERT":
            print(f"[NEW PRODUCT] {key} qty={new.get('Qty',{}).get('N')}")

        elif name == "MODIFY":
            o = int(old.get("Qty", {}).get("N", 0))
            n = int(new.get("Qty", {}).get("N", 0))
            if o != n:
                print(f"[STOCK] {key}: {o} -> {n}")
            if n == 0 and o > 0:
                print(f"[ALERT] {key} IS OUT OF STOCK - reorder now")
            elif n < 5 and o >= 5:
                print(f"[WARN] {key} low stock ({n} left)")

        elif name == "REMOVE":
            actor = rec.get("userIdentity", {}).get("principalId", "app")
            src   = "TTL expiry" if actor == "dynamodb.amazonaws.com" else "application delete"
            print(f"[DELETED] {key} via {src}")

    return {"processed": len(event["Records"])}
EOF

cd /tmp/fn && zip -q fn.zip lambda_function.py

aws lambda create-function \
  --function-name LabInventoryProcessor \
  --runtime python3.12 \
  --role arn:aws:iam::${ACCOUNT}:role/LabStreamRole \
  --handler lambda_function.lambda_handler \
  --zip-file fileb:///tmp/fn/fn.zip \
  --timeout 30
```

### 7.4 Wire the stream to the function

```bash
aws lambda create-event-source-mapping \
  --function-name LabInventoryProcessor \
  --event-source-arn "$STREAM_ARN" \
  --starting-position LATEST \
  --batch-size 10 \
  --maximum-batching-window-in-seconds 2 \
  --maximum-retry-attempts 3 \
  --bisect-batch-on-function-error

sleep 20
aws lambda list-event-source-mappings --function-name LabInventoryProcessor \
  --query 'EventSourceMappings[].{State:State,Batch:BatchSize}'
```

Wait until `State` is `Enabled`.

### 7.5 Generate events

```bash
aws dynamodb put-item --table-name Lab-Inventory \
  --item '{"Sku":{"S":"SKU-A"},"Qty":{"N":"20"},"Name":{"S":"Widget"}}'

aws dynamodb update-item --table-name Lab-Inventory \
  --key '{"Sku":{"S":"SKU-A"}}' \
  --update-expression "SET Qty = :q" --expression-attribute-values '{":q":{"N":"4"}}'

aws dynamodb update-item --table-name Lab-Inventory \
  --key '{"Sku":{"S":"SKU-A"}}' \
  --update-expression "SET Qty = :q" --expression-attribute-values '{":q":{"N":"0"}}'

aws dynamodb delete-item --table-name Lab-Inventory --key '{"Sku":{"S":"SKU-A"}}'
```

### 7.6 Read the logs

```bash
sleep 20
aws logs tail /aws/lambda/LabInventoryProcessor --since 5m --format short
```

Expected:

```
[NEW PRODUCT] SKU-A qty=20
[STOCK] SKU-A: 20 -> 4
[WARN] SKU-A low stock (4 left)
[STOCK] SKU-A: 4 -> 0
[ALERT] SKU-A IS OUT OF STOCK - reorder now
[DELETED] SKU-A via application delete
```

**No polling. No cron. No API call from your app.** The database triggered all of it.

### 7.7 Read the stream directly

```bash
SHARD=$(aws dynamodbstreams describe-stream --stream-arn "$STREAM_ARN" \
  --query 'StreamDescription.Shards[0].ShardId' --output text)
ITER=$(aws dynamodbstreams get-shard-iterator --stream-arn "$STREAM_ARN" \
  --shard-id "$SHARD" --shard-iterator-type TRIM_HORIZON \
  --query 'ShardIterator' --output text)
aws dynamodbstreams get-records --shard-iterator "$ITER" \
  --query 'Records[].{Event:eventName,Key:dynamodb.Keys.Sku.S}' --output table
```

### 7.8 Event filtering — only invoke on what matters

```bash
UUID=$(aws lambda list-event-source-mappings --function-name LabInventoryProcessor \
  --query 'EventSourceMappings[0].UUID' --output text)
aws lambda delete-event-source-mapping --uuid "$UUID"
sleep 15

aws lambda create-event-source-mapping \
  --function-name LabInventoryProcessor \
  --event-source-arn "$STREAM_ARN" \
  --starting-position LATEST --batch-size 10 \
  --filter-criteria '{"Filters":[{"Pattern":"{\"eventName\":[\"REMOVE\"]}"}]}'

sleep 20
aws dynamodb put-item --table-name Lab-Inventory --item '{"Sku":{"S":"SKU-B"},"Qty":{"N":"5"}}'
aws dynamodb delete-item --table-name Lab-Inventory --key '{"Sku":{"S":"SKU-B"}}'
sleep 20
aws logs tail /aws/lambda/LabInventoryProcessor --since 2m --format short
```

Only the delete appears. **The INSERT never invoked Lambda at all** — filtering happens before invocation, so you don't pay for it.

### ✅ Checkpoint

Explain: how to distinguish a TTL delete from an app delete; why stream consumers must be idempotent; what `--bisect-batch-on-function-error` protects you from; why event filtering saves money.

### 🧹 Cleanup

```bash
UUID=$(aws lambda list-event-source-mappings --function-name LabInventoryProcessor \
  --query 'EventSourceMappings[0].UUID' --output text)
aws lambda delete-event-source-mapping --uuid "$UUID" 2>/dev/null
aws lambda delete-function --function-name LabInventoryProcessor
aws dynamodb delete-table --table-name Lab-Inventory
aws iam detach-role-policy --role-name LabStreamRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaDynamoDBExecutionRole
aws iam detach-role-policy --role-name LabStreamRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name LabStreamRole
```

---

## Lab 8 — TTL & Session Store

**Goal:** build a self-cleaning session store and understand TTL's real timing behaviour.

### 8.1 Create and enable TTL

```bash
aws dynamodb create-table --table-name Lab-Sessions \
  --attribute-definitions AttributeName=SessionId,AttributeType=S \
  --key-schema AttributeName=SessionId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --stream-specification StreamEnabled=true,StreamViewType=OLD_IMAGE

aws dynamodb wait table-exists --table-name Lab-Sessions

aws dynamodb update-time-to-live --table-name Lab-Sessions \
  --time-to-live-specification "Enabled=true,AttributeName=ExpiresAt"

aws dynamodb describe-time-to-live --table-name Lab-Sessions
```

### 8.2 Write sessions with different lifetimes

```bash
NOW=$(date +%s)

aws dynamodb put-item --table-name Lab-Sessions --item "{
  \"SessionId\":{\"S\":\"sess-short\"},\"UserId\":{\"S\":\"U-1\"},
  \"ExpiresAt\":{\"N\":\"$((NOW + 60))\"}}"

aws dynamodb put-item --table-name Lab-Sessions --item "{
  \"SessionId\":{\"S\":\"sess-hour\"},\"UserId\":{\"S\":\"U-2\"},
  \"ExpiresAt\":{\"N\":\"$((NOW + 3600))\"}}"

aws dynamodb put-item --table-name Lab-Sessions --item "{
  \"SessionId\":{\"S\":\"sess-expired\"},\"UserId\":{\"S\":\"U-3\"},
  \"ExpiresAt\":{\"N\":\"$((NOW - 86400))\"}}"

aws dynamodb put-item --table-name Lab-Sessions --item '{
  "SessionId":{"S":"sess-forever"},"UserId":{"S":"U-4"}}'
```

### 8.3 The critical lesson — expired ≠ gone

```bash
aws dynamodb get-item --table-name Lab-Sessions --key '{"SessionId":{"S":"sess-expired"}}'
```

**It's still there**, despite being a day past expiry. TTL deletion typically completes within 48 hours, not instantly.

### 8.4 Therefore: always filter on read

```bash
NOW=$(date +%s)

cat > /tmp/get-session.sh << 'EOF'
#!/usr/bin/env bash
SID=$1
NOW=$(date +%s)
RESULT=$(aws dynamodb get-item --table-name Lab-Sessions \
  --key "{\"SessionId\":{\"S\":\"$SID\"}}" \
  --query "Item" --output json)

if [ "$RESULT" == "null" ]; then
  echo "$SID: not found"; exit 1
fi
EXP=$(echo "$RESULT" | jq -r '.ExpiresAt.N // "0"')
if [ "$EXP" != "0" ] && [ "$EXP" -lt "$NOW" ]; then
  echo "$SID: EXPIRED (logically) - treat as not found"
else
  echo "$SID: valid"
fi
EOF
chmod +x /tmp/get-session.sh

/tmp/get-session.sh sess-hour
/tmp/get-session.sh sess-expired
/tmp/get-session.sh sess-forever
```

**This filter is not optional.** Skipping it means honouring expired sessions — a real security bug.

### 8.5 The wrong-units mistake

```bash
# Milliseconds instead of seconds — will never expire (that's year 58,547)
aws dynamodb put-item --table-name Lab-Sessions --item '{
  "SessionId":{"S":"sess-bug-ms"},"ExpiresAt":{"N":"1785835200000"}}'

# String instead of Number — TTL ignores it entirely
aws dynamodb put-item --table-name Lab-Sessions --item '{
  "SessionId":{"S":"sess-bug-str"},"ExpiresAt":{"S":"1785835200"}}'
```

Both items are immortal. **TTL fails silently** — there is no error, no warning, just items that never go away. Audit for this.

### 8.6 Sliding-window session refresh

```bash
NEW_EXP=$(( $(date +%s) + 3600 ))
aws dynamodb update-item --table-name Lab-Sessions \
  --key '{"SessionId":{"S":"sess-hour"}}' \
  --update-expression "SET ExpiresAt = :e, LastSeen = :t" \
  --expression-attribute-values "{\":e\":{\"N\":\"$NEW_EXP\"},\":t\":{\"N\":\"$(date +%s)\"}}" \
  --return-values ALL_NEW
```

Every request pushes the expiry out. Inactive sessions expire on their own — zero cleanup code.

### 8.7 A rate limiter built on TTL

```bash
cat > /tmp/ratelimit.py << 'EOF'
import boto3, time
from botocore.exceptions import ClientError
t = boto3.resource("dynamodb").Table("Lab-Sessions")
LIMIT = 5

def allow(user):
    window = int(time.time() // 60)
    key = f"rate#{user}#{window}"
    try:
        t.update_item(
            Key={"SessionId": key},
            UpdateExpression="ADD Hits :one SET ExpiresAt = if_not_exists(ExpiresAt, :exp)",
            ConditionExpression="attribute_not_exists(Hits) OR Hits < :limit",
            ExpressionAttributeValues={":one": 1, ":limit": LIMIT,
                                       ":exp": (window + 2) * 60},
        )
        return True
    except ClientError as e:
        if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
            return False
        raise

for i in range(1, 9):
    print(f"request {i}: {'ALLOWED' if allow('U-1') else 'RATE LIMITED'}")
EOF
python3 /tmp/ratelimit.py
```

Requests 1–5 allowed, 6–8 blocked, and the counters delete themselves. A production-grade rate limiter in twenty lines.

### 8.8 Watch for TTL deletions in the stream

```bash
STREAM_ARN=$(aws dynamodb describe-table --table-name Lab-Sessions \
  --query 'Table.LatestStreamArn' --output text)
SHARD=$(aws dynamodbstreams describe-stream --stream-arn "$STREAM_ARN" \
  --query 'StreamDescription.Shards[0].ShardId' --output text)
ITER=$(aws dynamodbstreams get-shard-iterator --stream-arn "$STREAM_ARN" \
  --shard-id "$SHARD" --shard-iterator-type TRIM_HORIZON \
  --query 'ShardIterator' --output text)

aws dynamodbstreams get-records --shard-iterator "$ITER" \
  --query 'Records[?eventName==`REMOVE`].{Key:dynamodb.Keys.SessionId.S,By:userIdentity.principalId}' \
  --output table
```

When TTL eventually fires, `principalId` will read `dynamodb.amazonaws.com`. That's your archive-on-expiry hook.

### ✅ Checkpoint

Explain: why expired items still appear in reads; the two silent TTL failures; how sliding expiry works; how TTL + conditional writes make a rate limiter.

### 🧹 Cleanup

```bash
aws dynamodb delete-table --table-name Lab-Sessions
```

---

## Lab 9 — DAX Caching *(optional — costs money)*

> ⚠️ DAX bills per node-hour. A 3-node `dax.t3.small` cluster runs a few cents per hour. **Delete it when you finish.** Requires a VPC with at least two subnets.

**Goal:** measure the latency difference between DynamoDB and a warm cache.

### 9.1 Prepare networking

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --query 'Vpcs[0].VpcId' --output text)
SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID \
  --query 'Subnets[0:3].SubnetId' --output text)
echo "VPC=$VPC_ID SUBNETS=$SUBNETS"

aws dax create-subnet-group --subnet-group-name lab-dax-subnets --subnet-ids $SUBNETS

SG_ID=$(aws ec2 create-security-group --group-name lab-dax-sg \
  --description "DAX lab" --vpc-id $VPC_ID --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 8111 --source-group $SG_ID
```

### 9.2 DAX service role

```bash
cat > /tmp/dax-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{
  "Effect":"Allow","Principal":{"Service":"dax.amazonaws.com"},
  "Action":"sts:AssumeRole"}]}
EOF

aws iam create-role --role-name LabDAXRole \
  --assume-role-policy-document file:///tmp/dax-trust.json
aws iam attach-role-policy --role-name LabDAXRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
sleep 15
```

### 9.3 Table and data

```bash
aws dynamodb create-table --table-name Lab-Catalog \
  --attribute-definitions AttributeName=Sku,AttributeType=S \
  --key-schema AttributeName=Sku,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
aws dynamodb wait table-exists --table-name Lab-Catalog

python3 -c "
import boto3
t = boto3.resource('dynamodb').Table('Lab-Catalog')
with t.batch_writer() as b:
    for i in range(500):
        b.put_item(Item={'Sku': f'SKU-{i:04d}', 'Name': f'Product {i}',
                         'Price': i*10, 'Desc': 'x'*400})
print('500 products loaded')"
```

### 9.4 Create the cluster (takes ~10 minutes)

```bash
aws dax create-cluster \
  --cluster-name lab-dax \
  --node-type dax.t3.small \
  --replication-factor 3 \
  --iam-role-arn arn:aws:iam::${ACCOUNT}:role/LabDAXRole \
  --subnet-group-name lab-dax-subnets \
  --security-group-ids $SG_ID \
  --sse-specification Enabled=true

# Poll until available
while true; do
  S=$(aws dax describe-clusters --cluster-names lab-dax \
        --query 'Clusters[0].Status' --output text)
  echo "status: $S"; [ "$S" == "available" ] && break; sleep 30
done

aws dax describe-clusters --cluster-names lab-dax \
  --query 'Clusters[0].ClusterDiscoveryEndpoint'
```

### 9.5 Benchmark (run from an EC2 instance in the same VPC)

DAX is VPC-only — you can't reach it from your laptop. Launch a small EC2 instance in `$VPC_ID` with `$SG_ID` attached, then:

```python
# pip install amazon-dax-client boto3
import time, boto3, amazondax

TABLE = "Lab-Catalog"
ddb = boto3.resource("dynamodb")
dax = amazondax.AmazonDaxClient.resource(
    endpoint_url="dax://lab-dax.xxxxx.dax-clusters.us-east-1.amazonaws.com")

def bench(resource, label, n=1000):
    t = resource.Table(TABLE)
    t.get_item(Key={"Sku": "SKU-0001"})          # warm up
    start = time.perf_counter()
    for i in range(n):
        t.get_item(Key={"Sku": f"SKU-{i % 50:04d}"})
    total = (time.perf_counter() - start) * 1000
    print(f"{label}: {total/n:.3f} ms average over {n} reads")

bench(ddb, "DynamoDB")
bench(dax, "DAX      ")
```

Typical result: DynamoDB ~4–8 ms, DAX ~0.3–0.9 ms — **roughly an order of magnitude**.

### 9.6 Write-through in action

```python
dax.Table(TABLE).put_item(Item={"Sku": "SKU-9999", "Name": "New", "Price": 100})
print(dax.Table(TABLE).get_item(Key={"Sku": "SKU-9999"})["Item"])   # cached immediately
print(ddb.Table(TABLE).get_item(Key={"Sku": "SKU-9999"})["Item"])   # also in DynamoDB
```

The write went to DynamoDB **first**, then populated the item cache. But note: **Query results cached earlier are not invalidated by this write** — they expire only by TTL. That staleness window is DAX's main correctness caveat.

### 9.7 Tune cache TTLs

```bash
aws dax create-parameter-group --parameter-group-name lab-dax-params
aws dax update-parameter-group --parameter-group-name lab-dax-params \
  --parameter-name-values \
     ParameterName=record-ttl-millis,ParameterValue=600000 \
     ParameterName=query-ttl-millis,ParameterValue=30000
```

Shorter query TTL = fresher data, more cache misses. Pick based on how stale your reads may be.

### ✅ Checkpoint

Explain: why DAX must live in a VPC; why `ConsistentRead=True` bypasses it; the difference between the item cache and the query cache; when DAX is the wrong tool.

### 🧹 Cleanup — **do this now**

```bash
aws dax delete-cluster --cluster-name lab-dax
sleep 180
aws dax delete-subnet-group --subnet-group-name lab-dax-subnets
aws dax delete-parameter-group --parameter-group-name lab-dax-params
aws dynamodb delete-table --table-name Lab-Catalog
aws iam detach-role-policy --role-name LabDAXRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
aws iam delete-role --role-name LabDAXRole
aws ec2 delete-security-group --group-id $SG_ID
```

---

## Lab 10 — Single-Table Design

**Goal:** model a complete e-commerce domain in one table and satisfy nine access patterns.

### 10.1 The access patterns

| # | Pattern | Solution |
|---|---|---|
| 1 | Get a customer profile | Table: `PK=CUST#<id>, SK=PROFILE` |
| 2 | List a customer's orders, newest first | Table: `PK=CUST#<id>, begins_with(SK,"ORDER#")`, reversed |
| 3 | Get an order with all its line items | GSI1: `PK=ORDER#<id>` |
| 4 | Get a product | Table: `PK=PROD#<sku>, SK=META` |
| 5 | List products in a category by price | GSI1: `PK=CAT#<cat>, SK=PRICE#<padded>` |
| 6 | Find a customer by email | GSI1: `PK=EMAIL#<email>` |
| 7 | List all orders in a given status | GSI2: `PK=STATUS#<status>, SK=<date>` |
| 8 | List a customer's addresses | Table: `begins_with(SK,"ADDR#")` |
| 9 | Everything about a customer in one call | Table: `PK=CUST#<id>` |

### 10.2 Create the table

```bash
aws dynamodb create-table --table-name Lab-SingleTable \
  --attribute-definitions \
      AttributeName=PK,AttributeType=S \
      AttributeName=SK,AttributeType=S \
      AttributeName=GSI1PK,AttributeType=S \
      AttributeName=GSI1SK,AttributeType=S \
      AttributeName=GSI2PK,AttributeType=S \
      AttributeName=GSI2SK,AttributeType=S \
  --key-schema \
      AttributeName=PK,KeyType=HASH \
      AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes '[
    {"IndexName":"GSI1",
     "KeySchema":[{"AttributeName":"GSI1PK","KeyType":"HASH"},
                  {"AttributeName":"GSI1SK","KeyType":"RANGE"}],
     "Projection":{"ProjectionType":"ALL"}},
    {"IndexName":"GSI2",
     "KeySchema":[{"AttributeName":"GSI2PK","KeyType":"HASH"},
                  {"AttributeName":"GSI2SK","KeyType":"RANGE"}],
     "Projection":{"ProjectionType":"ALL"}}]' \
  --billing-mode PAY_PER_REQUEST

aws dynamodb wait table-exists --table-name Lab-SingleTable
```

**Note the generic key names.** `PK`, `SK`, `GSI1PK` mean nothing on their own — that's the point. Each entity type gives them different meaning.

### 10.3 Load the domain

```bash
cat > /tmp/seed-single.py << 'EOF'
import boto3
from decimal import Decimal
t = boto3.resource("dynamodb").Table("Lab-SingleTable")
items = []

# --- Customers ---
for cid, name, email in [("1001","Asha Rao","asha@example.com"),
                          ("1002","Karan Patel","karan@example.com")]:
    items.append({"PK": f"CUST#{cid}", "SK": "PROFILE", "Type": "Customer",
                  "Name": name, "Email": email, "Tier": "GOLD",
                  "GSI1PK": f"EMAIL#{email}", "GSI1SK": f"CUST#{cid}"})
    items.append({"PK": f"CUST#{cid}", "SK": "ADDR#HOME", "Type": "Address",
                  "Line1": "12 MG Road", "City": "Hyderabad", "Zip": "500081"})
    items.append({"PK": f"CUST#{cid}", "SK": "ADDR#WORK", "Type": "Address",
                  "Line1": "Tech Park B", "City": "Hyderabad", "Zip": "500032"})

# --- Products ---
prods = [("SKU-01","Keyboard","ELECTRONICS",8999),
         ("SKU-02","Mouse","ELECTRONICS",2499),
         ("SKU-03","Monitor","ELECTRONICS",21999),
         ("SKU-04","Desk Lamp","FURNITURE",1899)]
for sku, name, cat, price in prods:
    items.append({"PK": f"PROD#{sku}", "SK": "META", "Type": "Product",
                  "Name": name, "Category": cat, "Price": Decimal(price),
                  "GSI1PK": f"CAT#{cat}", "GSI1SK": f"PRICE#{price:08d}"})

# --- Orders + line items ---
orders = [("2001","1001","2026-08-01","SHIPPED",  [("SKU-01",1),("SKU-02",2)]),
          ("2002","1001","2026-08-03","PENDING",  [("SKU-03",1)]),
          ("2003","1002","2026-08-02","PENDING",  [("SKU-04",3)])]
for oid, cid, date, status, lines in orders:
    total = sum(dict((p[0], p[3]) for p in prods)[s] * q for s, q in lines)
    items.append({"PK": f"CUST#{cid}", "SK": f"ORDER#{date}#{oid}", "Type": "Order",
                  "OrderId": oid, "Status": status, "Total": Decimal(total),
                  "GSI1PK": f"ORDER#{oid}", "GSI1SK": "META",
                  "GSI2PK": f"STATUS#{status}", "GSI2SK": date})
    for sku, qty in lines:
        items.append({"PK": f"ORDER#{oid}", "SK": f"ITEM#{sku}", "Type": "LineItem",
                      "Sku": sku, "Qty": qty,
                      "GSI1PK": f"ORDER#{oid}", "GSI1SK": f"ITEM#{sku}"})

with t.batch_writer() as b:
    for i in items:
        b.put_item(Item=i)
print(f"Loaded {len(items)} items representing 2 customers, 4 products, 3 orders")
EOF
python3 /tmp/seed-single.py
```

### 10.4 See the shape

```bash
aws dynamodb scan --table-name Lab-SingleTable \
  --query 'Items[].{PK:PK.S,SK:SK.S,Type:Type.S}' --output table
```

Customers, addresses, products, orders, and line items — **all in one table**. Confusing at first glance, extremely efficient in practice.

### 10.5 Run every access pattern

```bash
echo "── AP1: customer profile ──"
aws dynamodb get-item --table-name Lab-SingleTable \
  --key '{"PK":{"S":"CUST#1001"},"SK":{"S":"PROFILE"}}' \
  --query 'Item.{Name:Name.S,Tier:Tier.S}'

echo "── AP2: customer's orders, newest first ──"
aws dynamodb query --table-name Lab-SingleTable \
  --key-condition-expression "PK = :pk AND begins_with(SK, :sk)" \
  --expression-attribute-values '{":pk":{"S":"CUST#1001"},":sk":{"S":"ORDER#"}}' \
  --no-scan-index-forward \
  --query 'Items[].{Order:OrderId.S,Status:Status.S,Total:Total.N}' --output table

echo "── AP3: order + its line items (one call) ──"
aws dynamodb query --table-name Lab-SingleTable --index-name GSI1 \
  --key-condition-expression "GSI1PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"ORDER#2001"}}' \
  --query 'Items[].{Type:Type.S,Detail:SK.S}' --output table

echo "── AP4: get a product ──"
aws dynamodb get-item --table-name Lab-SingleTable \
  --key '{"PK":{"S":"PROD#SKU-03"},"SK":{"S":"META"}}' \
  --query 'Item.{Name:Name.S,Price:Price.N}'

echo "── AP5: electronics by price, cheapest first ──"
aws dynamodb query --table-name Lab-SingleTable --index-name GSI1 \
  --key-condition-expression "GSI1PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"CAT#ELECTRONICS"}}' \
  --query 'Items[].{Product:Name.S,Price:Price.N}' --output table

echo "── AP6: find customer by email ──"
aws dynamodb query --table-name Lab-SingleTable --index-name GSI1 \
  --key-condition-expression "GSI1PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"EMAIL#asha@example.com"}}' \
  --query 'Items[].{Name:Name.S,PK:PK.S}'

echo "── AP7: all PENDING orders ──"
aws dynamodb query --table-name Lab-SingleTable --index-name GSI2 \
  --key-condition-expression "GSI2PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"STATUS#PENDING"}}' \
  --query 'Items[].{Order:OrderId.S,Date:GSI2SK.S,Total:Total.N}' --output table

echo "── AP8: customer's addresses ──"
aws dynamodb query --table-name Lab-SingleTable \
  --key-condition-expression "PK = :pk AND begins_with(SK, :sk)" \
  --expression-attribute-values '{":pk":{"S":"CUST#1001"},":sk":{"S":"ADDR#"}}' \
  --query 'Items[].{Type:SK.S,City:City.S}' --output table

echo "── AP9: EVERYTHING about a customer, one query ──"
aws dynamodb query --table-name Lab-SingleTable \
  --key-condition-expression "PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"CUST#1001"}}' \
  --return-consumed-capacity TOTAL \
  --query '{Items:Items[].{SK:SK.S,Type:Type.S},RCU:ConsumedCapacity.CapacityUnits}'
```

**AP9 is the payoff.** Profile, both addresses, and all orders in one round trip, ~0.5 RCU. In a relational schema that's three or four queries or a multi-way join.

### 10.6 Note the padding trick

`GSI1SK = "PRICE#00008999"` is zero-padded on purpose. Sort keys sort lexicographically, so without padding `"PRICE#2499"` would sort after `"PRICE#21999"`. Try it with unpadded values to see the bug for yourself.

### 10.7 Update across entities atomically

```bash
aws dynamodb transact-write-items --transact-items '[
  {"Update":{"TableName":"Lab-SingleTable",
    "Key":{"PK":{"S":"CUST#1001"},"SK":{"S":"ORDER#2026-08-03#2002"}},
    "UpdateExpression":"SET #s = :new, GSI2PK = :g",
    "ConditionExpression":"#s = :old",
    "ExpressionAttributeNames":{"#s":"Status"},
    "ExpressionAttributeValues":{":new":{"S":"SHIPPED"},":old":{"S":"PENDING"},
                                 ":g":{"S":"STATUS#SHIPPED"}}}}]'

# It has moved between GSI2 partitions
aws dynamodb query --table-name Lab-SingleTable --index-name GSI2 \
  --key-condition-expression "GSI2PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"STATUS#PENDING"}}' \
  --query 'Items[].OrderId.S'
```

**Important:** when the status changed you had to update `GSI2PK` too. Keeping the index key in sync is your job — DynamoDB won't derive it. This is real overhead of single-table design.

### ✅ Checkpoint

Explain: why key names are generic; what GSI overloading means; why the price is zero-padded; what maintenance burden the GSI key attributes create.

### 🧹 Cleanup

Keep `Lab-SingleTable` for Lab 11.

---

## Lab 11 — Backup, Restore & Export

**Goal:** rehearse a real disaster recovery. Delete data on purpose and get it back.

### 11.1 Enable PITR

```bash
aws dynamodb update-continuous-backups --table-name Lab-SingleTable \
  --point-in-time-recovery-specification \
    "PointInTimeRecoveryEnabled=true,RecoveryPeriodInDays=7"

aws dynamodb describe-continuous-backups --table-name Lab-SingleTable \
  --query 'ContinuousBackupsDescription.PointInTimeRecoveryDescription'
```

> PITR needs a few minutes before the earliest restorable time is available.

### 11.2 Take an on-demand backup

```bash
BACKUP_ARN=$(aws dynamodb create-backup \
  --table-name Lab-SingleTable \
  --backup-name SingleTable-BeforeIncident \
  --query 'BackupDetails.BackupArn' --output text)
echo "Backup: $BACKUP_ARN"

aws dynamodb list-backups --table-name Lab-SingleTable \
  --query 'BackupSummaries[].{Name:BackupName,Status:BackupStatus,Size:BackupSizeBytes}' \
  --output table
```

### 11.3 Record the "good" state

```bash
BEFORE=$(aws dynamodb scan --table-name Lab-SingleTable --select COUNT --query 'Count' --output text)
echo "Items before incident: $BEFORE"
echo "Safe timestamp: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
sleep 60   # let PITR record this state
```

### 11.4 Cause the incident

```bash
python3 - << 'EOF'
import boto3
t = boto3.resource("dynamodb").Table("Lab-SingleTable")
resp = t.scan(FilterExpression=boto3.dynamodb.conditions.Attr("Type").eq("Order"))
with t.batch_writer() as b:
    for i in resp["Items"]:
        b.delete_item(Key={"PK": i["PK"], "SK": i["SK"]})
print(f"Oops — deleted {len(resp['Items'])} orders")
EOF

aws dynamodb scan --table-name Lab-SingleTable --select COUNT --query 'Count'
```

### 11.5 Restore from PITR

```bash
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Lab-SingleTable \
  --target-table-name Lab-SingleTable-Restored \
  --use-latest-restorable-time

# Watch it come up (several minutes)
while true; do
  S=$(aws dynamodb describe-table --table-name Lab-SingleTable-Restored \
       --query 'Table.TableStatus' --output text 2>/dev/null || echo "CREATING")
  echo "status: $S"; [ "$S" == "ACTIVE" ] && break; sleep 30
done

aws dynamodb scan --table-name Lab-SingleTable-Restored --select COUNT --query 'Count'
```

> Depending on exactly when PITR captured state, the restored table may or may not include the deleted orders. To restore a specific earlier moment, use `--restore-date-time <the safe timestamp you printed>`.

### 11.6 Check what the restore did NOT bring back

```bash
aws dynamodb describe-continuous-backups --table-name Lab-SingleTable-Restored \
  --query 'ContinuousBackupsDescription.PointInTimeRecoveryDescription.PointInTimeRecoveryStatus'

aws dynamodb describe-time-to-live --table-name Lab-SingleTable-Restored
aws dynamodb describe-table --table-name Lab-SingleTable-Restored \
  --query 'Table.{Stream:StreamSpecification,Protection:DeletionProtectionEnabled}'
```

**PITR is DISABLED on the restored table.** So are TTL, streams, and auto scaling. Your runbook must re-apply all of them — this is the step teams forget during a real incident.

### 11.7 Restore from the on-demand backup

```bash
aws dynamodb restore-table-from-backup \
  --target-table-name Lab-SingleTable-FromBackup \
  --backup-arn "$BACKUP_ARN"

aws dynamodb wait table-exists --table-name Lab-SingleTable-FromBackup
aws dynamodb scan --table-name Lab-SingleTable-FromBackup --select COUNT --query 'Count'
```

This one should match `$BEFORE` exactly.

### 11.8 Export to S3

```bash
BUCKET="ddb-lab-export-${ACCOUNT}"
aws s3 mb "s3://$BUCKET" --region $REGION

EXPORT_ARN=$(aws dynamodb export-table-to-point-in-time \
  --table-arn "arn:aws:dynamodb:${REGION}:${ACCOUNT}:table/Lab-SingleTable" \
  --s3-bucket "$BUCKET" --s3-prefix exports/ \
  --export-format DYNAMODB_JSON \
  --query 'ExportDescription.ExportArn' --output text)

while true; do
  S=$(aws dynamodb describe-export --export-arn "$EXPORT_ARN" \
       --query 'ExportDescription.ExportStatus' --output text)
  echo "export: $S"; [ "$S" != "IN_PROGRESS" ] && break; sleep 30
done

aws s3 ls "s3://$BUCKET/exports/" --recursive --human-readable
```

**Zero read capacity consumed** — the export is served from the continuous backup, not the live table.

### 11.9 Import into a new table

```bash
# Find and inspect an exported data file
KEY=$(aws s3 ls "s3://$BUCKET/exports/" --recursive \
      | grep 'data/.*\.json\.gz' | head -1 | awk '{print $4}')
aws s3 cp "s3://$BUCKET/$KEY" - | gunzip | head -2
```

The export format is line-delimited JSON, ready for Athena, Glue, or `import-table`.

### 11.10 Deletion protection

```bash
aws dynamodb update-table --table-name Lab-SingleTable --deletion-protection-enabled

# Now try to delete it
aws dynamodb delete-table --table-name Lab-SingleTable
```

Fails with `ValidationException`. **Enable this on every production table.**

### ✅ Checkpoint

Explain: the difference between PITR and on-demand backups; what settings a restore loses; why export-to-S3 doesn't consume RCU; why restores always create a new table.

### 🧹 Cleanup

```bash
aws dynamodb update-table --table-name Lab-SingleTable --no-deletion-protection-enabled
sleep 10
for t in Lab-SingleTable Lab-SingleTable-Restored Lab-SingleTable-FromBackup; do
  aws dynamodb delete-table --table-name $t 2>/dev/null
done
aws dynamodb delete-backup --backup-arn "$BACKUP_ARN"
aws s3 rb "s3://$BUCKET" --force
```

---

## Lab 12 — Deploy a Serverless API

**Goal:** ship a complete, production-shaped REST API: API Gateway → Lambda → DynamoDB, with least-privilege IAM, PITR, and alarms.

### 12.1 The production table

```bash
aws dynamodb create-table --table-name Prod-Tasks \
  --attribute-definitions \
      AttributeName=PK,AttributeType=S \
      AttributeName=SK,AttributeType=S \
      AttributeName=GSI1PK,AttributeType=S \
      AttributeName=GSI1SK,AttributeType=S \
  --key-schema \
      AttributeName=PK,KeyType=HASH \
      AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes '[{
      "IndexName":"GSI1",
      "KeySchema":[{"AttributeName":"GSI1PK","KeyType":"HASH"},
                   {"AttributeName":"GSI1SK","KeyType":"RANGE"}],
      "Projection":{"ProjectionType":"ALL"}}]' \
  --billing-mode PAY_PER_REQUEST \
  --sse-specification Enabled=true,SSEType=KMS \
  --deletion-protection-enabled \
  --tags Key=Environment,Value=Production Key=App,Value=TaskAPI

aws dynamodb wait table-exists --table-name Prod-Tasks

aws dynamodb update-continuous-backups --table-name Prod-Tasks \
  --point-in-time-recovery-specification \
    "PointInTimeRecoveryEnabled=true,RecoveryPeriodInDays=7"

aws dynamodb update-time-to-live --table-name Prod-Tasks \
  --time-to-live-specification "Enabled=true,AttributeName=ExpiresAt"
```

### 12.2 Least-privilege IAM

```bash
cat > /tmp/api-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{
  "Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},
  "Action":"sts:AssumeRole"}]}
EOF

aws iam create-role --role-name TaskAPIRole \
  --assume-role-policy-document file:///tmp/api-trust.json

cat > /tmp/api-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TableAccess",
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem","dynamodb:Query","dynamodb:PutItem",
                 "dynamodb:UpdateItem","dynamodb:DeleteItem"],
      "Resource": [
        "arn:aws:dynamodb:${REGION}:${ACCOUNT}:table/Prod-Tasks",
        "arn:aws:dynamodb:${REGION}:${ACCOUNT}:table/Prod-Tasks/index/*"
      ]
    },
    { "Sid": "NoScans", "Effect": "Deny", "Action": "dynamodb:Scan", "Resource": "*" }
  ]
}
EOF

aws iam put-role-policy --role-name TaskAPIRole \
  --policy-name TaskAPIDynamoDB --policy-document file:///tmp/api-policy.json
aws iam attach-role-policy --role-name TaskAPIRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
sleep 15
```

Note both the `/index/*` resource **and** the explicit `Scan` deny.

### 12.3 The API handler

```bash
mkdir -p /tmp/api && cat > /tmp/api/app.py << 'PYEOF'
import json, os, uuid, time
import boto3
from boto3.dynamodb.conditions import Key, Attr
from botocore.config import Config
from botocore.exceptions import ClientError

cfg = Config(retries={"max_attempts": 5, "mode": "adaptive"},
             connect_timeout=1, read_timeout=3)
table = boto3.resource("dynamodb", config=cfg).Table(os.environ["TABLE_NAME"])


def resp(code, body):
    return {"statusCode": code,
            "headers": {"Content-Type": "application/json",
                        "Access-Control-Allow-Origin": "*"},
            "body": json.dumps(body, default=str)}


def create_task(user, body):
    tid = str(uuid.uuid4())[:8]
    now = int(time.time())
    item = {
        "PK": f"USER#{user}",
        "SK": f"TASK#{now}#{tid}",
        "TaskId": tid,
        "Title": body["title"],
        "Status": body.get("status", "TODO"),
        "Priority": body.get("priority", "MEDIUM"),
        "CreatedAt": now,
        "Version": 1,
        "GSI1PK": f"STATUS#{body.get('status','TODO')}",
        "GSI1SK": f"{now}",
    }
    if body.get("ttl_days"):
        item["ExpiresAt"] = now + int(body["ttl_days"]) * 86400
    table.put_item(Item=item, ConditionExpression=Attr("PK").not_exists())
    return resp(201, item)


def list_tasks(user, qs):
    kwargs = {
        "KeyConditionExpression": Key("PK").eq(f"USER#{user}")
                                  & Key("SK").begins_with("TASK#"),
        "ScanIndexForward": False,
        "Limit": int(qs.get("limit", 20)),
    }
    if qs.get("status"):
        kwargs["FilterExpression"] = Attr("Status").eq(qs["status"])
    if qs.get("cursor"):
        kwargs["ExclusiveStartKey"] = json.loads(qs["cursor"])

    r = table.query(**kwargs)
    out = {"tasks": r["Items"], "count": r["Count"]}
    if "LastEvaluatedKey" in r:
        out["cursor"] = json.dumps(r["LastEvaluatedKey"], default=str)
    return resp(200, out)


def list_by_status(status, qs):
    r = table.query(
        IndexName="GSI1",
        KeyConditionExpression=Key("GSI1PK").eq(f"STATUS#{status}"),
        ScanIndexForward=False,
        Limit=int(qs.get("limit", 20)),
    )
    return resp(200, {"tasks": r["Items"], "count": r["Count"]})


def update_task(user, sk, body):
    exprs, names, vals = [], {}, {}
    for field in ("Title", "Status", "Priority"):
        key = field.lower()
        if key in body:
            exprs.append(f"#{key} = :{key}")
            names[f"#{key}"] = field
            vals[f":{key}"] = body[key]
    if not exprs:
        return resp(400, {"error": "no updatable fields supplied"})

    exprs.append("Version = Version + :one")
    vals[":one"] = 1
    if "Status" in [names[k] for k in names]:
        exprs.append("GSI1PK = :gsi")
        vals[":gsi"] = f"STATUS#{body['status']}"

    try:
        r = table.update_item(
            Key={"PK": f"USER#{user}", "SK": sk},
            UpdateExpression="SET " + ", ".join(exprs),
            ConditionExpression=Attr("PK").exists()
                                & Attr("Version").eq(body.get("version", 1)),
            ExpressionAttributeNames=names,
            ExpressionAttributeValues=vals,
            ReturnValues="ALL_NEW",
        )
        return resp(200, r["Attributes"])
    except ClientError as e:
        if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
            return resp(409, {"error": "version conflict or task not found"})
        raise


def delete_task(user, sk):
    try:
        r = table.delete_item(
            Key={"PK": f"USER#{user}", "SK": sk},
            ConditionExpression=Attr("PK").exists(),
            ReturnValues="ALL_OLD",
        )
        return resp(200, {"deleted": r["Attributes"]["TaskId"]})
    except ClientError as e:
        if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
            return resp(404, {"error": "task not found"})
        raise


def lambda_handler(event, context):
    try:
        method = event["requestContext"]["http"]["method"]
        path   = event["rawPath"]
        qs     = event.get("queryStringParameters") or {}
        body   = json.loads(event["body"]) if event.get("body") else {}
        user   = qs.get("user") or body.get("user") or "demo-user"

        if path == "/tasks" and method == "POST":
            return create_task(user, body)
        if path == "/tasks" and method == "GET":
            return resp(200, list_by_status(qs["status"], qs)) if False else (
                list_by_status(qs["status"], qs) if qs.get("byStatus") else list_tasks(user, qs))
        if path.startswith("/tasks/") and method == "PATCH":
            return update_task(user, path.split("/tasks/", 1)[1], body)
        if path.startswith("/tasks/") and method == "DELETE":
            return delete_task(user, path.split("/tasks/", 1)[1])

        return resp(404, {"error": "route not found"})

    except KeyError as e:
        return resp(400, {"error": f"missing field: {e}"})
    except ClientError as e:
        code = e.response["Error"]["Code"]
        if code in ("ProvisionedThroughputExceededException", "ThrottlingException"):
            return resp(429, {"error": "rate limited, please retry"})
        print(f"AWS error: {code} - {e}")
        return resp(500, {"error": "internal error"})
    except Exception as e:
        print(f"Unhandled: {type(e).__name__}: {e}")
        return resp(500, {"error": "internal error"})
PYEOF

cd /tmp/api && zip -q api.zip app.py

aws lambda create-function \
  --function-name TaskAPI \
  --runtime python3.12 \
  --role arn:aws:iam::${ACCOUNT}:role/TaskAPIRole \
  --handler app.lambda_handler \
  --zip-file fileb:///tmp/api/api.zip \
  --timeout 10 --memory-size 256 \
  --environment "Variables={TABLE_NAME=Prod-Tasks}"
```

### 12.4 Expose it via API Gateway

```bash
API_URL=$(aws lambda create-function-url-config \
  --function-name TaskAPI --auth-type NONE \
  --cors '{"AllowOrigins":["*"],"AllowMethods":["*"],"AllowHeaders":["content-type"]}' \
  --query 'FunctionUrl' --output text)

aws lambda add-permission --function-name TaskAPI \
  --statement-id FunctionURLAllowPublicAccess \
  --action lambda:InvokeFunctionUrl \
  --principal "*" --function-url-auth-type NONE

echo "API: $API_URL"
```

> ⚠️ `--auth-type NONE` is a public endpoint. Fine for a lab, **never** for real data — use `AWS_IAM`, a Cognito authorizer, or API Gateway with a JWT authorizer in production.

### 12.5 Exercise it

```bash
sleep 10

echo "── create ──"
curl -sX POST "${API_URL}tasks" -H 'Content-Type: application/json' \
  -d '{"user":"asha","title":"Write DynamoDB docs","priority":"HIGH"}' | jq

curl -sX POST "${API_URL}tasks" -H 'Content-Type: application/json' \
  -d '{"user":"asha","title":"Review PR","status":"IN_PROGRESS"}' | jq -r '.TaskId'

curl -sX POST "${API_URL}tasks" -H 'Content-Type: application/json' \
  -d '{"user":"asha","title":"Temp note","ttl_days":1}' | jq -r '.TaskId'

echo "── list ──"
curl -s "${API_URL}tasks?user=asha&limit=10" | jq '.count, .tasks[].Title'

echo "── list by status via GSI ──"
curl -s "${API_URL}tasks?byStatus=1&status=TODO" | jq '.count'

echo "── update with optimistic lock ──"
SK=$(curl -s "${API_URL}tasks?user=asha&limit=1" | jq -r '.tasks[0].SK')
curl -sX PATCH "${API_URL}tasks/${SK}?user=asha" -H 'Content-Type: application/json' \
  -d '{"status":"DONE","version":1}' | jq '{Status,Version}'

echo "── stale version is rejected (409) ──"
curl -sX PATCH "${API_URL}tasks/${SK}?user=asha" -H 'Content-Type: application/json' \
  -d '{"status":"TODO","version":1}' | jq

echo "── delete ──"
curl -sX DELETE "${API_URL}tasks/${SK}?user=asha" | jq
```

### 12.6 Add monitoring

```bash
aws sns create-topic --name task-api-alerts
TOPIC=$(aws sns list-topics --query "Topics[?contains(TopicArn,'task-api-alerts')].TopicArn" --output text)

aws cloudwatch put-metric-alarm --alarm-name TaskAPI-DDB-Throttles \
  --namespace AWS/DynamoDB --metric-name ThrottledRequests \
  --dimensions Name=TableName,Value=Prod-Tasks \
  --statistic Sum --period 300 --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching --alarm-actions "$TOPIC"

aws cloudwatch put-metric-alarm --alarm-name TaskAPI-DDB-SystemErrors \
  --namespace AWS/DynamoDB --metric-name SystemErrors \
  --dimensions Name=TableName,Value=Prod-Tasks \
  --statistic Sum --period 300 --evaluation-periods 2 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold --alarm-actions "$TOPIC"

aws cloudwatch put-metric-alarm --alarm-name TaskAPI-Lambda-Errors \
  --namespace AWS/Lambda --metric-name Errors \
  --dimensions Name=FunctionName,Value=TaskAPI \
  --statistic Sum --period 300 --evaluation-periods 1 --threshold 3 \
  --comparison-operator GreaterThanOrEqualToThreshold --alarm-actions "$TOPIC"

aws dynamodb update-contributor-insights \
  --table-name Prod-Tasks --contributor-insights-action ENABLE
```

### 12.7 Verify your production checklist

```bash
echo "── Table configuration audit ──"
aws dynamodb describe-table --table-name Prod-Tasks --query 'Table.{
  Billing:BillingModeSummary.BillingMode,
  Encryption:SSEDescription.Status,
  DeletionProtection:DeletionProtectionEnabled,
  Indexes:GlobalSecondaryIndexes[].IndexName}'

aws dynamodb describe-continuous-backups --table-name Prod-Tasks \
  --query 'ContinuousBackupsDescription.PointInTimeRecoveryDescription.PointInTimeRecoveryStatus'

aws dynamodb describe-time-to-live --table-name Prod-Tasks \
  --query 'TimeToLiveDescription.TimeToLiveStatus'

echo "── IAM: confirm Scan is denied ──"
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::${ACCOUNT}:role/TaskAPIRole \
  --action-names dynamodb:Scan dynamodb:Query \
  --resource-arns arn:aws:dynamodb:${REGION}:${ACCOUNT}:table/Prod-Tasks \
  --query 'EvaluationResults[].{Action:EvalActionName,Decision:EvalDecision}' --output table
```

Expected: `Scan → explicitDeny`, `Query → allowed`.

### 12.8 What you built

```
   curl / browser
        │  HTTPS
        ▼
  Lambda Function URL  (CORS enabled)
        │
        ▼
  TaskAPI Lambda  ── adaptive retries, structured errors, 429 on throttle
        │  IAM role: table + index only, Scan explicitly denied
        ▼
  Prod-Tasks (DynamoDB)
   ├─ PK/SK single-table layout
   ├─ GSI1 for status lookup
   ├─ TTL for auto-expiring tasks
   ├─ PITR (7-day window)
   ├─ KMS encryption
   ├─ Deletion protection
   └─ Contributor Insights
        │
        ▼
  CloudWatch alarms → SNS
```

### ✅ Final checkpoint

You can now: design a key schema from access patterns, use GSIs correctly, handle throttling and concurrency, wire streams, automate expiry, enforce least-privilege access, back up and restore, and deploy the whole thing.

### 🧹 Cleanup

```bash
aws lambda delete-function-url-config --function-name TaskAPI
aws lambda delete-function --function-name TaskAPI
aws iam delete-role-policy --role-name TaskAPIRole --policy-name TaskAPIDynamoDB
aws iam detach-role-policy --role-name TaskAPIRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name TaskAPIRole
aws dynamodb update-table --table-name Prod-Tasks --no-deletion-protection-enabled
sleep 10
aws dynamodb delete-table --table-name Prod-Tasks
aws cloudwatch delete-alarms --alarm-names \
  TaskAPI-DDB-Throttles TaskAPI-DDB-SystemErrors TaskAPI-Lambda-Errors
aws sns delete-topic --topic-arn "$TOPIC"
```

---

## Final Sweep — Delete Everything

```bash
for t in Lab-Products Lab-Orders Lab-OrdersLSI Lab-Capacity Lab-Accounts \
         Lab-Ledger Lab-Users Lab-Inventory Lab-Sessions Lab-Catalog \
         Lab-SingleTable Lab-SingleTable-Restored Lab-SingleTable-FromBackup \
         Prod-Tasks; do
  aws dynamodb update-table --table-name "$t" --no-deletion-protection-enabled 2>/dev/null
  aws dynamodb delete-table --table-name "$t" 2>/dev/null && echo "deleted $t"
done

echo "--- remaining tables ---"
aws dynamodb list-tables --query 'TableNames'

echo "--- remaining backups ---"
aws dynamodb list-backups --query 'BackupSummaries[].BackupName'

echo "--- DAX clusters (should be empty) ---"
aws dax describe-clusters --query 'Clusters[].ClusterName'
```

---

## Where to Go Next

**Extend these labs:**
- Turn Lab 10's table into a global table across two Regions and measure replication latency
- Add a Streams → Lambda → OpenSearch pipeline for full-text search on tasks
- Rewrite Lab 12 in Terraform or CDK
- Load-test Lab 12 with `artillery` or `k6` and watch Contributor Insights
- Model your own application's access patterns in NoSQL Workbench

**Reference material:**
➡️ **[README.md](./README.md)** — the concepts behind each lab
➡️ **[commands-cheatsheet.md](./commands-cheatsheet.md)** — every command
➡️ **[troubleshooting.md](./troubleshooting.md)** — when a lab doesn't behave
