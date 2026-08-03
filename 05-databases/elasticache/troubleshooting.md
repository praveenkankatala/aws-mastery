# ElastiCache Troubleshooting Guide

> Real error messages, why they actually happen, and what to do about them. Organized by category so you can jump straight to your problem. If you hit something not covered here, it's very likely a networking or permissions issue — those two categories account for most real-world ElastiCache pain.

---

## Table of Contents
1. [Connection & Networking Issues](#1-connection--networking-issues)
2. [Authentication & Authorization Errors](#2-authentication--authorization-errors)
3. [Performance & Memory Issues](#3-performance--memory-issues)
4. [Replication & Failover Issues](#4-replication--failover-issues)
5. [Cluster Mode / Sharding Errors](#5-cluster-mode--sharding-errors)
6. [Backup & Restore Problems](#6-backup--restore-problems)
7. [Parameter Group Issues](#7-parameter-group-issues)
8. [Scaling & Modification Errors](#8-scaling--modification-errors)
9. [Client Library Gotchas](#9-client-library-gotchas)
10. [Cost & Billing Surprises](#10-cost--billing-surprises)
11. [Key CloudWatch Metrics to Watch](#11-key-cloudwatch-metrics-to-watch)

---

## 1. Connection & Networking Issues

### ❌ "Could not connect to Redis... Connection timed out"

**Why it happens:** This is, by far, the #1 issue people hit — and it's almost always one of three things:
1. Your security group doesn't allow inbound traffic on port 6379/11211 from where you're connecting.
2. You're trying to connect from **outside the VPC** (ElastiCache has no public endpoint — ever).
3. You're in the right VPC but the wrong subnet/route table, and there's no route between them.

**How to fix it:**
```bash
# 1. Confirm the cluster's security group actually allows your source
aws ec2 describe-security-groups --group-ids <CACHE_SG_ID> --query "SecurityGroups[0].IpPermissions"

# 2. Confirm you're testing FROM an EC2 instance inside the same VPC (not your laptop)
curl -s http://169.254.169.254/latest/meta-data/instance-id   # should return something, if run on an EC2 instance

# 3. Confirm the instance's security group is what you added as the source
aws ec2 describe-instances --instance-ids <INSTANCE_ID> --query "Reservations[0].Instances[0].SecurityGroups"
```
**Fix:** Add an ingress rule that sources from your app's security group (not `0.0.0.0/0`, and not just a CIDR you assume is right):
```bash
aws ec2 authorize-security-group-ingress \
  --group-id <CACHE_SG_ID> --protocol tcp --port 6379 --source-group <APP_SG_ID>
```

### ❌ "Name or service not known" / DNS resolution failure

**Why it happens:** You copied the endpoint before the cluster finished creating, or you're using a stale endpoint after a topology change (e.g., after enabling cluster mode, or after a Multi-AZ failover with an unusual client).

**Fix:**
```bash
aws elasticache describe-replication-groups \
  --replication-group-id <YOUR_GROUP_ID> \
  --query "ReplicationGroups[0].{Status:Status,Primary:NodeGroups[0].PrimaryEndpoint}"
```
Confirm `Status` is `available`, then re-copy the endpoint exactly — no trailing spaces, no typos in the region suffix.

### ❌ Can connect via `redis-cli` from a bastion, but the app (in Lambda/ECS) can't

**Why it happens:** Lambda functions not attached to a VPC **cannot reach ElastiCache at all** — there's no path. Similarly, ECS tasks using `awsvpc` networking need their own ENI in the correct subnet + security group.

**Fix:**
- For Lambda: attach it to the same VPC, subnets, and security group config (with egress to the cache SG).
- For ECS/Fargate: confirm the task definition's network configuration includes the right subnets and security groups.
- Remember: Lambda-in-VPC adds cold-start latency (ENI attachment) — factor that into your design.

---

## 2. Authentication & Authorization Errors

### ❌ `NOAUTH Authentication required.`

**Why it happens:** The cluster has an AUTH token or RBAC enabled, and your client didn't provide credentials.

**Fix:**
```bash
redis-cli -h <ENDPOINT> -p 6379 -a '<YOUR_AUTH_TOKEN>'
# or, in application code:
# redis.Redis(host=..., password="<YOUR_AUTH_TOKEN>")
```

### ❌ `WRONGPASS invalid username-password pair or user is disabled.`

**Why it happens:** Either the password is wrong, or (very common) you're passing a username but the cluster is using simple AUTH (not RBAC) — or vice versa.

**Fix:** Check whether the replication group has `UserGroupIds` set (RBAC) or just an `AuthToken` (simple AUTH):
```bash
aws elasticache describe-replication-groups \
  --replication-group-id <YOUR_GROUP_ID> \
  --query "ReplicationGroups[0].{UserGroups:UserGroupIds,AuthEnabled:AuthTokenEnabled}"
```
If `UserGroups` is empty, don't pass `--user` at all — just `-a <token>`. If it's populated, you must pass both `--user <name>` and `--pass <password>`.

### ❌ `NOPERM this user has no permissions to access one of the keys used as arguments`

**Why it happens:** Working as intended! Your RBAC user's `access-string` doesn't include the key pattern you're touching.

**Fix:** Review and widen the access string:
```bash
aws elasticache describe-users --user-id <USER_ID> --query "Users[0].AccessString"

aws elasticache modify-user \
  --user-id <USER_ID> \
  --access-string "on ~orders:* ~invoices:* +get +set +exists -@all"
```

### ❌ `ERR unknown command` after switching to an RBAC user

**Why it happens:** The user's access string restricts *commands*, not just keys. A user scoped with `+get +exists -@all` genuinely cannot run `SET`, `DEL`, or anything else — that minus sign at the end (`-@all`) denies everything except what's explicitly allowed before it.

**Fix:** Add the specific command categories the user actually needs, e.g. `+@read` for all read commands, or list them individually.

### ❌ Can't rotate the AUTH token without downtime

**Why it happens:** People often set a new token with `--auth-token-update-strategy SET` directly, which immediately invalidates the old token — breaking every currently-connected client at once.

**Fix:** Use the two-phase rotation:
```bash
# Phase 1: ROTATE - both old and new tokens work simultaneously
aws elasticache modify-replication-group \
  --replication-group-id <GROUP_ID> --auth-token '<NEW_TOKEN>' \
  --auth-token-update-strategy ROTATE --apply-immediately

# ... update all your app instances to use the new token, verify they're connected ...

# Phase 2: SET - now safely finalize, old token stops working
aws elasticache modify-replication-group \
  --replication-group-id <GROUP_ID> --auth-token '<NEW_TOKEN>' \
  --auth-token-update-strategy SET --apply-immediately
```

---

## 3. Performance & Memory Issues

### ❌ High `Evictions` metric — data disappearing unexpectedly

**Why it happens:** Your dataset (or working set) has grown larger than the node's available memory, and your `maxmemory-policy` is evicting keys to make room.

**Diagnosis:**
```bash
redis-cli -h <ENDPOINT> -p 6379 INFO memory | grep -E "used_memory_human|maxmemory_human|maxmemory_policy"
redis-cli -h <ENDPOINT> -p 6379 INFO stats | grep evicted_keys
```

**Fix (pick one):**
- Scale up the node type (more RAM).
- Scale out (add shards, if Cluster Mode Enabled).
- Set more aggressive TTLs so keys expire before they need to be evicted.
- If evictions are surprising you because you didn't expect ANY data loss, you may have wanted `noeviction` (accepting write errors instead of silent data loss) — but that trades one problem for another, so choose deliberately.

### ❌ High CPU utilization with seemingly low traffic

**Why it happens:** Common culprits: expensive commands (`KEYS *`, large `SORT`, big `SMEMBERS`/`HGETALL` on huge collections), or a Lua script with an inefficient loop, or too many client connections thrashing.

**Diagnosis:**
```bash
redis-cli -h <ENDPOINT> -p 6379 SLOWLOG GET 10
redis-cli -h <ENDPOINT> -p 6379 CLIENT LIST | wc -l
```

**Fix:**
- Replace `KEYS *` with `SCAN` (cursor-based, non-blocking) everywhere in your codebase.
- Break up giant hashes/sets/lists into smaller ones if a single key is holding tens of thousands of elements.
- Check for a "hot key" — one key getting hammered by every request, saturating a single shard's CPU (this doesn't spread across the cluster even in Cluster Mode Enabled, since one key = one slot = one shard).

### ❌ Latency spikes that come and go

**Why it happens:** Frequently, this is **background persistence** (RDB snapshot / AOF rewrite) briefly consuming CPU/I/O, or a **replica falling behind** and catching up in bursts.

**Diagnosis:**
```bash
redis-cli -h <ENDPOINT> -p 6379 INFO persistence | grep -E "rdb_bgsave_in_progress|aof_rewrite_in_progress"
```

**Fix:** If snapshotting overlaps with peak traffic, move the snapshot window (`--snapshot-window`) to a genuinely low-traffic period.

### ❌ `OOM command not allowed when used memory > 'maxmemory'`

**Why it happens:** Your `maxmemory-policy` is set to `noeviction` (or effectively behaving as one), and the node is full — Redis refuses new writes rather than silently evicting data.

**Fix:** Either free up memory (delete unneeded keys, add TTLs), scale up, or deliberately switch the eviction policy if data loss is acceptable for this use case:
```bash
aws elasticache modify-cache-parameter-group \
  --cache-parameter-group-name <PARAM_GROUP> \
  --parameter-name-values "ParameterName=maxmemory-policy,ParameterValue=allkeys-lru"
```

---

## 4. Replication & Failover Issues

### ❌ Replica stuck in `ReplicationLag` climbing steadily

**Why it happens:** The replica can't keep up with the write rate from the primary — usually because it's an undersized node type, or it's doing its own background work (e.g., its own snapshot) at the same time.

**Diagnosis:**
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache --metric-name ReplicationLag \
  --dimensions Name=CacheClusterId,Value=<REPLICA_NODE_ID> \
  --start-time "$(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%S)" --end-time "$(date -u +%Y-%m-%dT%H:%M:%S)" \
  --period 60 --statistics Maximum
```
**Fix:** Scale the replica (and typically the whole replication group) to a larger node type, or reduce write volume per shard by sharding further.

### ❌ Failover took much longer than expected

**Why it happens:** Failover time depends on DNS propagation and client-side connection pooling/retry behavior, not just the AWS-side promotion (which itself is usually under a minute). Clients that cache DNS aggressively, or don't retry on connection errors, can appear "down" longer than the actual outage.

**Fix:** Use a client library with built-in reconnect/retry logic and reasonable DNS TTL respect. Don't hand-roll a persistent single TCP connection without reconnect logic in production code.

### ❌ `test-failover` command fails: "Test Failover is not currently supported..."

**Why it happens:** `test-failover` requires Multi-AZ to be enabled, and there's a cooldown — you can only test-failover a given node group a limited number of times within a rolling 24-hour window.

**Fix:**
```bash
aws elasticache describe-replication-groups \
  --replication-group-id <GROUP_ID> \
  --query "ReplicationGroups[0].MultiAZ"
```
Confirm it says `enabled`. If you've already tested recently, wait for the cooldown window to reset.

---

## 5. Cluster Mode / Sharding Errors

### ❌ `CROSSSLOT Keys in request don't hash to the same slot`

**Why it happens:** You ran a multi-key command (`MGET`, `MSET`, a transaction, a Lua script touching multiple keys) where the keys land on different shards. Redis Cluster physically cannot execute a single atomic operation across shards.

**Fix:** Use **hash tags** to force related keys onto the same shard:
```bash
# Before (may fail):
MSET user:1:name "Alice" user:1:age "30"

# After (guaranteed same shard):
MSET {user:1}:name "Alice" {user:1}:age "30"
```

### ❌ `MOVED` errors flooding a non-cluster-aware client

**Why it happens:** You're using a plain Redis client (not cluster-aware) against a Cluster Mode Enabled deployment. The client doesn't know how to follow slot redirects.

**Fix:** Switch to a cluster-aware client library (e.g., Python's `redis.cluster.RedisCluster`, Node's `ioredis` with `Cluster` mode, Java's Lettuce/Jedis cluster clients) and connect via the **Configuration Endpoint**, not an individual node endpoint.

### ❌ Resharding operation stuck at "modifying" for a long time

**Why it happens:** Large keyspaces take time to migrate slot-by-slot, and resharding intentionally throttles itself to avoid saturating cluster bandwidth/CPU during the migration.

**Diagnosis:**
```bash
aws elasticache describe-replication-groups \
  --replication-group-id <GROUP_ID> \
  --query "ReplicationGroups[0].Status"
```
**Fix:** This is usually just patience. If it's been many hours with no progress, check CloudWatch for stuck/errored nodes and consider opening an AWS Support case — but don't force-delete mid-reshard.

---

## 6. Backup & Restore Problems

### ❌ Restore creates a NEW cluster instead of restoring in place

**Why it happens:** This is expected behavior, not a bug — ElastiCache snapshots always restore into a **new** replication group. There's no "restore in place" operation.

**Fix:** Plan for this: restore to a new group name, validate the data, then cut your application over (update connection strings/env vars), and only delete the old (broken) group afterward.

### ❌ `Snapshotting is not supported for this replication group`

**Why it happens:** Snapshots aren't available on `cache.t1`/`cache.t2` micro node types in some configurations, or the engine version predates snapshot support, or you're using Memcached (which has **no persistence or snapshot capability at all**).

**Fix:** For Memcached, there's no snapshot feature — build backup/recovery into your application layer instead (e.g., re-populate from the source of truth). For Redis, move to a supported node type.

### ❌ Cross-region snapshot copy fails with a KMS error

**Why it happens:** If the source snapshot is encrypted with a customer-managed KMS key, that key doesn't automatically exist (or have the right permissions) in the destination region.

**Fix:** Create/replicate an appropriate KMS key in the destination region and grant the ElastiCache service permission to use it before retrying the copy.

---

## 7. Parameter Group Issues

### ❌ Changed a parameter, but the cluster's behavior didn't change

**Why it happens:** Some parameters are **dynamic** (apply immediately) and others require a **reboot** to take effect. You may have changed a static parameter and not rebooted.

**Diagnosis:**
```bash
aws elasticache describe-cache-parameters \
  --cache-parameter-group-name <PARAM_GROUP> \
  --query "Parameters[?ParameterName=='<YOUR_PARAM>'].{Value:ParameterValue,IsModifiable:IsModifiable,Source:Source}"
```
**Fix:** If needed, reboot the affected node(s) during a maintenance window:
```bash
aws elasticache reboot-cache-cluster --cache-cluster-id <NODE_ID> --cache-node-ids-to-reboot 0001
```

### ❌ Can't modify the default parameter group

**Why it happens:** AWS-provided default parameter groups (e.g., `default.redis7`) are **read-only** by design — you can't tune them.

**Fix:** Create a custom parameter group derived from the same family, tweak that, and attach it to your cluster.

---

## 8. Scaling & Modification Errors

### ❌ `InvalidParameterCombination: Cannot modify node type while a modification is in progress`

**Why it happens:** You issued a second `modify-*` command while the first one was still applying.

**Fix:** Check status first, then wait:
```bash
aws elasticache describe-replication-groups --replication-group-id <GROUP_ID> --query "ReplicationGroups[0].Status"
```
Only issue the next change once `Status` returns to `available`.

### ❌ Vertical scaling causes a brief connection blip

**Why it happens:** Changing node type isn't truly "live" — under the hood, ElastiCache provisions new nodes of the target size and fails over to them, which briefly interrupts connections (similar to a failover event).

**Fix:** Schedule node type changes during low-traffic windows, and make sure your client has reconnect logic (same guidance as failover, above).

### ❌ Decreasing shard count fails: "must specify node groups to retain"

**Why it happens:** When shrinking a Cluster Mode Enabled deployment, ElastiCache needs to know explicitly which shards survive so it can migrate data out of the ones being removed.

**Fix:**
```bash
aws elasticache modify-replication-group-shard-configuration \
  --replication-group-id <GROUP_ID> \
  --node-group-count 2 \
  --node-groups-to-retain 0001 0002 \
  --apply-immediately
```

---

## 9. Client Library Gotchas

### ❌ App works fine in testing, falls over under real load ("too many connections")

**Why it happens:** A new TCP connection per-request (instead of a connection pool) exhausts the `maxclients` limit on the node, especially common in serverless/Lambda architectures where each invocation might open a fresh connection.

**Fix:** Use a proper connection pool sized sensibly for your concurrency, and for Lambda, consider a connection-reuse pattern across warm invocations, or a proxy layer.

### ❌ App only ever talks to the Primary, never uses replicas (wasted read capacity)

**Why it happens:** Many client libraries default to a single endpoint. If you only configured the Primary Endpoint, you're not using the Reader Endpoint (or the individual replica endpoints) at all.

**Fix:** For read-heavy workloads, explicitly route read queries to the Reader Endpoint (Cluster Mode Disabled) or configure your cluster client's read-from-replica option (Cluster Mode Enabled — many clients support `readonly`/`read-from-replica` flags).

### ❌ Cluster client throws errors immediately after a reshard

**Why it happens:** Some client libraries cache the slot-to-node mapping and don't refresh it automatically after a topology change.

**Fix:** Ensure your client library version supports automatic slot-map refresh on `MOVED` errors, or manually trigger a topology refresh after known reshard events.

---

## 10. Cost & Billing Surprises

### ❌ "Why am I still being charged after I deleted my cluster?"

**Why it happens:** Manual **snapshots persist independently** of the cluster that created them, and are billed separately (per GB stored). Deleting a replication group does not delete its manual snapshots.

**Fix:**
```bash
aws elasticache describe-snapshots --query "Snapshots[*].{Name:SnapshotName,Size:DataTiering}"
aws elasticache delete-snapshot --snapshot-name <NAME>
```

### ❌ Serverless bill much higher than expected

**Why it happens:** No `CacheUsageLimits` were set, so the cache scaled up freely with traffic/data growth, and ECPU consumption from an inefficient access pattern (e.g., large `SCAN`s, big values) can add up fast.

**Fix:** Set explicit usage limits:
```bash
aws elasticache modify-serverless-cache \
  --serverless-cache-name <NAME> \
  --cache-usage-limits "DataStorage={Maximum=10,Unit=GB},ECPUPerSecond={Maximum=5000}"
```
And set a CloudWatch billing alarm as a safety net regardless.

---

## 11. Key CloudWatch Metrics to Watch

A quick-reference table for "what number means trouble":

| Metric | Watch for | Likely meaning |
|---|---|---|
| `CPUUtilization` | Sustained > 70–80% | Undersized node, hot key, or expensive commands |
| `DatabaseMemoryUsagePercentage` | > 80–90% | Approaching eviction/OOM territory |
| `Evictions` | Any sustained non-zero value (unless intentional) | Data being kicked out due to memory pressure |
| `CurrConnections` | Rapid growth without matching traffic growth | Connection leak — pooling misconfigured |
| `ReplicationLag` | Growing over time, not just spiky | Replica can't keep up — undersized or overloaded |
| `SwapUsage` | Any non-zero value | Node is under severe memory pressure — treat as urgent |
| `NetworkBytesIn/Out` | Sudden unexplained spike | Possible hot key, misbehaving client, or traffic surge |
| `NewConnections` | High rate | Clients not reusing/pooling connections |
| `EngineCPUUtilization` (vs. `CPUUtilization`) | High while overall CPU looks fine | The engine thread itself is the bottleneck, not the OS overall (relevant since Redis is largely single-threaded per shard) |

---

## Still Stuck?

- Re-read [README.md § Core Features Deep-Dive](./README.md#5-core-features--deep-dive) — a lot of "bugs" are actually expected behavior once you understand the underlying mechanism (replication lag, CROSSSLOT, read-only replicas, etc.).
- Run through the relevant lab again in [hands-on-labs.md](./hands-on-labs.md) — reproducing the working case often reveals what's different in your broken one.
- Check `aws elasticache describe-events` for the resource — AWS often logs exactly what happened (maintenance, failover, node replacement) around the time things went sideways.

---

*Back to [README.md](./README.md) · [commands-cheatsheet.md](./commands-cheatsheet.md) · [hands-on-labs.md](./hands-on-labs.md)*
