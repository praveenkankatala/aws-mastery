# Amazon DynamoDB — CLI Commands Cheatsheet

> Every command you'll realistically need, grouped by what you're trying to do. Copy, paste, adjust the names.

**Assumed setup:** AWS CLI v2, credentials configured, `--region` either configured or added to each command.

---

## Table of Contents

- [0. Setup & Conventions](#0-setup--conventions)
- [1. Table Management](#1-table-management)
- [2. Writing Items](#2-writing-items)
- [3. Reading Items](#3-reading-items)
- [4. Updating Items](#4-updating-items)
- [5. Deleting Items](#5-deleting-items)
- [6. Batch Operations](#6-batch-operations)
- [7. Transactions](#7-transactions)
- [8. Secondary Indexes](#8-secondary-indexes)
- [9. Capacity & Auto Scaling](#9-capacity--auto-scaling)
- [10. DynamoDB Streams](#10-dynamodb-streams)
- [11. Time to Live (TTL)](#11-time-to-live-ttl)
- [12. Backup & Restore](#12-backup--restore)
- [13. Import & Export with S3](#13-import--export-with-s3)
- [14. Global Tables](#14-global-tables)
- [15. DAX](#15-dax)
- [16. PartiQL](#16-partiql)
- [17. Security & Policies](#17-security--policies)
- [18. Tagging](#18-tagging)
- [19. Monitoring & Insights](#19-monitoring--insights)
- [20. DynamoDB Local](#20-dynamodb-local)
- [21. Quotas & Account Limits](#21-quotas--account-limits)
- [22. Useful JMESPath Queries](#22-useful-jmespath-queries)
- [23. Handy One-Liners & Scripts](#23-handy-one-liners--scripts)
- [24. Expression Syntax Reference](#24-expression-syntax-reference)
- [25. Common Flags Reference](#25-common-flags-reference)

---

## 0. Setup & Conventions

```bash
# Configure
aws configure
aws configure set region us-east-1
aws configure set output json

# Verify identity and access
aws sts get-caller-identity
aws dynamodb list-tables

# Helpful shell variables used throughout this document
export TABLE=Orders
export REGION=us-east-1
export ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
```

**Two ways to pass complex data:**

```bash
# Inline JSON (watch your shell quoting)
--item '{"UserId":{"S":"U-1042"}}'

# From a file (cleaner for anything non-trivial)
--item file://item.json
```

**If you hit binary encoding errors on CLI v2:**

```bash
aws dynamodb put-item --cli-binary-format raw-in-base64-out ...
# or set it permanently
aws configure set cli_binary_format raw-in-base64-out
```

---

## 1. Table Management

### Create a table — minimal

```bash
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions AttributeName=UserId,AttributeType=S \
  --key-schema AttributeName=UserId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### Create a table — composite key

```bash
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
      AttributeName=UserId,AttributeType=S \
      AttributeName=OrderDate,AttributeType=S \
  --key-schema \
      AttributeName=UserId,KeyType=HASH \
      AttributeName=OrderDate,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST
```

### Create a table — provisioned capacity

```bash
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions AttributeName=UserId,AttributeType=S \
  --key-schema AttributeName=UserId,KeyType=HASH \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=5
```

### Create a table — the full production version

```bash
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
      AttributeName=UserId,AttributeType=S \
      AttributeName=OrderDate,AttributeType=S \
      AttributeName=Status,AttributeType=S \
      AttributeName=Total,AttributeType=N \
  --key-schema \
      AttributeName=UserId,KeyType=HASH \
      AttributeName=OrderDate,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes '[
    {
      "IndexName": "StatusIndex",
      "KeySchema": [
        {"AttributeName":"Status","KeyType":"HASH"},
        {"AttributeName":"OrderDate","KeyType":"RANGE"}],
      "Projection": {"ProjectionType":"INCLUDE","NonKeyAttributes":["UserId","Total"]}
    }]' \
  --local-secondary-indexes '[
    {
      "IndexName": "TotalIndex",
      "KeySchema": [
        {"AttributeName":"UserId","KeyType":"HASH"},
        {"AttributeName":"Total","KeyType":"RANGE"}],
      "Projection": {"ProjectionType":"ALL"}
    }]' \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --sse-specification Enabled=true,SSEType=KMS \
  --table-class STANDARD \
  --deletion-protection-enabled \
  --tags Key=Environment,Value=Production Key=Owner,Value=Platform
```

> ⚠️ `--attribute-definitions` must contain **exactly** the attributes used in any key schema (table, LSI, or GSI) — no more, no less.

### Wait for the table to be ready

```bash
aws dynamodb wait table-exists --table-name Orders
aws dynamodb wait table-not-exists --table-name Orders    # after delete
```

### Inspect

```bash
# List all tables
aws dynamodb list-tables
aws dynamodb list-tables --max-items 10
aws dynamodb list-tables --exclusive-start-table-name Orders   # paginate

# Full description
aws dynamodb describe-table --table-name Orders

# Just the status
aws dynamodb describe-table --table-name Orders --query 'Table.TableStatus' --output text

# Item count and size (updated roughly every 6 hours)
aws dynamodb describe-table --table-name Orders \
  --query 'Table.{Items:ItemCount,Bytes:TableSizeBytes}'

# Key schema
aws dynamodb describe-table --table-name Orders --query 'Table.KeySchema'

# All indexes
aws dynamodb describe-table --table-name Orders \
  --query 'Table.{GSI:GlobalSecondaryIndexes[].IndexName,LSI:LocalSecondaryIndexes[].IndexName}'
```

### Update

```bash
# Switch to provisioned
aws dynamodb update-table --table-name Orders \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=20,WriteCapacityUnits=10

# Switch to on-demand
aws dynamodb update-table --table-name Orders --billing-mode PAY_PER_REQUEST

# Cap an on-demand table
aws dynamodb update-table --table-name Orders \
  --on-demand-throughput MaxReadRequestUnits=10000,MaxWriteRequestUnits=5000

# Change table class
aws dynamodb update-table --table-name Orders --table-class STANDARD_INFREQUENT_ACCESS

# Pre-warm throughput ahead of a spike
aws dynamodb update-table --table-name Orders \
  --warm-throughput ReadUnitsPerSecond=40000,WriteUnitsPerSecond=20000

# Deletion protection
aws dynamodb update-table --table-name Orders --deletion-protection-enabled
aws dynamodb update-table --table-name Orders --no-deletion-protection-enabled

# Change KMS key
aws dynamodb update-table --table-name Orders \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=alias/my-key
```

### Delete

```bash
aws dynamodb delete-table --table-name Orders
aws dynamodb wait table-not-exists --table-name Orders
```

---

## 2. Writing Items

### PutItem

```bash
aws dynamodb put-item --table-name Orders \
  --item '{
    "UserId":    {"S":"U-1042"},
    "OrderDate": {"S":"2026-08-04"},
    "Status":    {"S":"PENDING"},
    "Total":     {"N":"149.99"},
    "Paid":      {"BOOL":false},
    "Tags":      {"SS":["gift","express"]},
    "Address":   {"M":{"City":{"S":"Hyderabad"},"Zip":{"S":"500081"}}},
    "Lines":     {"L":[{"S":"SKU-1"},{"S":"SKU-2"}]},
    "Note":      {"NULL":true}
  }'
```

### PutItem from a file

```bash
cat > order.json << 'EOF'
{
  "UserId":    {"S":"U-1043"},
  "OrderDate": {"S":"2026-08-05"},
  "Status":    {"S":"PENDING"},
  "Total":     {"N":"220.00"}
}
EOF

aws dynamodb put-item --table-name Orders --item file://order.json
```

### Conditional put — insert only, never overwrite

```bash
aws dynamodb put-item --table-name Orders \
  --item file://order.json \
  --condition-expression "attribute_not_exists(UserId) AND attribute_not_exists(OrderDate)"
```

### Return the previous version

```bash
aws dynamodb put-item --table-name Orders \
  --item file://order.json \
  --return-values ALL_OLD \
  --return-consumed-capacity TOTAL \
  --return-item-collection-metrics SIZE
```

### See what failed a condition

```bash
aws dynamodb put-item --table-name Orders \
  --item file://order.json \
  --condition-expression "attribute_not_exists(UserId)" \
  --return-values-on-condition-check-failure ALL_OLD
```

---

## 3. Reading Items

### GetItem

```bash
aws dynamodb get-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}'
```

### GetItem — strongly consistent, projected, with cost

```bash
aws dynamodb get-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}' \
  --consistent-read \
  --projection-expression "OrderId, #s, Total, Address.City" \
  --expression-attribute-names '{"#s":"Status"}' \
  --return-consumed-capacity TOTAL
```

### Query — all items for a partition key

```bash
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u" \
  --expression-attribute-values '{":u":{"S":"U-1042"}}'
```

### Query — sort key conditions

```bash
# begins_with
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u AND begins_with(OrderDate, :prefix)" \
  --expression-attribute-values '{":u":{"S":"U-1042"},":prefix":{"S":"2026-08"}}'

# BETWEEN
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u AND OrderDate BETWEEN :a AND :b" \
  --expression-attribute-values '{":u":{"S":"U-1042"},":a":{"S":"2026-01-01"},":b":{"S":"2026-06-30"}}'

# Greater than
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u AND OrderDate > :d" \
  --expression-attribute-values '{":u":{"S":"U-1042"},":d":{"S":"2026-01-01"}}'
```

### Query — newest first, limited

```bash
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u" \
  --expression-attribute-values '{":u":{"S":"U-1042"}}' \
  --no-scan-index-forward \
  --limit 10
```

### Query — with a filter (applied after reading)

```bash
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u" \
  --filter-expression "#s = :st AND Total > :min" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{
      ":u":{"S":"U-1042"},":st":{"S":"PENDING"},":min":{"N":"100"}}'
```

### Query — an index

```bash
aws dynamodb query --table-name Orders \
  --index-name StatusIndex \
  --key-condition-expression "#s = :st" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":st":{"S":"PENDING"}}'
```

### Query — count only

```bash
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u" \
  --expression-attribute-values '{":u":{"S":"U-1042"}}' \
  --select COUNT
```

### Query — pagination

```bash
# First page
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u" \
  --expression-attribute-values '{":u":{"S":"U-1042"}}' \
  --limit 5

# Next page — feed back LastEvaluatedKey
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u" \
  --expression-attribute-values '{":u":{"S":"U-1042"}}' \
  --limit 5 \
  --exclusive-start-key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-03-30"}}'
```

### Scan

```bash
# Full scan (be careful)
aws dynamodb scan --table-name Orders

# With a filter
aws dynamodb scan --table-name Orders \
  --filter-expression "#s = :st" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":st":{"S":"PENDING"}}'

# Count items in a table
aws dynamodb scan --table-name Orders --select COUNT

# Scan an index
aws dynamodb scan --table-name Orders --index-name StatusIndex

# Parallel scan — segment 0 of 4 (run 4 processes concurrently)
aws dynamodb scan --table-name Orders --total-segments 4 --segment 0
```

### Scan — automatic full pagination

```bash
# CLI paginates automatically unless you set --page-size / --max-items
aws dynamodb scan --table-name Orders --page-size 100 --output json > all-items.json
```

---

## 4. Updating Items

### SET — assign values

```bash
aws dynamodb update-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}' \
  --update-expression "SET #s = :st, UpdatedAt = :now" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":st":{"S":"SHIPPED"},":now":{"S":"2026-08-04T12:00:00Z"}}' \
  --return-values ALL_NEW
```

### SET — arithmetic

```bash
--update-expression "SET Total = Total + :amt, Discount = Discount - :d"
--expression-attribute-values '{":amt":{"N":"10"},":d":{"N":"5"}}'
```

### SET — only if absent

```bash
--update-expression "SET CreatedAt = if_not_exists(CreatedAt, :now), UpdatedAt = :now"
```

### SET — append to a list

```bash
--update-expression "SET Lines = list_append(Lines, :new)"
--expression-attribute-values '{":new":{"L":[{"S":"SKU-9"}]}}'

# Prepend instead
--update-expression "SET Lines = list_append(:new, Lines)"
```

### SET — nested attributes

```bash
--update-expression "SET Address.City = :c, Metadata.#src = :s"
--expression-attribute-names '{"#src":"Source"}'
--expression-attribute-values '{":c":{"S":"Bengaluru"},":s":{"S":"mobile"}}'
```

### ADD — atomic counter

```bash
aws dynamodb update-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}' \
  --update-expression "ADD ViewCount :one" \
  --expression-attribute-values '{":one":{"N":"1"}}' \
  --return-values UPDATED_NEW
```

### ADD / DELETE — set members

```bash
# Add tags
--update-expression "ADD Tags :t"
--expression-attribute-values '{":t":{"SS":["priority"]}}'

# Remove tags
--update-expression "DELETE Tags :t"
--expression-attribute-values '{":t":{"SS":["express"]}}'
```

### REMOVE — delete attributes

```bash
--update-expression "REMOVE Discount, TempFlag, Lines[2]"
```

### Combined clauses

```bash
aws dynamodb update-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}' \
  --update-expression "SET #s = :st ADD Revisions :one REMOVE TempFlag DELETE Tags :old" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{
      ":st":{"S":"SHIPPED"},":one":{"N":"1"},":old":{"SS":["draft"]}}'
```

### Conditional update — optimistic locking

```bash
aws dynamodb update-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}' \
  --update-expression "SET #s = :new, Version = Version + :one" \
  --condition-expression "Version = :expected" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{
      ":new":{"S":"SHIPPED"},":one":{"N":"1"},":expected":{"N":"3"}}'
```

### Conditional update — state machine guard

```bash
--condition-expression "#s = :pending AND attribute_exists(PaymentId)"
--expression-attribute-values '{":pending":{"S":"PENDING"}}'
```

---

## 5. Deleting Items

```bash
# Simple
aws dynamodb delete-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}'

# Conditional
aws dynamodb delete-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}' \
  --condition-expression "#s = :cancelled" \
  --expression-attribute-names '{"#s":"Status"}' \
  --expression-attribute-values '{":cancelled":{"S":"CANCELLED"}}'

# Return what was deleted
aws dynamodb delete-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}' \
  --return-values ALL_OLD
```

> 💡 There is no "delete all items" API. To empty a table, either delete and recreate it (fastest, cheapest) or scan + `BatchWriteItem` delete (see [§23](#23-handy-one-liners--scripts)).

---

## 6. Batch Operations

### BatchWriteItem (max 25 items)

```bash
cat > batch.json << 'EOF'
{
  "Orders": [
    { "PutRequest": { "Item": {
        "UserId":{"S":"U-2001"},"OrderDate":{"S":"2026-08-01"},"Total":{"N":"50"}}}},
    { "PutRequest": { "Item": {
        "UserId":{"S":"U-2002"},"OrderDate":{"S":"2026-08-02"},"Total":{"N":"75"}}}},
    { "DeleteRequest": { "Key": {
        "UserId":{"S":"U-9999"},"OrderDate":{"S":"2026-01-01"}}}}
  ]
}
EOF

aws dynamodb batch-write-item \
  --request-items file://batch.json \
  --return-consumed-capacity TOTAL
```

> ⚠️ Always inspect `UnprocessedItems` in the response and retry those keys with exponential backoff.

### BatchGetItem (max 100 items)

```bash
cat > batch-get.json << 'EOF'
{
  "Orders": {
    "Keys": [
      {"UserId":{"S":"U-2001"},"OrderDate":{"S":"2026-08-01"}},
      {"UserId":{"S":"U-2002"},"OrderDate":{"S":"2026-08-02"}}
    ],
    "ConsistentRead": true,
    "ProjectionExpression": "UserId, Total"
  }
}
EOF

aws dynamodb batch-get-item --request-items file://batch-get.json
```

### Multi-table batch

```json
{
  "Orders":   { "Keys": [ {"UserId":{"S":"U-1"},"OrderDate":{"S":"2026-08-01"}} ] },
  "Products": { "Keys": [ {"Sku":{"S":"SKU-9"}} ] }
}
```

---

## 7. Transactions

### TransactWriteItems

```bash
cat > txn.json << 'EOF'
{
  "TransactItems": [
    { "Update": {
        "TableName": "Accounts",
        "Key": {"AccountId":{"S":"A-1"}},
        "UpdateExpression": "SET Balance = Balance - :amt",
        "ConditionExpression": "Balance >= :amt",
        "ExpressionAttributeValues": {":amt":{"N":"100"}}}},
    { "Update": {
        "TableName": "Accounts",
        "Key": {"AccountId":{"S":"A-2"}},
        "UpdateExpression": "SET Balance = Balance + :amt",
        "ExpressionAttributeValues": {":amt":{"N":"100"}}}},
    { "Put": {
        "TableName": "Ledger",
        "Item": {"TxnId":{"S":"T-9001"},"Amount":{"N":"100"}},
        "ConditionExpression": "attribute_not_exists(TxnId)"}},
    { "ConditionCheck": {
        "TableName": "Accounts",
        "Key": {"AccountId":{"S":"A-1"}},
        "ConditionExpression": "#s = :active",
        "ExpressionAttributeNames": {"#s":"Status"},
        "ExpressionAttributeValues": {":active":{"S":"ACTIVE"}}}}
  ]
}
EOF

aws dynamodb transact-write-items \
  --cli-input-json file://txn.json \
  --client-request-token "idem-key-2026-08-04-001" \
  --return-consumed-capacity TOTAL
```

### TransactGetItems

```bash
aws dynamodb transact-get-items --transact-items '[
  {"Get":{"TableName":"Accounts","Key":{"AccountId":{"S":"A-1"}}}},
  {"Get":{"TableName":"Accounts","Key":{"AccountId":{"S":"A-2"}}}}
]'
```

---

## 8. Secondary Indexes

### Add a GSI to an existing table

```bash
aws dynamodb update-table --table-name Orders \
  --attribute-definitions \
      AttributeName=Status,AttributeType=S \
      AttributeName=OrderDate,AttributeType=S \
  --global-secondary-index-updates '[{
    "Create": {
      "IndexName": "StatusIndex",
      "KeySchema": [
        {"AttributeName":"Status","KeyType":"HASH"},
        {"AttributeName":"OrderDate","KeyType":"RANGE"}],
      "Projection": {"ProjectionType":"INCLUDE","NonKeyAttributes":["UserId","Total"]}
    }}]'
```

### Add a GSI to a provisioned table

```bash
--global-secondary-index-updates '[{
  "Create": {
    "IndexName": "StatusIndex",
    "KeySchema": [{"AttributeName":"Status","KeyType":"HASH"}],
    "Projection": {"ProjectionType":"KEYS_ONLY"},
    "ProvisionedThroughput": {"ReadCapacityUnits":10,"WriteCapacityUnits":5}
  }}]'
```

### Update a GSI's capacity

```bash
aws dynamodb update-table --table-name Orders \
  --global-secondary-index-updates '[{
    "Update": {
      "IndexName": "StatusIndex",
      "ProvisionedThroughput": {"ReadCapacityUnits":50,"WriteCapacityUnits":25}
    }}]'
```

### Delete a GSI

```bash
aws dynamodb update-table --table-name Orders \
  --global-secondary-index-updates '[{"Delete":{"IndexName":"StatusIndex"}}]'
```

### Check backfill progress

```bash
aws dynamodb describe-table --table-name Orders \
  --query 'Table.GlobalSecondaryIndexes[].{
      Name:IndexName,Status:IndexStatus,Backfilling:Backfilling,Items:ItemCount}' \
  --output table
```

---

## 9. Capacity & Auto Scaling

### Register scalable targets

```bash
# Reads
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
  --min-capacity 5 --max-capacity 1000

# Writes
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --min-capacity 5 --max-capacity 500

# A GSI
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Orders/index/StatusIndex" \
  --scalable-dimension "dynamodb:index:ReadCapacityUnits" \
  --min-capacity 5 --max-capacity 500
```

### Attach scaling policies

```bash
aws application-autoscaling put-scaling-policy \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits" \
  --policy-name OrdersReadTargetTracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {"PredefinedMetricType":"DynamoDBReadCapacityUtilization"},
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300}'
```

### Scheduled scaling (for predictable spikes)

```bash
aws application-autoscaling put-scheduled-action \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:WriteCapacityUnits" \
  --scheduled-action-name scale-up-for-batch \
  --schedule "cron(45 1 * * ? *)" \
  --scalable-target-action MinCapacity=500,MaxCapacity=2000
```

### Inspect & remove

```bash
aws application-autoscaling describe-scalable-targets --service-namespace dynamodb
aws application-autoscaling describe-scaling-policies --service-namespace dynamodb
aws application-autoscaling describe-scaling-activities --service-namespace dynamodb

aws application-autoscaling deregister-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/Orders" \
  --scalable-dimension "dynamodb:table:ReadCapacityUnits"
```

### Warm throughput

```bash
# Check current warm throughput
aws dynamodb describe-table --table-name Orders --query 'Table.WarmThroughput'

# Raise it before a launch
aws dynamodb update-table --table-name Orders \
  --warm-throughput ReadUnitsPerSecond=50000,WriteUnitsPerSecond=25000
```

---

## 10. DynamoDB Streams

### Enable / disable / change view type

```bash
aws dynamodb update-table --table-name Orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

aws dynamodb update-table --table-name Orders \
  --stream-specification StreamEnabled=false
```

### Get the stream ARN

```bash
aws dynamodb describe-table --table-name Orders \
  --query 'Table.LatestStreamArn' --output text
```

### Read a stream directly (`dynamodbstreams` service)

```bash
STREAM_ARN=$(aws dynamodb describe-table --table-name Orders \
  --query 'Table.LatestStreamArn' --output text)

# List streams
aws dynamodbstreams list-streams --table-name Orders

# Describe shards
aws dynamodbstreams describe-stream --stream-arn "$STREAM_ARN"

# Get a shard iterator
SHARD_ID=$(aws dynamodbstreams describe-stream --stream-arn "$STREAM_ARN" \
  --query 'StreamDescription.Shards[0].ShardId' --output text)

ITER=$(aws dynamodbstreams get-shard-iterator \
  --stream-arn "$STREAM_ARN" \
  --shard-id "$SHARD_ID" \
  --shard-iterator-type TRIM_HORIZON \
  --query 'ShardIterator' --output text)

# Read records
aws dynamodbstreams get-records --shard-iterator "$ITER"
```

Shard iterator types: `TRIM_HORIZON` (oldest), `LATEST` (new only), `AT_SEQUENCE_NUMBER`, `AFTER_SEQUENCE_NUMBER`.

### Hook up Lambda

```bash
aws lambda create-event-source-mapping \
  --function-name ProcessOrderChanges \
  --event-source-arn "$STREAM_ARN" \
  --starting-position LATEST \
  --batch-size 100 \
  --maximum-batching-window-in-seconds 5 \
  --maximum-retry-attempts 3 \
  --maximum-record-age-in-seconds 3600 \
  --bisect-batch-on-function-error \
  --parallelization-factor 2 \
  --destination-config '{"OnFailure":{"Destination":"arn:aws:sqs:us-east-1:123456789012:stream-dlq"}}'
```

### Event filtering

```bash
aws lambda create-event-source-mapping \
  --function-name ProcessShipped \
  --event-source-arn "$STREAM_ARN" \
  --starting-position LATEST \
  --filter-criteria '{"Filters":[{"Pattern":"{\"dynamodb\":{\"NewImage\":{\"Status\":{\"S\":[\"SHIPPED\"]}}}}"}]}'
```

### Manage event source mappings

```bash
aws lambda list-event-source-mappings --function-name ProcessOrderChanges
aws lambda update-event-source-mapping --uuid <uuid> --enabled
aws lambda update-event-source-mapping --uuid <uuid> --no-enabled
aws lambda delete-event-source-mapping --uuid <uuid>
```

### Kinesis Data Streams CDC

```bash
aws dynamodb enable-kinesis-streaming-destination \
  --table-name Orders \
  --stream-arn arn:aws:kinesis:us-east-1:123456789012:stream/orders-cdc

aws dynamodb describe-kinesis-streaming-destination --table-name Orders

aws dynamodb disable-kinesis-streaming-destination \
  --table-name Orders \
  --stream-arn arn:aws:kinesis:us-east-1:123456789012:stream/orders-cdc
```

---

## 11. Time to Live (TTL)

```bash
# Enable
aws dynamodb update-time-to-live --table-name Sessions \
  --time-to-live-specification "Enabled=true,AttributeName=ExpiresAt"

# Check status
aws dynamodb describe-time-to-live --table-name Sessions

# Disable
aws dynamodb update-time-to-live --table-name Sessions \
  --time-to-live-specification "Enabled=false,AttributeName=ExpiresAt"
```

### Write an item that expires in one hour

```bash
EXPIRES=$(( $(date +%s) + 3600 ))

aws dynamodb put-item --table-name Sessions \
  --item "{\"SessionId\":{\"S\":\"S-001\"},\"ExpiresAt\":{\"N\":\"$EXPIRES\"}}"
```

### Filter out logically-expired items on read

```bash
NOW=$(date +%s)
aws dynamodb query --table-name Sessions \
  --key-condition-expression "SessionId = :s" \
  --filter-expression "ExpiresAt > :now" \
  --expression-attribute-values "{\":s\":{\"S\":\"S-001\"},\":now\":{\"N\":\"$NOW\"}}"
```

### Epoch helpers

```bash
date +%s                                    # now
date -d "+30 days" +%s                      # Linux
date -v+30d +%s                             # macOS
python3 -c "import time;print(int(time.time())+86400)"   # portable
date -d @1785835200                         # epoch → human (Linux)
```

---

## 12. Backup & Restore

### Point-in-Time Recovery

```bash
aws dynamodb update-continuous-backups --table-name Orders \
  --point-in-time-recovery-specification \
    "PointInTimeRecoveryEnabled=true,RecoveryPeriodInDays=14"

aws dynamodb describe-continuous-backups --table-name Orders

# What window is available?
aws dynamodb describe-continuous-backups --table-name Orders \
  --query 'ContinuousBackupsDescription.PointInTimeRecoveryDescription'
```

### Restore to a point in time

```bash
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Orders \
  --target-table-name Orders-Restored \
  --restore-date-time 2026-08-04T09:15:00Z

# Restore the earliest available point
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Orders \
  --target-table-name Orders-Restored \
  --use-latest-restorable-time

# Faster restore: skip GSIs
aws dynamodb restore-table-to-point-in-time \
  --source-table-name Orders \
  --target-table-name Orders-Restored \
  --use-latest-restorable-time \
  --global-secondary-index-override '[]'

# Cross-Region restore
aws dynamodb restore-table-to-point-in-time \
  --source-table-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --target-table-name Orders-DR \
  --use-latest-restorable-time \
  --region eu-west-1
```

### On-demand backups

```bash
aws dynamodb create-backup --table-name Orders --backup-name Orders-2026-08-04

aws dynamodb list-backups --table-name Orders
aws dynamodb list-backups --time-range-lower-bound 2026-08-01T00:00:00Z

aws dynamodb describe-backup --backup-arn <arn>

aws dynamodb restore-table-from-backup \
  --target-table-name Orders-FromBackup \
  --backup-arn <arn>

aws dynamodb delete-backup --backup-arn <arn>
```

### AWS Backup integration

```bash
aws backup start-backup-job \
  --backup-vault-name Default \
  --resource-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --iam-role-arn arn:aws:iam::123456789012:role/service-role/AWSBackupDefaultServiceRole

aws backup list-recovery-points-by-resource \
  --resource-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders
```

---

## 13. Import & Export with S3

### Export

```bash
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --s3-bucket my-exports \
  --s3-prefix orders/full \
  --export-format DYNAMODB_JSON \
  --export-time 2026-08-04T00:00:00Z

# Incremental export — only changes between two times
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --s3-bucket my-exports \
  --s3-prefix orders/incremental \
  --export-type INCREMENTAL_EXPORT \
  --incremental-export-specification \
    ExportFromTime=2026-08-03T00:00:00Z,ExportToTime=2026-08-04T00:00:00Z

aws dynamodb list-exports --table-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders
aws dynamodb describe-export --export-arn <arn>
```

### Import

```bash
aws dynamodb import-table \
  --s3-bucket-source S3Bucket=my-imports,S3KeyPrefix=orders/ \
  --input-format CSV \
  --input-format-options '{"Csv":{"Delimiter":","}}' \
  --input-compression-type GZIP \
  --table-creation-parameters '{
      "TableName":"OrdersImported",
      "AttributeDefinitions":[
        {"AttributeName":"UserId","AttributeType":"S"},
        {"AttributeName":"OrderDate","AttributeType":"S"}],
      "KeySchema":[
        {"AttributeName":"UserId","KeyType":"HASH"},
        {"AttributeName":"OrderDate","KeyType":"RANGE"}],
      "BillingMode":"PAY_PER_REQUEST"}'

aws dynamodb list-imports
aws dynamodb describe-import --import-arn <arn>

# How many rows failed?
aws dynamodb describe-import --import-arn <arn> \
  --query 'ImportTableDescription.{Processed:ProcessedItemCount,Errors:ErrorCount}'
```

---

## 14. Global Tables

```bash
# Prerequisite: streams with NEW_AND_OLD_IMAGES
aws dynamodb update-table --table-name Orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

# Add a replica Region (MREC — eventual consistency)
aws dynamodb update-table --table-name Orders \
  --replica-updates '[{"Create":{"RegionName":"eu-west-1"}}]'

# Add a replica with its own KMS key and index capacity
aws dynamodb update-table --table-name Orders \
  --replica-updates '[{"Create":{
      "RegionName":"ap-south-1",
      "KMSMasterKeyId":"alias/aws/dynamodb",
      "GlobalSecondaryIndexes":[{
        "IndexName":"StatusIndex",
        "ProvisionedThroughputOverride":{"ReadCapacityUnits":20}}]
  }}]'

# Remove a replica
aws dynamodb update-table --table-name Orders \
  --replica-updates '[{"Delete":{"RegionName":"eu-west-1"}}]'

# Inspect replicas
aws dynamodb describe-table --table-name Orders \
  --query 'Table.Replicas[].{Region:RegionName,Status:ReplicaStatus,Detail:ReplicaStatusDescription}' \
  --output table

# Global table settings (v2019.11.21)
aws dynamodb describe-table --table-name Orders --query 'Table.GlobalTableVersion'
```

### Create a table with MRSC (multi-Region strong consistency)

```bash
aws dynamodb create-table \
  --table-name Inventory \
  --attribute-definitions AttributeName=Sku,AttributeType=S \
  --key-schema AttributeName=Sku,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --multi-region-consistency STRONG \
  --replica-updates '[
     {"Create":{"RegionName":"us-east-2"}},
     {"Create":{"RegionName":"us-west-2"}}]'
```

> ⚠️ MRSC requires exactly three Regions (or two replicas plus a witness), must be chosen at creation, and cannot be changed later.

---

## 15. DAX

```bash
# Subnet group
aws dax create-subnet-group \
  --subnet-group-name dax-subnets \
  --subnet-ids subnet-aaa subnet-bbb subnet-ccc

# Cluster
aws dax create-cluster \
  --cluster-name orders-dax \
  --node-type dax.t3.small \
  --replication-factor 3 \
  --iam-role-arn arn:aws:iam::123456789012:role/DAXServiceRole \
  --subnet-group-name dax-subnets \
  --security-group-ids sg-0123456789abcdef0 \
  --sse-specification Enabled=true

# Get the endpoint
aws dax describe-clusters --cluster-names orders-dax \
  --query 'Clusters[0].ClusterDiscoveryEndpoint'

# Parameter group — cache TTLs
aws dax create-parameter-group --parameter-group-name dax-params
aws dax update-parameter-group --parameter-group-name dax-params \
  --parameter-name-values \
     ParameterName=record-ttl-millis,ParameterValue=300000 \
     ParameterName=query-ttl-millis,ParameterValue=60000

# Scale / delete
aws dax increase-replication-factor --cluster-name orders-dax --new-replication-factor 5
aws dax decrease-replication-factor --cluster-name orders-dax --new-replication-factor 3
aws dax delete-cluster --cluster-name orders-dax     # remember: nodes bill hourly
```

---

## 16. PartiQL

```bash
# SELECT
aws dynamodb execute-statement \
  --statement "SELECT * FROM \"Orders\" WHERE UserId = 'U-1042'"

# SELECT from an index
aws dynamodb execute-statement \
  --statement "SELECT UserId, Total FROM \"Orders\".\"StatusIndex\" WHERE Status = 'PENDING'"

# With parameters (safer)
aws dynamodb execute-statement \
  --statement "SELECT * FROM \"Orders\" WHERE UserId = ?" \
  --parameters '[{"S":"U-1042"}]'

# INSERT
aws dynamodb execute-statement \
  --statement "INSERT INTO \"Orders\" VALUE {'UserId':'U-3001','OrderDate':'2026-08-04','Total':99}"

# UPDATE
aws dynamodb execute-statement \
  --statement "UPDATE \"Orders\" SET Status='SHIPPED' WHERE UserId='U-3001' AND OrderDate='2026-08-04'"

# DELETE
aws dynamodb execute-statement \
  --statement "DELETE FROM \"Orders\" WHERE UserId='U-3001' AND OrderDate='2026-08-04'"

# Paginate
aws dynamodb execute-statement \
  --statement "SELECT * FROM \"Orders\"" --limit 10 --next-token <token>

# Batch (up to 25 statements, read OR write, not mixed)
aws dynamodb batch-execute-statement --statements '[
  {"Statement":"SELECT * FROM \"Orders\" WHERE UserId=? AND OrderDate=?","Parameters":[{"S":"U-1"},{"S":"2026-08-01"}]},
  {"Statement":"SELECT * FROM \"Orders\" WHERE UserId=? AND OrderDate=?","Parameters":[{"S":"U-2"},{"S":"2026-08-02"}]}
]'

# Transactional PartiQL
aws dynamodb execute-transaction --transact-statements '[
  {"Statement":"UPDATE \"Accounts\" SET Balance=Balance-100 WHERE AccountId=?","Parameters":[{"S":"A-1"}]},
  {"Statement":"UPDATE \"Accounts\" SET Balance=Balance+100 WHERE AccountId=?","Parameters":[{"S":"A-2"}]}
]'
```

---

## 17. Security & Policies

### Resource-based policy

```bash
cat > table-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "CrossAccountRead",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::999988887777:role/PartnerRole"},
    "Action": ["dynamodb:GetItem","dynamodb:Query"],
    "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Orders"
  }]
}
EOF

aws dynamodb put-resource-policy \
  --resource-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --policy file://table-policy.json

aws dynamodb get-resource-policy \
  --resource-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders

aws dynamodb delete-resource-policy \
  --resource-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders
```

### VPC endpoint

```bash
# Gateway endpoint (free, route-table based)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0abc123 \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-0def456

# Interface endpoint / PrivateLink
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0abc123 \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-aaa subnet-bbb \
  --security-group-ids sg-0123 \
  --private-dns-enabled
```

### Test a policy before you deploy it

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/AppRole \
  --action-names dynamodb:Query dynamodb:Scan \
  --resource-arns arn:aws:dynamodb:us-east-1:123456789012:table/Orders
```

---

## 18. Tagging

```bash
aws dynamodb tag-resource \
  --resource-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --tags Key=Environment,Value=Production Key=CostCenter,Value=Platform

aws dynamodb list-tags-of-resource \
  --resource-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders

aws dynamodb untag-resource \
  --resource-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --tag-keys CostCenter
```

---

## 19. Monitoring & Insights

### Contributor Insights

```bash
aws dynamodb update-contributor-insights \
  --table-name Orders --contributor-insights-action ENABLE

aws dynamodb update-contributor-insights \
  --table-name Orders --index-name StatusIndex --contributor-insights-action ENABLE

aws dynamodb describe-contributor-insights --table-name Orders
aws dynamodb list-contributor-insights
```

### CloudWatch metrics

```bash
# Consumed read capacity, last hour
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedReadCapacityUnits \
  --dimensions Name=TableName,Value=Orders \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Sum Average

# Throttles
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB --metric-name ThrottledRequests \
  --dimensions Name=TableName,Value=Orders \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 3600 --statistics Sum

# Latency by operation
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB --metric-name SuccessfulRequestLatency \
  --dimensions Name=TableName,Value=Orders Name=Operation,Value=Query \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average Maximum
```

### Alarms

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name Orders-Throttled \
  --alarm-description "DynamoDB Orders table is throttling" \
  --namespace AWS/DynamoDB --metric-name ThrottledRequests \
  --dimensions Name=TableName,Value=Orders \
  --statistic Sum --period 300 --evaluation-periods 1 \
  --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts

aws cloudwatch put-metric-alarm \
  --alarm-name Orders-SystemErrors \
  --namespace AWS/DynamoDB --metric-name SystemErrors \
  --dimensions Name=TableName,Value=Orders \
  --statistic Sum --period 300 --evaluation-periods 2 \
  --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

### Per-request cost telemetry

```bash
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u" \
  --expression-attribute-values '{":u":{"S":"U-1042"}}' \
  --return-consumed-capacity INDEXES \
  --query 'ConsumedCapacity'
```

---

## 20. DynamoDB Local

```bash
# Run in Docker
docker run -d -p 8000:8000 --name ddb-local amazon/dynamodb-local

# Persist data across restarts
docker run -d -p 8000:8000 -v "$PWD/ddb-data:/home/dynamodblocal/data" \
  --name ddb-local amazon/dynamodb-local \
  -jar DynamoDBLocal.jar -sharedDb -dbPath ./data

# Point the CLI at it (dummy credentials are required but unused)
export AWS_ACCESS_KEY_ID=local
export AWS_SECRET_ACCESS_KEY=local
export AWS_DEFAULT_REGION=us-east-1

alias ddbl='aws dynamodb --endpoint-url http://localhost:8000'

ddbl create-table --table-name Orders \
  --attribute-definitions AttributeName=UserId,AttributeType=S \
  --key-schema AttributeName=UserId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

ddbl list-tables
ddbl put-item --table-name Orders --item '{"UserId":{"S":"U-1"}}'
ddbl scan --table-name Orders
```

```python
# boto3 against local
import boto3
ddb = boto3.resource("dynamodb", endpoint_url="http://localhost:8000",
                     region_name="us-east-1",
                     aws_access_key_id="local", aws_secret_access_key="local")
```

---

## 21. Quotas & Account Limits

```bash
aws dynamodb describe-limits

aws service-quotas list-service-quotas --service-code dynamodb --output table

aws service-quotas get-service-quota \
  --service-code dynamodb --quota-code L-F98FE922    # max tables per Region

aws service-quotas request-service-quota-increase \
  --service-code dynamodb --quota-code L-F98FE922 --desired-value 5000
```

---

## 22. Useful JMESPath Queries

```bash
# Table names only, one per line
aws dynamodb list-tables --query 'TableNames[]' --output text | tr '\t' '\n'

# Item values without the type descriptors
aws dynamodb get-item --table-name Orders \
  --key '{"UserId":{"S":"U-1042"},"OrderDate":{"S":"2026-08-04"}}' \
  --query 'Item.{User:UserId.S,Status:Status.S,Total:Total.N}'

# Query results as a table
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u" \
  --expression-attribute-values '{":u":{"S":"U-1042"}}' \
  --query 'Items[].{Date:OrderDate.S,Status:Status.S,Total:Total.N}' \
  --output table

# Every table's billing mode
for t in $(aws dynamodb list-tables --query 'TableNames[]' --output text); do
  mode=$(aws dynamodb describe-table --table-name "$t" \
    --query 'Table.BillingModeSummary.BillingMode' --output text)
  printf "%-40s %s\n" "$t" "${mode:-PROVISIONED}"
done

# Tables without PITR enabled — a useful audit
for t in $(aws dynamodb list-tables --query 'TableNames[]' --output text); do
  s=$(aws dynamodb describe-continuous-backups --table-name "$t" \
    --query 'ContinuousBackupsDescription.PointInTimeRecoveryDescription.PointInTimeRecoveryStatus' \
    --output text 2>/dev/null)
  [ "$s" != "ENABLED" ] && echo "NO PITR: $t"
done

# Tables without deletion protection
for t in $(aws dynamodb list-tables --query 'TableNames[]' --output text); do
  d=$(aws dynamodb describe-table --table-name "$t" \
    --query 'Table.DeletionProtectionEnabled' --output text)
  [ "$d" != "True" ] && echo "UNPROTECTED: $t"
done
```

---

## 23. Handy One-Liners & Scripts

### Count items (accurate but costs a full scan)

```bash
aws dynamodb scan --table-name Orders --select COUNT --query 'Count'
```

### Estimated count (free, but up to 6 hours stale)

```bash
aws dynamodb describe-table --table-name Orders --query 'Table.ItemCount'
```

### Dump a whole table to a file

```bash
aws dynamodb scan --table-name Orders --output json > orders-dump.json
```

### Empty a table (small tables only)

```bash
#!/usr/bin/env bash
set -euo pipefail
TABLE=Orders
PK=UserId
SK=OrderDate

aws dynamodb scan --table-name "$TABLE" \
  --projection-expression "$PK, $SK" \
  --query "Items[].{$PK:$PK,$SK:$SK}" --output json \
| jq -c ".[] | {DeleteRequest:{Key:.}}" \
| while read -r req; do
    echo "{\"$TABLE\":[$req]}" > /tmp/del.json
    aws dynamodb batch-write-item --request-items file:///tmp/del.json > /dev/null
  done
echo "Table $TABLE emptied."
```

> 💡 For large tables, `delete-table` + `create-table` is faster and free.

### Copy a table to a new one

```bash
# Best for large tables: export to S3 → import to a new table (no capacity cost)
aws dynamodb export-table-to-point-in-time \
  --table-arn "arn:aws:dynamodb:$REGION:$ACCOUNT:table/Orders" \
  --s3-bucket my-exports --s3-prefix copy/ --export-format DYNAMODB_JSON
# then: aws dynamodb import-table --s3-bucket-source ... (see §13)
```

### Bulk-load a JSON array with jq

```bash
jq -c '.[] | {PutRequest:{Item:.}}' items.json \
| jq -s -c '_nwise(25) | {"Orders": .}' \
| while read -r batch; do
    echo "$batch" > /tmp/b.json
    aws dynamodb batch-write-item --request-items file:///tmp/b.json
  done
```

### Watch a table's status until it's ACTIVE

```bash
watch -n 5 "aws dynamodb describe-table --table-name Orders \
  --query 'Table.{Status:TableStatus,GSI:GlobalSecondaryIndexes[].IndexStatus}'"
```

### Cost check on a single query

```bash
aws dynamodb query --table-name Orders \
  --key-condition-expression "UserId = :u" \
  --expression-attribute-values '{":u":{"S":"U-1042"}}' \
  --return-consumed-capacity INDEXES \
  --query '{RCU:ConsumedCapacity.CapacityUnits,Scanned:ScannedCount,Returned:Count}'
```

---

## 24. Expression Syntax Reference

### Key condition operators (Query only)

```
Partition key:  MUST be  =
Sort key:       =  <  <=  >  >=  BETWEEN ... AND ...  begins_with(sk, :prefix)
```

### Filter / condition operators

```
Comparison:  =  <>  <  <=  >  >=
Range:       BETWEEN :a AND :b
Set:         IN (:a, :b, :c)
Logical:     AND  OR  NOT  ( )
```

### Functions

| Function | Where | Purpose |
|---|---|---|
| `attribute_exists(path)` | condition, filter | Attribute is present |
| `attribute_not_exists(path)` | condition, filter | Attribute is absent |
| `attribute_type(path, "S")` | condition, filter | Type check |
| `begins_with(path, :s)` | key, condition, filter | Prefix match |
| `contains(path, :s)` | condition, filter | Substring or set membership |
| `size(path)` | condition, filter | Length of value |
| `list_append(:a, :b)` | update | Concatenate lists |
| `if_not_exists(path, :default)` | update | Default when absent |

### Update clauses

```
SET     attr = :val
        attr = attr + :n
        attr = if_not_exists(attr, :default)
        attr = list_append(attr, :items)
        map.nested = :val
        list[0] = :val

REMOVE  attr, other.nested, list[2]

ADD     numAttr :delta          (Number only)
        setAttr :members        (Set only)

DELETE  setAttr :members        (Set only)
```

Multiple clauses in one expression are separated by spaces, each clause appearing at most once:

```
SET a = :x, b = :y REMOVE c ADD d :n DELETE e :m
```

### Reserved words — a partial list

`ABORT ACTION ADD ALL ALTER AND ANY AS ASC BETWEEN BINARY BOOLEAN BY BYTE COUNT DATA DATE DELETE DESC DOUBLE DROP EXISTS FALSE FROM FULL GROUP HASH IN INDEX INSERT INT INTO KEY LANGUAGE LIST LOCAL LONG MAP MAX MIN NAME NULL NUMBER OBJECT OR ORDER OWNER PATH RANGE RECORD ROLE ROW SCAN SCHEMA SELECT SET SIZE SOURCE START STATE STATUS STRING TABLE TIME TIMESTAMP TO TRUE TYPE UPDATE USER VALUE VIEW WHERE YEAR ZONE` … (~573 total)

> 💡 Don't memorize the list. Alias every attribute name with `#`.

---

## 25. Common Flags Reference

| Flag | Applies to | Purpose |
|---|---|---|
| `--table-name` | most | Target table |
| `--index-name` | query, scan | Target a GSI or LSI |
| `--key` | get, update, delete | Full primary key |
| `--item` | put | Complete item |
| `--consistent-read` | get, query, scan, batch-get | Strongly consistent (not on GSI) |
| `--projection-expression` | reads | Attributes to return |
| `--filter-expression` | query, scan | Post-read filter (no cost saving) |
| `--key-condition-expression` | query | The efficient key selector |
| `--condition-expression` | writes | Allow/reject the write |
| `--update-expression` | update | The mutation |
| `--expression-attribute-names` | any expression | `#alias` → real attribute name |
| `--expression-attribute-values` | any expression | `:alias` → typed value |
| `--return-values` | writes | `NONE`, `ALL_OLD`, `ALL_NEW`, `UPDATED_OLD`, `UPDATED_NEW` |
| `--return-values-on-condition-check-failure` | writes | `ALL_OLD` — see what blocked you |
| `--return-consumed-capacity` | most | `NONE`, `TOTAL`, `INDEXES` |
| `--return-item-collection-metrics` | writes | `NONE`, `SIZE` — LSI collection size |
| `--limit` | query, scan | Max items read per page (before filter) |
| `--exclusive-start-key` | query, scan | Pagination cursor |
| `--scan-index-forward` / `--no-scan-index-forward` | query | Sort ascending / descending |
| `--select` | query, scan | `ALL_ATTRIBUTES`, `ALL_PROJECTED_ATTRIBUTES`, `SPECIFIC_ATTRIBUTES`, `COUNT` |
| `--segment` / `--total-segments` | scan | Parallel scan |
| `--client-request-token` | transact-write | Idempotency for 10 minutes |
| `--endpoint-url` | all | Point at DynamoDB Local |
| `--region` | all | Override configured Region |
| `--output` | all | `json`, `table`, `text`, `yaml` |
| `--query` | all | JMESPath filter on the response |
| `--no-cli-pager` | all | Stop the CLI from opening `less` |

---

## Next Steps

➡️ **[hands-on-labs.md](./hands-on-labs.md)** — put these commands to work
➡️ **[troubleshooting.md](./troubleshooting.md)** — when a command fails
➡️ **[README.md](./README.md)** — the concepts behind the commands
