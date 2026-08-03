# ElastiCache CLI Command Cheatsheet

> Every command here is organized by *task*, not alphabetically — because when you're working, you think "I need to create a subnet group," not "I need the C section." All commands assume AWS CLI v2 and a configured profile/region.

📌 **Tip:** Add `--region us-east-1` (or your region) to any command if it's not set as your default. Add `--output table` to any command for prettier terminal output.

---

## Table of Contents
1. [Setup & Identity](#1-setup--identity)
2. [Networking Prerequisites](#2-networking-prerequisites)
3. [Parameter Groups](#3-parameter-groups)
4. [Memcached Cluster Commands](#4-memcached-cluster-commands)
5. [Redis Replication Group Commands](#5-redis-replication-group-commands)
6. [Cluster Mode / Sharding Commands](#6-cluster-mode--sharding-commands)
7. [Serverless Cache Commands](#7-serverless-cache-commands)
8. [Snapshots & Backups](#8-snapshots--backups)
9. [Users & RBAC (Redis 6+)](#9-users--rbac-redis-6)
10. [Global Datastore (Cross-Region)](#10-global-datastore-cross-region)
11. [Security & Encryption](#11-security--encryption)
12. [Tagging](#12-tagging)
13. [Monitoring & Events](#13-monitoring--events)
14. [Scaling Operations](#14-scaling-operations)
15. [Cleanup / Deletion](#15-cleanup--deletion)
16. [Engine-Level Client Commands (redis-cli / memcached)](#16-engine-level-client-commands)

---

## 1. Setup & Identity

```bash
# Verify CLI version (need v2)
aws --version

# Configure credentials/profile
aws configure

# Confirm which account/identity you're using
aws sts get-caller-identity

# List available ElastiCache regions/endpoints (sanity check)
aws elasticache describe-cache-engine-versions --engine redis
```

---

## 2. Networking Prerequisites

```bash
# List your VPCs
aws ec2 describe-vpcs

# List subnets in a VPC (grab subnet IDs across 2+ AZs)
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-0123456789abcdef0"

# Create a security group for ElastiCache access
aws ec2 create-security-group \
  --group-name elasticache-sg \
  --description "Allow app servers to reach ElastiCache" \
  --vpc-id vpc-0123456789abcdef0

# Allow inbound Redis (6379) only from another security group (your app tier)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123elasticache \
  --protocol tcp --port 6379 \
  --source-group sg-0abc123appservers

# Same, but for Memcached (11211)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123elasticache \
  --protocol tcp --port 11211 \
  --source-group sg-0abc123appservers

# Create a Cache Subnet Group (required before creating any cluster)
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name my-cache-subnet-group \
  --cache-subnet-group-description "Subnets for ElastiCache" \
  --subnet-ids subnet-aaaa1111 subnet-bbbb2222

# List existing subnet groups
aws elasticache describe-cache-subnet-groups

# Delete a subnet group (once no clusters use it)
aws elasticache delete-cache-subnet-group --cache-subnet-group-name my-cache-subnet-group
```

---

## 3. Parameter Groups

```bash
# List available parameter group families (engine + version combos)
aws elasticache describe-cache-parameter-groups

# Create a custom parameter group based on a default family
aws elasticache create-cache-parameter-group \
  --cache-parameter-group-name my-redis7-params \
  --cache-parameter-group-family redis7 \
  --description "Custom params for production Redis"

# Modify a parameter (e.g., eviction policy)
aws elasticache modify-cache-parameter-group \
  --cache-parameter-group-name my-redis7-params \
  --parameter-name-values "ParameterName=maxmemory-policy,ParameterValue=allkeys-lru"

# View current values in a parameter group
aws elasticache describe-cache-parameters \
  --cache-parameter-group-name my-redis7-params

# Reset a parameter group to engine defaults
aws elasticache reset-cache-parameter-group \
  --cache-parameter-group-name my-redis7-params \
  --reset-all-parameters

# Delete a custom parameter group
aws elasticache delete-cache-parameter-group \
  --cache-parameter-group-name my-redis7-params
```

---

## 4. Memcached Cluster Commands

```bash
# Create a Memcached cluster
aws elasticache create-cache-cluster \
  --cache-cluster-id my-memcached-cluster \
  --engine memcached \
  --engine-version 1.6.22 \
  --cache-node-type cache.t4g.micro \
  --num-cache-nodes 2 \
  --cache-subnet-group-name my-cache-subnet-group \
  --security-group-ids sg-0abc123elasticache

# Describe a cluster (check status, get endpoint)
aws elasticache describe-cache-clusters \
  --cache-cluster-id my-memcached-cluster \
  --show-cache-node-info

# Add more nodes to a Memcached cluster (horizontal scaling)
aws elasticache modify-cache-cluster \
  --cache-cluster-id my-memcached-cluster \
  --num-cache-nodes 4 \
  --apply-immediately

# Change node type (vertical scaling) — requires apply-immediately or maintenance window
aws elasticache modify-cache-cluster \
  --cache-cluster-id my-memcached-cluster \
  --cache-node-type cache.m7g.large \
  --apply-immediately

# Delete a Memcached cluster
aws elasticache delete-cache-cluster --cache-cluster-id my-memcached-cluster
```

---

## 5. Redis Replication Group Commands

```bash
# Create a Redis replication group — Cluster Mode DISABLED, Multi-AZ, 1 primary + 2 replicas
aws elasticache create-replication-group \
  --replication-group-id my-redis-group \
  --replication-group-description "Production Redis - session store" \
  --engine redis \
  --engine-version 7.1 \
  --cache-node-type cache.r7g.large \
  --num-cache-clusters 3 \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --cache-subnet-group-name my-cache-subnet-group \
  --security-group-ids sg-0abc123elasticache \
  --cache-parameter-group-name my-redis7-params \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled \
  --auth-token 'MyStrongToken123!'

# Describe a replication group (status, endpoints, member clusters)
aws elasticache describe-replication-groups \
  --replication-group-id my-redis-group

# Get just the Primary + Reader endpoint info
aws elasticache describe-replication-groups \
  --replication-group-id my-redis-group \
  --query "ReplicationGroups[0].{Primary:NodeGroups[0].PrimaryEndpoint,Reader:NodeGroups[0].ReaderEndpoint}"

# Add a read replica
aws elasticache increase-replica-count \
  --replication-group-id my-redis-group \
  --new-replica-count 3 \
  --apply-immediately

# Remove a read replica
aws elasticache decrease-replica-count \
  --replication-group-id my-redis-group \
  --new-replica-count 2 \
  --apply-immediately

# Manually trigger a failover from primary to a specific replica (chaos testing!)
aws elasticache test-failover \
  --replication-group-id my-redis-group \
  --node-group-id 0001

# Modify replication group (e.g., change node type, toggle Multi-AZ)
aws elasticache modify-replication-group \
  --replication-group-id my-redis-group \
  --cache-node-type cache.r7g.xlarge \
  --apply-immediately

# Delete a replication group (optionally take a final snapshot first)
aws elasticache delete-replication-group \
  --replication-group-id my-redis-group \
  --final-snapshot-identifier my-redis-group-final-snapshot
```

---

## 6. Cluster Mode / Sharding Commands

```bash
# Create a Redis replication group with CLUSTER MODE ENABLED (3 shards, 1 replica each)
aws elasticache create-replication-group \
  --replication-group-id my-sharded-redis \
  --replication-group-description "Sharded Redis cluster" \
  --engine redis \
  --engine-version 7.1 \
  --cache-node-type cache.r7g.large \
  --num-node-groups 3 \
  --replicas-per-node-group 1 \
  --automatic-failover-enabled \
  --cache-subnet-group-name my-cache-subnet-group \
  --security-group-ids sg-0abc123elasticache

# Get the Configuration Endpoint (used by cluster-aware clients)
aws elasticache describe-replication-groups \
  --replication-group-id my-sharded-redis \
  --query "ReplicationGroups[0].ConfigurationEndpoint"

# Reshard: increase number of shards online
aws elasticache modify-replication-group-shard-configuration \
  --replication-group-id my-sharded-redis \
  --node-group-count 5 \
  --apply-immediately \
  --resharding-configuration NodeGroupId=0004,PreferredAvailabilityZones=us-east-1a NodeGroupId=0005,PreferredAvailabilityZones=us-east-1b

# Reshard: decrease number of shards (must specify which to retain)
aws elasticache modify-replication-group-shard-configuration \
  --replication-group-id my-sharded-redis \
  --node-group-count 2 \
  --node-groups-to-retain 0001 0002 \
  --apply-immediately

# List shard/node group details
aws elasticache describe-replication-groups \
  --replication-group-id my-sharded-redis \
  --query "ReplicationGroups[0].NodeGroups"
```

---

## 7. Serverless Cache Commands

```bash
# Create a Serverless Redis cache
aws elasticache create-serverless-cache \
  --serverless-cache-name my-serverless-redis \
  --engine redis \
  --security-group-ids sg-0abc123elasticache \
  --subnet-ids subnet-aaaa1111 subnet-bbbb2222 \
  --cache-usage-limits "DataStorage={Maximum=10,Unit=GB},ECPUPerSecond={Maximum=5000}"

# Describe a serverless cache (get endpoint)
aws elasticache describe-serverless-caches \
  --serverless-cache-name my-serverless-redis

# Modify usage limits
aws elasticache modify-serverless-cache \
  --serverless-cache-name my-serverless-redis \
  --cache-usage-limits "DataStorage={Maximum=20,Unit=GB},ECPUPerSecond={Maximum=10000}"

# Create an on-demand snapshot of a serverless cache
aws elasticache create-serverless-cache-snapshot \
  --serverless-cache-snapshot-name my-serverless-snapshot \
  --serverless-cache-name my-serverless-redis

# Delete a serverless cache
aws elasticache delete-serverless-cache --serverless-cache-name my-serverless-redis
```

---

## 8. Snapshots & Backups

```bash
# Create a manual snapshot of a Redis replication group
aws elasticache create-snapshot \
  --replication-group-id my-redis-group \
  --snapshot-name my-manual-snapshot-2026-08-03

# List all snapshots
aws elasticache describe-snapshots

# Describe a specific snapshot
aws elasticache describe-snapshots --snapshot-name my-manual-snapshot-2026-08-03

# Restore a NEW replication group from a snapshot
aws elasticache create-replication-group \
  --replication-group-id my-redis-restored \
  --replication-group-description "Restored from snapshot" \
  --snapshot-name my-manual-snapshot-2026-08-03 \
  --cache-node-type cache.r7g.large \
  --cache-subnet-group-name my-cache-subnet-group \
  --security-group-ids sg-0abc123elasticache

# Copy a snapshot (e.g., to another region for DR)
aws elasticache copy-snapshot \
  --source-snapshot-name my-manual-snapshot-2026-08-03 \
  --target-snapshot-name my-manual-snapshot-copy \
  --target-bucket my-s3-export-bucket

# Delete a snapshot
aws elasticache delete-snapshot --snapshot-name my-manual-snapshot-2026-08-03

# Set automatic backup retention on an existing replication group (7 days, window 05:00-06:00 UTC)
aws elasticache modify-replication-group \
  --replication-group-id my-redis-group \
  --snapshot-retention-limit 7 \
  --snapshot-window "05:00-06:00" \
  --apply-immediately
```

---

## 9. Users & RBAC (Redis 6+)

```bash
# Create a user with specific permissions (read-only on keys prefixed "session:")
aws elasticache create-user \
  --user-id readonly-session-user \
  --user-name readonly-session-user \
  --engine redis \
  --access-string "on ~session:* +get +exists -@all" \
  --authentication-mode Type=password,Passwords='MyUserPassword123!' \
  --no-passwords-required-false 2>/dev/null || true

# (Simplified, common form)
aws elasticache create-user \
  --user-id app-readwrite-user \
  --user-name app-readwrite-user \
  --engine redis \
  --access-string "on ~* +@all" \
  --authentication-mode Type=password,Passwords='SuperSecurePassword456!'

# Create a User Group (collections of users, attached to a replication group)
aws elasticache create-user-group \
  --user-group-id my-app-user-group \
  --engine redis \
  --user-ids default app-readwrite-user readonly-session-user

# Attach a user group to an existing replication group
aws elasticache modify-replication-group \
  --replication-group-id my-redis-group \
  --user-group-ids-to-add my-app-user-group \
  --apply-immediately

# List users
aws elasticache describe-users

# Modify a user's access string
aws elasticache modify-user \
  --user-id readonly-session-user \
  --access-string "on ~session:* ~cache:* +get +exists -@all"

# Delete a user (must remove from all user groups first)
aws elasticache delete-user --user-id readonly-session-user
```

---

## 10. Global Datastore (Cross-Region)

```bash
# Create a Global Datastore from an existing (primary region) replication group
aws elasticache create-global-replication-group \
  --global-replication-group-id-suffix my-global-redis \
  --primary-replication-group-id my-redis-group

# Add a secondary region to the Global Datastore
aws elasticache create-replication-group \
  --replication-group-id my-redis-group-secondary \
  --replication-group-description "Secondary region replica" \
  --global-replication-group-id myglobaldatastore-my-global-redis \
  --cache-subnet-group-name my-cache-subnet-group-us-west-2 \
  --region us-west-2

# Describe the global datastore
aws elasticache describe-global-replication-groups \
  --global-replication-group-id myglobaldatastore-my-global-redis \
  --show-member-info

# Failover: promote a secondary region to be the new primary (disaster recovery)
aws elasticache failover-global-replication-group \
  --global-replication-group-id myglobaldatastore-my-global-redis \
  --primary-region us-west-2 \
  --primary-replication-group-id my-redis-group-secondary

# Delete the global datastore (must remove all secondary regions first)
aws elasticache delete-global-replication-group \
  --global-replication-group-id myglobaldatastore-my-global-redis \
  --retain-primary-replication-group
```

---

## 11. Security & Encryption

```bash
# Rotate/reset the AUTH token on an existing replication group (dual-token rotation, zero downtime)
aws elasticache modify-replication-group \
  --replication-group-id my-redis-group \
  --auth-token 'NewStrongToken789!' \
  --auth-token-update-strategy ROTATE \
  --apply-immediately

# Then finalize rotation (removes the old token entirely)
aws elasticache modify-replication-group \
  --replication-group-id my-redis-group \
  --auth-token 'NewStrongToken789!' \
  --auth-token-update-strategy SET \
  --apply-immediately

# Enable in-transit encryption on an existing cluster (requires new cluster in most engine versions)
# NOTE: transit encryption generally must be enabled AT CREATION for older engine versions;
# newer Redis versions (7+) support enabling it via modify with a maintenance window.
aws elasticache modify-replication-group \
  --replication-group-id my-redis-group \
  --transit-encryption-enabled \
  --transit-encryption-mode preferred \
  --apply-immediately

# Check what KMS key is used for at-rest encryption
aws elasticache describe-replication-groups \
  --replication-group-id my-redis-group \
  --query "ReplicationGroups[0].KmsKeyId"
```

---

## 12. Tagging

```bash
# Add tags to a resource (works for clusters, replication groups, snapshots, etc.)
aws elasticache add-tags-to-resource \
  --resource-name arn:aws:elasticache:us-east-1:123456789012:replicationgroup:my-redis-group \
  --tags Key=Environment,Value=Production Key=Team,Value=Platform

# List tags on a resource
aws elasticache list-tags-for-resource \
  --resource-name arn:aws:elasticache:us-east-1:123456789012:replicationgroup:my-redis-group

# Remove tags
aws elasticache remove-tags-from-resource \
  --resource-name arn:aws:elasticache:us-east-1:123456789012:replicationgroup:my-redis-group \
  --tag-keys Team
```

---

## 13. Monitoring & Events

```bash
# List recent events for a specific cluster/replication group (failovers, maintenance, etc.)
aws elasticache describe-events \
  --source-identifier my-redis-group \
  --source-type replication-group \
  --duration 1440

# Get CloudWatch metric: CPU utilization over the last hour
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name CPUUtilization \
  --dimensions Name=CacheClusterId,Value=my-redis-group-001 \
  --start-time "$(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%S)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%S)" \
  --period 300 \
  --statistics Average

# Get evictions metric (are you losing data you didn't intend to?)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name Evictions \
  --dimensions Name=CacheClusterId,Value=my-redis-group-001 \
  --start-time "$(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%S)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%S)" \
  --period 300 \
  --statistics Sum

# Create a CloudWatch alarm on high memory usage
aws cloudwatch put-metric-alarm \
  --alarm-name redis-high-memory \
  --namespace AWS/ElastiCache \
  --metric-name DatabaseMemoryUsagePercentage \
  --dimensions Name=CacheClusterId,Value=my-redis-group-001 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-alerts-topic
```

---

## 14. Scaling Operations

```bash
# Vertical scaling: change node type on a replication group
aws elasticache modify-replication-group \
  --replication-group-id my-redis-group \
  --cache-node-type cache.r7g.2xlarge \
  --apply-immediately

# Horizontal read scaling: add replicas
aws elasticache increase-replica-count \
  --replication-group-id my-redis-group \
  --new-replica-count 4 \
  --apply-immediately

# Horizontal write/data scaling: reshard (see section 6)
aws elasticache modify-replication-group-shard-configuration \
  --replication-group-id my-sharded-redis \
  --node-group-count 6 \
  --apply-immediately
```

---

## 15. Cleanup / Deletion

```bash
# Delete a Memcached cluster
aws elasticache delete-cache-cluster --cache-cluster-id my-memcached-cluster

# Delete a Redis replication group (with final snapshot)
aws elasticache delete-replication-group \
  --replication-group-id my-redis-group \
  --final-snapshot-identifier final-backup-before-delete

# Delete without a final snapshot (be sure you mean it!)
aws elasticache delete-replication-group \
  --replication-group-id my-redis-group \
  --no-final-snapshot 2>/dev/null || \
aws elasticache delete-replication-group --replication-group-id my-redis-group

# Delete a serverless cache
aws elasticache delete-serverless-cache --serverless-cache-name my-serverless-redis

# Delete subnet group, parameter group, security group (in that order, after clusters are gone)
aws elasticache delete-cache-subnet-group --cache-subnet-group-name my-cache-subnet-group
aws elasticache delete-cache-parameter-group --cache-parameter-group-name my-redis7-params
aws ec2 delete-security-group --group-id sg-0abc123elasticache
```

---

## 16. Engine-Level Client Commands

These aren't AWS CLI commands — they're what you run *after* connecting to the endpoint, using `redis-cli` or the Memcached protocol. Included here because "how do I actually talk to the thing" is half the battle.

### Connecting

```bash
# Redis — no auth
redis-cli -h my-redis-group.xxxxxx.ng.0001.use1.cache.amazonaws.com -p 6379

# Redis — with AUTH token
redis-cli -h my-redis-group.xxxxxx.ng.0001.use1.cache.amazonaws.com -p 6379 -a 'MyStrongToken123!'

# Redis — with TLS enabled
redis-cli -h my-redis-group.xxxxxx.ng.0001.use1.cache.amazonaws.com -p 6379 --tls -a 'MyStrongToken123!'

# Redis — connect as a specific RBAC user
redis-cli -h my-redis-group.xxxxxx.ng.0001.use1.cache.amazonaws.com -p 6379 --user app-readwrite-user --pass 'SuperSecurePassword456!'

# Memcached (using telnet or nc, since there's no official "memcached-cli")
telnet my-memcached-cluster.xxxxxx.cfg.use1.cache.amazonaws.com 11211
```

### Basic Redis Data Operations

```bash
# Strings
SET user:1001 "John Doe"
GET user:1001
SET user:1001:visits 0
INCR user:1001:visits
EXPIRE user:1001 3600        # TTL of 1 hour
TTL user:1001

# Hashes (great for objects)
HSET user:1001:profile name "John" age "29" city "Hyderabad"
HGETALL user:1001:profile
HGET user:1001:profile name

# Lists (queues/stacks)
LPUSH tasks:queue "task1"
RPUSH tasks:queue "task2"
LRANGE tasks:queue 0 -1
LPOP tasks:queue

# Sets (unique collections)
SADD tags:post:55 "aws" "redis" "caching"
SMEMBERS tags:post:55
SISMEMBER tags:post:55 "redis"

# Sorted Sets (leaderboards!)
ZADD leaderboard 100 "playerA"
ZADD leaderboard 250 "playerB"
ZREVRANGE leaderboard 0 9 WITHSCORES   # top 10, highest first
ZRANK leaderboard "playerA"

# Pub/Sub
SUBSCRIBE notifications        # run in one terminal
PUBLISH notifications "hello!" # run in another terminal

# Keys & housekeeping
KEYS user:*          # ⚠️ avoid in production — use SCAN instead
SCAN 0 MATCH user:* COUNT 100
DEL user:1001
FLUSHALL             # ⚠️ wipes EVERYTHING — dev/test only
DBSIZE
INFO memory
CLIENT LIST
SLOWLOG GET 10
```

### Basic Memcached Operations (via `nc`/telnet protocol)

```bash
set greeting 0 900 5
hello
get greeting
delete greeting
stats
```
`set <key> <flags> <exptime_seconds> <bytes>` followed by the value on the next line — Memcached's text protocol is more manual than Redis's.

---

*Back to [README.md](./README.md) · Next: [hands-on-labs.md](./hands-on-labs.md)*
