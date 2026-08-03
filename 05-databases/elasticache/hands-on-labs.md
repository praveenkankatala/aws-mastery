# ElastiCache Hands-On Labs

> Nine labs, ground up. Each one builds on the last, but you can also jump to whichever one you need. Every lab includes: **objective**, **what you'll need**, **step-by-step instructions (Console + CLI)**, **validation**, and **cleanup**. Do the cleanup steps — ElastiCache nodes cost money even when idle.

⚠️ **Cost warning:** Every lab here uses real, billable AWS resources. `cache.t4g.micro` nodes are cheap but not free. Always run the cleanup section when you're done for the day.

---

## Lab 0 — Environment Setup (Do This First, Once)

### Objective
Set up the shared networking foundation every other lab depends on: a security group and a cache subnet group.

### You'll need
- AWS CLI v2 configured (`aws configure`)
- A VPC with at least 2 subnets in different AZs (the **default VPC** works fine for learning)

### Steps

**1. Find your VPC and subnets:**
```bash
aws ec2 describe-vpcs --query "Vpcs[*].{ID:VpcId,CIDR:CidrBlock,Default:IsDefault}" --output table

# Grab subnets from 2 different AZs in that VPC
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<YOUR_VPC_ID>" \
  --query "Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}" --output table
```
Note down two subnet IDs from **different AZs** (e.g., `subnet-aaa111` in `us-east-1a`, `subnet-bbb222` in `us-east-1b`).

**2. Create a security group for the cache:**
```bash
aws ec2 create-security-group \
  --group-name lab-elasticache-sg \
  --description "ElastiCache lab security group" \
  --vpc-id <YOUR_VPC_ID>
```
Save the returned `GroupId` — you'll use it constantly. Call it `<CACHE_SG_ID>` from here on.

**3. Create a security group for your "app server" (an EC2 instance you'll launch in Lab 3):**
```bash
aws ec2 create-security-group \
  --group-name lab-app-sg \
  --description "App server security group" \
  --vpc-id <YOUR_VPC_ID>
```
Save this as `<APP_SG_ID>`.

**4. Allow the app tier to reach the cache tier (not the whole internet):**
```bash
aws ec2 authorize-security-group-ingress \
  --group-id <CACHE_SG_ID> \
  --protocol tcp --port 6379 \
  --source-group <APP_SG_ID>

aws ec2 authorize-security-group-ingress \
  --group-id <CACHE_SG_ID> \
  --protocol tcp --port 11211 \
  --source-group <APP_SG_ID>
```

**5. Allow SSH into your future app server from your own IP (so you can log in):**
```bash
MY_IP=$(curl -s ifconfig.me)
aws ec2 authorize-security-group-ingress \
  --group-id <APP_SG_ID> \
  --protocol tcp --port 22 \
  --cidr ${MY_IP}/32
```

**6. Create the Cache Subnet Group:**
```bash
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name lab-subnet-group \
  --cache-subnet-group-description "Subnet group for lab exercises" \
  --subnet-ids subnet-aaa111 subnet-bbb222
```

### Validation
```bash
aws elasticache describe-cache-subnet-groups --cache-subnet-group-name lab-subnet-group
```
You should see both subnets listed with `Subnets` populated and no errors.

### Cleanup
Nothing to clean up yet — you'll reuse these resources for every lab below. Delete them only in **Lab 9**.

---

## Lab 1 — Your First Cache: Memcached Cluster From Scratch

### Objective
Stand up the simplest possible ElastiCache deployment, connect to it, and prove data is actually being cached.

### You'll need
- Lab 0 completed
- 15–20 minutes (cluster creation takes a few minutes)

### Steps

**1. Create the Memcached cluster:**
```bash
aws elasticache create-cache-cluster \
  --cache-cluster-id lab1-memcached \
  --engine memcached \
  --engine-version 1.6.22 \
  --cache-node-type cache.t4g.micro \
  --num-cache-nodes 2 \
  --cache-subnet-group-name lab-subnet-group \
  --security-group-ids <CACHE_SG_ID>
```

**2. Watch it come online (status goes from `creating` → `available`):**
```bash
watch -n 15 "aws elasticache describe-cache-clusters --cache-cluster-id lab1-memcached --query 'CacheClusters[0].CacheClusterStatus'"
```
This typically takes 3–5 minutes for Memcached. Press `Ctrl+C` once it says `"available"`.

**3. Grab the Configuration Endpoint:**
```bash
aws elasticache describe-cache-clusters \
  --cache-cluster-id lab1-memcached \
  --show-cache-node-info \
  --query "CacheClusters[0].ConfigurationEndpoint"
```

**4. Launch a small EC2 instance in the same VPC/subnet (to connect from, since Memcached has no public access):**
```bash
aws ec2 run-instances \
  --image-id <LATEST_AMAZON_LINUX_2023_AMI_ID> \
  --instance-type t3.micro \
  --key-name <YOUR_KEY_PAIR> \
  --subnet-id subnet-aaa111 \
  --security-group-ids <APP_SG_ID> \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-app-server}]'
```
> 💡 Find the latest AMI with: `aws ec2 describe-images --owners amazon --filters "Name=name,Values=al2023-ami-*-x86_64" "Name=state,Values=available" --query "sort_by(Images,&CreationDate)[-1].ImageId" --output text`

**5. SSH into the instance, install a Memcached client, and connect:**
```bash
ssh -i <YOUR_KEY_PAIR>.pem ec2-user@<INSTANCE_PUBLIC_IP>

# On the instance:
sudo yum install -y memcached
echo -e "set greeting 0 900 5\r\nhello\r" | nc <CONFIG_ENDPOINT> 11211
echo -e "get greeting\r" | nc <CONFIG_ENDPOINT> 11211
```

### Validation
The `get greeting` command should return `hello` back — proof your app server can write to and read from the cluster over the network, through the security group you configured.

### Cleanup
```bash
aws elasticache delete-cache-cluster --cache-cluster-id lab1-memcached
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>
```

---

## Lab 2 — Highly Available Redis (Cluster Mode Disabled + Multi-AZ)

### Objective
Deploy a production-style Redis setup: one primary, two replicas, automatic failover, and understand the difference between the Primary and Reader endpoints.

### Steps

**1. Create a custom parameter group (best practice — don't rely on the default):**
```bash
aws elasticache create-cache-parameter-group \
  --cache-parameter-group-name lab2-redis-params \
  --cache-parameter-group-family redis7 \
  --description "Lab 2 custom Redis params"

aws elasticache modify-cache-parameter-group \
  --cache-parameter-group-name lab2-redis-params \
  --parameter-name-values "ParameterName=maxmemory-policy,ParameterValue=allkeys-lru"
```

**2. Create the replication group:**
```bash
aws elasticache create-replication-group \
  --replication-group-id lab2-redis-ha \
  --replication-group-description "HA Redis - Lab 2" \
  --engine redis \
  --engine-version 7.1 \
  --cache-node-type cache.t4g.small \
  --num-cache-clusters 3 \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --cache-subnet-group-name lab-subnet-group \
  --security-group-ids <CACHE_SG_ID> \
  --cache-parameter-group-name lab2-redis-params
```

**3. Wait for it to become available:**
```bash
watch -n 15 "aws elasticache describe-replication-groups --replication-group-id lab2-redis-ha --query 'ReplicationGroups[0].Status'"
```
This takes roughly 10–15 minutes for a 3-node Redis replication group — grab a coffee.

**4. Get both endpoints:**
```bash
aws elasticache describe-replication-groups \
  --replication-group-id lab2-redis-ha \
  --query "ReplicationGroups[0].NodeGroups[0].{Primary:PrimaryEndpoint.Address,Reader:ReaderEndpoint.Address}"
```

**5. From your EC2 instance (Lab 1's box, or a new one), connect and test:**
```bash
sudo yum install -y https://download1.rpmfusion.org/free/el/rpmfusion-free-release-9.noarch.rpm 2>/dev/null
sudo amazon-linux-extras install -y redis6 2>/dev/null || sudo yum install -y redis6

redis-cli -h <PRIMARY_ENDPOINT> -p 6379
> SET test:key "hello from primary"
> GET test:key
> exit

# Now connect to the READER endpoint (read-only)
redis-cli -h <READER_ENDPOINT> -p 6379
> GET test:key          # should return the same value, proving replication works
> SET test:key "trying to write"   # should ERROR - replicas are read-only
```

### Validation
- Writing to the **Primary Endpoint** succeeds.
- Reading the same key from the **Reader Endpoint** returns the correct value (proves async replication is working).
- Writing to the **Reader Endpoint** fails with a `READONLY` error (proves replicas genuinely can't take writes — this is expected, not a bug).

### Cleanup
Keep this one running — Lab 5 (failover testing) and Lab 6 (backups) reuse it. Clean it up in Lab 9.

---

## Lab 3 — Connecting a Real Application (Python + Node.js Examples)

### Objective
Move past `redis-cli` and connect using an actual application client library — the way you'd do it in production.

### Steps

**1. On your EC2 instance, install Python and the Redis client:**
```bash
sudo yum install -y python3-pip
pip3 install redis
```

**2. Create a simple caching script:**
```python
# cache_demo.py
import redis
import time

r = redis.Redis(
    host="<PRIMARY_ENDPOINT>",
    port=6379,
    decode_responses=True,
    socket_connect_timeout=5,
)

def get_expensive_data(user_id):
    cache_key = f"user:{user_id}:profile"
    cached = r.get(cache_key)
    if cached:
        print("CACHE HIT")
        return cached

    print("CACHE MISS - simulating slow DB call...")
    time.sleep(2)  # pretend this is a slow DB query
    data = f"Profile data for user {user_id}"
    r.setex(cache_key, 60, data)  # cache for 60 seconds
    return data

if __name__ == "__main__":
    print(get_expensive_data(101))  # first call: MISS, takes ~2s
    print(get_expensive_data(101))  # second call: HIT, instant
```

**3. Run it:**
```bash
python3 cache_demo.py
```

**4. Node.js version (if you prefer JS):**
```javascript
// cache_demo.js
const { createClient } = require('redis');

async function main() {
  const client = createClient({ socket: { host: '<PRIMARY_ENDPOINT>', port: 6379 } });
  await client.connect();

  const cacheKey = 'user:101:profile';
  let cached = await client.get(cacheKey);

  if (cached) {
    console.log('CACHE HIT:', cached);
  } else {
    console.log('CACHE MISS - simulating slow DB call...');
    await new Promise(res => setTimeout(res, 2000));
    const data = 'Profile data for user 101';
    await client.setEx(cacheKey, 60, data);
    console.log(data);
  }

  await client.quit();
}

main();
```
```bash
npm install redis
node cache_demo.js
```

### Validation
Run the script twice in a row. The first run should print `CACHE MISS` and take ~2 seconds. The second run (within 60 seconds) should print `CACHE HIT` and return instantly.

### Cleanup
Nothing new was provisioned — this lab only ran client code against Lab 2's cluster.

---

## Lab 4 — Cluster Mode Enabled (Sharding in Practice)

### Objective
Deploy a sharded Redis cluster, connect with a cluster-aware client, and watch keys distribute across shards.

### Steps

**1. Create a sharded replication group (3 shards, 1 replica each):**
```bash
aws elasticache create-replication-group \
  --replication-group-id lab4-sharded-redis \
  --replication-group-description "Sharded Redis - Lab 4" \
  --engine redis \
  --engine-version 7.1 \
  --cache-node-type cache.t4g.small \
  --num-node-groups 3 \
  --replicas-per-node-group 1 \
  --automatic-failover-enabled \
  --cache-subnet-group-name lab-subnet-group \
  --security-group-ids <CACHE_SG_ID>
```

**2. Wait for availability, then grab the Configuration Endpoint:**
```bash
aws elasticache describe-replication-groups \
  --replication-group-id lab4-sharded-redis \
  --query "ReplicationGroups[0].ConfigurationEndpoint"
```

**3. Connect using cluster mode in redis-cli:**
```bash
redis-cli -c -h <CONFIG_ENDPOINT> -p 6379
```
The `-c` flag is critical — it tells `redis-cli` to follow `MOVED` redirects automatically.

**4. Insert a bunch of keys and watch the redirects happen:**
```bash
> SET key1 "a"
> SET key2 "b"
> SET user123 "c"
-> Redirected to slot [xxxx] located at <shard-2-endpoint>:6379
> SET key3 "d"
```
Notice how different keys silently redirect to different shard endpoints — that's the hash slot mechanism in action.

**5. Try a multi-key operation across shards (this will fail without hash tags):**
```bash
> MSET {user1}.name "Alice" {user1}.age "30"
> MGET {user1}.name {user1}.age
```
The `{user1}` hash tag forces both keys onto the same shard, so the multi-key `MGET` succeeds. Try it without the braces and, depending on where each key lands, you may get a `CROSSSLOT` error.

**6. In Python, use a cluster-aware client:**
```python
from redis.cluster import RedisCluster

rc = RedisCluster(host="<CONFIG_ENDPOINT>", port=6379)
rc.set("{user1}.name", "Alice")
print(rc.get("{user1}.name"))
```

### Validation
You should see the `MOVED` redirect messages when connected without `-c`, and clean, transparent access when connected with `-c` or via `RedisCluster`. The `CROSSSLOT` error when testing multi-key ops without hash tags is expected and instructive — not a failure.

### Cleanup
```bash
aws elasticache delete-replication-group --replication-group-id lab4-sharded-redis
```

---

## Lab 5 — Testing Failover (Building Confidence in HA)

### Objective
Deliberately break the primary node in Lab 2's cluster and watch ElastiCache heal itself.

### Steps

**1. Confirm the current primary node:**
```bash
aws elasticache describe-replication-groups \
  --replication-group-id lab2-redis-ha \
  --query "ReplicationGroups[0].NodeGroups[0].NodeGroupMembers[*].{Node:CacheClusterId,Role:CurrentRole}"
```

**2. In one terminal, start a loop writing to the primary so you can see the interruption live:**
```bash
while true; do
  redis-cli -h <PRIMARY_ENDPOINT> -p 6379 SET heartbeat "$(date)" && echo "write ok: $(date)"
  sleep 1
done
```

**3. In another terminal, trigger a manual failover:**
```bash
aws elasticache test-failover \
  --replication-group-id lab2-redis-ha \
  --node-group-id 0001
```

**4. Watch the first terminal.** You'll see a handful of write failures (connection reset / timeout) for a short window — typically under a minute — before writes succeed again automatically, now against the newly-promoted primary.

**5. Confirm the roles have switched:**
```bash
aws elasticache describe-replication-groups \
  --replication-group-id lab2-redis-ha \
  --query "ReplicationGroups[0].NodeGroups[0].NodeGroupMembers[*].{Node:CacheClusterId,Role:CurrentRole}"
```

### Validation
The node that was previously a replica is now `primary`, and your write loop resumed on its own — no manual DNS change, no application redeploy. This is what "automatic failover" actually looks like in practice, and it's worth seeing once so you trust it under real pressure.

### Cleanup
No cleanup needed — the cluster is still healthy and running (just with roles swapped, which is fine).

---

## Lab 6 — Backups: Snapshot, Restore, and Disaster Recovery Drill

### Objective
Take a manual snapshot, delete some data, and restore from the snapshot — the exact motions you'd go through during a real incident.

### Steps

**1. Seed some data you'll want to "recover" later:**
```bash
redis-cli -h <PRIMARY_ENDPOINT> -p 6379 MSET account:1 "1000" account:2 "2500" account:3 "750"
```

**2. Create a manual snapshot:**
```bash
aws elasticache create-snapshot \
  --replication-group-id lab2-redis-ha \
  --snapshot-name lab6-pre-incident-snapshot
```

**3. Wait for it to complete:**
```bash
watch -n 15 "aws elasticache describe-snapshots --snapshot-name lab6-pre-incident-snapshot --query 'Snapshots[0].SnapshotStatus'"
```

**4. Simulate "disaster" — accidentally wipe data:**
```bash
redis-cli -h <PRIMARY_ENDPOINT> -p 6379 FLUSHALL
redis-cli -h <PRIMARY_ENDPOINT> -p 6379 KEYS '*'   # confirms it's empty
```

**5. Restore into a brand-new replication group from the snapshot (you cannot restore in-place onto a live one):**
```bash
aws elasticache create-replication-group \
  --replication-group-id lab6-restored-redis \
  --replication-group-description "Restored from snapshot" \
  --snapshot-name lab6-pre-incident-snapshot \
  --cache-node-type cache.t4g.small \
  --cache-subnet-group-name lab-subnet-group \
  --security-group-ids <CACHE_SG_ID>
```

**6. Once available, verify the data came back:**
```bash
redis-cli -h <RESTORED_PRIMARY_ENDPOINT> -p 6379 MGET account:1 account:2 account:3
```

**7. Enable automatic daily backups going forward, so you don't have to remember to do this manually:**
```bash
aws elasticache modify-replication-group \
  --replication-group-id lab2-redis-ha \
  --snapshot-retention-limit 7 \
  --snapshot-window "05:00-06:00" \
  --apply-immediately
```

### Validation
`account:1`, `account:2`, `account:3` return their original values in the *restored* cluster, proving the snapshot correctly captured your data before the (simulated) disaster.

### Cleanup
```bash
aws elasticache delete-replication-group --replication-group-id lab6-restored-redis
aws elasticache delete-snapshot --snapshot-name lab6-pre-incident-snapshot
```

---

## Lab 7 — Security: Encryption in Transit, at Rest, and RBAC Users

### Objective
Deploy a fully locked-down Redis cluster with TLS, encryption at rest, and role-based users — the setup you'd actually want in production.

### Steps

**1. Create a security-hardened replication group:**
```bash
aws elasticache create-replication-group \
  --replication-group-id lab7-secure-redis \
  --replication-group-description "Fully secured Redis - Lab 7" \
  --engine redis \
  --engine-version 7.1 \
  --cache-node-type cache.t4g.small \
  --num-cache-clusters 2 \
  --automatic-failover-enabled \
  --cache-subnet-group-name lab-subnet-group \
  --security-group-ids <CACHE_SG_ID> \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled \
  --transit-encryption-mode required \
  --auth-token 'Lab7Str0ngT0ken!'
```

**2. Connect using TLS (a plain connection will now be refused):**
```bash
redis-cli -h <PRIMARY_ENDPOINT> -p 6379 --tls -a 'Lab7Str0ngT0ken!'
```

**3. Now layer on RBAC — create a read-only user scoped to one key pattern:**
```bash
aws elasticache create-user \
  --user-id lab7-readonly \
  --user-name lab7-readonly \
  --engine redis \
  --access-string "on ~orders:* +get +exists +scan -@all" \
  --authentication-mode Type=password,Passwords='ReadOnlyPass123!'

aws elasticache create-user-group \
  --user-group-id lab7-usergroup \
  --engine redis \
  --user-ids default lab7-readonly

aws elasticache modify-replication-group \
  --replication-group-id lab7-secure-redis \
  --user-group-ids-to-add lab7-usergroup \
  --apply-immediately
```

**4. Test the scoped user's permissions:**
```bash
redis-cli -h <PRIMARY_ENDPOINT> -p 6379 --tls --user lab7-readonly --pass 'ReadOnlyPass123!'
> SET orders:1 "widget"          # should FAIL - user has no write permission
> GET orders:1                    # should FAIL initially (key doesn't exist yet, but let's fix that)

# Connect as default/admin instead to seed data:
redis-cli -h <PRIMARY_ENDPOINT> -p 6379 --tls -a 'Lab7Str0ngT0ken!'
> SET orders:1 "widget"

# Now back as the readonly user:
redis-cli -h <PRIMARY_ENDPOINT> -p 6379 --tls --user lab7-readonly --pass 'ReadOnlyPass123!'
> GET orders:1        # should SUCCEED
> SET orders:1 "gadget"   # should FAIL - NOPERM error
> GET account:1          # should FAIL - outside this user's key pattern
```

### Validation
The read-only user can `GET` keys under `orders:*` but cannot write, and cannot even read keys outside its permitted pattern. This is RBAC doing its job — least-privilege access enforced by Redis itself, not just by app-layer discipline.

### Cleanup
```bash
aws elasticache modify-replication-group --replication-group-id lab7-secure-redis --user-group-ids-to-remove lab7-usergroup --apply-immediately
aws elasticache delete-user-group --user-group-id lab7-usergroup
aws elasticache delete-user --user-id lab7-readonly
aws elasticache delete-replication-group --replication-group-id lab7-secure-redis
```

---

## Lab 8 — Monitoring: CloudWatch Metrics and Alarms

### Objective
Set up real observability — the thing most tutorials skip, and the thing that actually saves you during an incident.

### Steps

**1. Look at current key metrics for Lab 2's cluster:**
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name CPUUtilization \
  --dimensions Name=CacheClusterId,Value=lab2-redis-ha-001 \
  --start-time "$(date -u -d '-3 hours' +%Y-%m-%dT%H:%M:%S)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%S)" \
  --period 300 \
  --statistics Average Maximum
```

**2. Create an SNS topic to receive alarm notifications:**
```bash
aws sns create-topic --name lab8-cache-alerts
aws sns subscribe --topic-arn <TOPIC_ARN> --protocol email --notification-endpoint you@example.com
```
Check your inbox and confirm the subscription.

**3. Create alarms for the metrics that actually matter:**
```bash
# High memory pressure
aws cloudwatch put-metric-alarm \
  --alarm-name lab8-redis-high-memory \
  --namespace AWS/ElastiCache \
  --metric-name DatabaseMemoryUsagePercentage \
  --dimensions Name=CacheClusterId,Value=lab2-redis-ha-001 \
  --statistic Average --period 300 --threshold 80 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 2 \
  --alarm-actions <TOPIC_ARN>

# Unexpected evictions (data being kicked out you didn't plan for)
aws cloudwatch put-metric-alarm \
  --alarm-name lab8-redis-evictions \
  --namespace AWS/ElastiCache \
  --metric-name Evictions \
  --dimensions Name=CacheClusterId,Value=lab2-redis-ha-001 \
  --statistic Sum --period 300 --threshold 0 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 1 \
  --alarm-actions <TOPIC_ARN>

# Replication lag (secondary falling behind)
aws cloudwatch put-metric-alarm \
  --alarm-name lab8-redis-replication-lag \
  --namespace AWS/ElastiCache \
  --metric-name ReplicationLag \
  --dimensions Name=CacheClusterId,Value=lab2-redis-ha-002 \
  --statistic Average --period 300 --threshold 10 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 2 \
  --alarm-actions <TOPIC_ARN>
```

**4. Trigger the eviction alarm on purpose (fun part):**
```bash
# Set a tiny maxmemory temporarily via parameter group, then flood it with data
redis-cli -h <PRIMARY_ENDPOINT> -p 6379 CONFIG SET maxmemory 1mb
for i in $(seq 1 5000); do
  redis-cli -h <PRIMARY_ENDPOINT> -p 6379 SET "flood:$i" "$(head -c 200 </dev/urandom | base64)" > /dev/null
done
redis-cli -h <PRIMARY_ENDPOINT> -p 6379 INFO stats | grep evicted_keys
```

### Validation
You should get an email/SNS notification once the eviction alarm fires, and `evicted_keys` in `INFO stats` should be a non-zero, growing number.

### Cleanup
```bash
redis-cli -h <PRIMARY_ENDPOINT> -p 6379 CONFIG SET maxmemory 0    # restore to unlimited/parameter-group default
aws cloudwatch delete-alarms --alarm-names lab8-redis-high-memory lab8-redis-evictions lab8-redis-replication-lag
aws sns delete-topic --topic-arn <TOPIC_ARN>
```

---

## Lab 9 — Full Teardown (Do This Last)

### Objective
Delete everything so you're not paying for idle lab resources.

### Steps
```bash
# Replication groups / clusters
aws elasticache delete-replication-group --replication-group-id lab2-redis-ha --no-final-snapshot 2>/dev/null
aws elasticache delete-replication-group --replication-group-id lab2-redis-ha

# EC2 instance(s)
aws ec2 describe-instances --filters "Name=tag:Name,Values=lab-app-server" --query "Reservations[*].Instances[*].InstanceId" --output text
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>

# Parameter group (only after all clusters using it are gone)
aws elasticache delete-cache-parameter-group --cache-parameter-group-name lab2-redis-params

# Subnet group (only after all clusters using it are gone)
aws elasticache delete-cache-subnet-group --cache-subnet-group-name lab-subnet-group

# Security groups (only after nothing references them)
aws ec2 delete-security-group --group-id <CACHE_SG_ID>
aws ec2 delete-security-group --group-id <APP_SG_ID>

# Any remaining snapshots (these ARE billed, even with no live cluster)
aws elasticache describe-snapshots --query "Snapshots[*].SnapshotName" --output text
aws elasticache delete-snapshot --snapshot-name <ANY_REMAINING_SNAPSHOT>
```

### Validation
```bash
aws elasticache describe-replication-groups --query "ReplicationGroups[*].ReplicationGroupId"
aws elasticache describe-cache-clusters --query "CacheClusters[*].CacheClusterId"
aws elasticache describe-snapshots --query "Snapshots[*].SnapshotName"
```
All three should return empty lists. If a `delete-*` command errors with "still in use," it usually means a dependent resource (cluster, snapshot) hasn't finished deleting yet — wait a minute and retry.

---

*Back to [README.md](./README.md) · See also: [commands-cheatsheet.md](./commands-cheatsheet.md) · [troubleshooting.md](./troubleshooting.md)*
