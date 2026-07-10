# AWS RDS — Hands-On Labs

Twelve progressive labs plus two real-world incident scenarios. Each lab builds on the last — go in order if you're new to RDS. Every lab references the concept explained in [`README.md`](./README.md) and the commands in [`commands-cheatsheet.md`](./commands-cheatsheet.md).

> **Prerequisite for all labs:** a VPC with at least two private subnets across two AZs, and an AWS CLI profile configured with sufficient RDS/EC2/IAM permissions.

## Table of Contents
- [Lab 1: Launch a Baseline Instance in a Private Subnet](#lab-1-launch-a-baseline-instance-in-a-private-subnet)
- [Lab 2: Convert to Multi-AZ and Force a Failover](#lab-2-convert-to-multi-az-and-force-a-failover)
- [Lab 3: Deploy Read Replicas (Same-Region + Cross-Region)](#lab-3-deploy-read-replicas-same-region--cross-region)
- [Lab 4: Promote a Read Replica](#lab-4-promote-a-read-replica)
- [Lab 5: Manual Snapshot Before a Risky Migration](#lab-5-manual-snapshot-before-a-risky-migration)
- [Lab 6: Point-in-Time Recovery Drill](#lab-6-point-in-time-recovery-drill)
- [Lab 7: Build a Custom Parameter Group](#lab-7-build-a-custom-parameter-group)
- [Lab 8: Encrypt an Existing Unencrypted Database](#lab-8-encrypt-an-existing-unencrypted-database)
- [Lab 9: Enforce SSL/TLS In Transit](#lab-9-enforce-ssltls-in-transit)
- [Lab 10: Set Up IAM Database Authentication](#lab-10-set-up-iam-database-authentication)
- [Lab 11: Deploy RDS Proxy in Front of a Lambda Workload](#lab-11-deploy-rds-proxy-in-front-of-a-lambda-workload)
- [Lab 12: Blue/Green Major Version Upgrade](#lab-12-bluegreen-major-version-upgrade)
- [Scenario A: The "Too Many Connections" Outage](#scenario-a-the-too-many-connections-outage)
- [Scenario B: The Catastrophic Accidental Drop](#scenario-b-the-catastrophic-accidental-drop)

---

## Lab 1: Launch a Baseline Instance in a Private Subnet

**Goal:** stand up a secure, non-public MySQL instance.

1. Create a DB subnet group spanning two private subnets in different AZs.
2. Create a security group `db-sg` with **no inbound rules yet**.
3. Launch the instance with `--no-publicly-accessible` (see cheatsheet §1).
4. Verify: `aws rds describe-db-instances --db-instance-identifier prod-mysql-db` → confirm `PubliclyAccessible: false` and `DBInstanceStatus: available`.

**Check yourself:** can you reach the instance from your laptop? You shouldn't be able to — that's correct.

---

## Lab 2: Convert to Multi-AZ and Force a Failover

**Goal:** observe synchronous replication and automatic failover.

1. Modify the instance to enable Multi-AZ (cheatsheet §2).
2. Once `MultiAZ: true`, trigger a failover test:
   ```bash
   aws rds reboot-db-instance --db-instance-identifier prod-mysql-db --force-failover
   ```
3. Watch `DBInstanceStatus` cycle through `rebooting` → `available`.
4. Time the gap in your application logs — expect **30–60 seconds** of disruption, with **zero data loss** on committed transactions.

**Discussion:** why is the standby not queryable during this test? *(Because Multi-AZ standby is passive by design — for readable standbys, you need a Multi-AZ Cluster instead.)*

---

## Lab 3: Deploy Read Replicas (Same-Region + Cross-Region)

**Goal:** offload read traffic and validate replica lag.

1. Confirm automated backups are ON (prerequisite for replicas).
2. Create a same-region replica and a cross-region replica (cheatsheet §3).
3. Write a row to the primary, then immediately query the replica — measure the lag with:
   ```sql
   SHOW SLAVE STATUS\G   -- Seconds_Behind_Master
   ```
   or for PostgreSQL: `SELECT now() - pg_last_xact_replay_timestamp();`
4. Point a reporting/analytics query at the replica endpoint instead of the primary.

---

## Lab 4: Promote a Read Replica

**Goal:** understand the one-way door of promotion.

1. Promote `prod-mysql-replica-1` (cheatsheet §3).
2. Confirm it now accepts writes and is fully detached from the source.
3. Note: this link **cannot be undone** — the replica is now a fully independent primary.

---

## Lab 5: Manual Snapshot Before a Risky Migration

**Goal:** build the "pristine baseline" habit before any schema change.

1. Take a manual snapshot (cheatsheet §4).
2. Run a deliberately risky `ALTER TABLE` on a test table.
3. If something breaks, restore from the snapshot into a **new** instance and compare.
4. Delete the original instance — confirm the manual snapshot **still exists** (unlike automated backups).

---

## Lab 6: Point-in-Time Recovery Drill

**Goal:** practice recovering to the exact second before data loss.

1. Insert 3 rows into a test table, noting the timestamp after each insert.
2. "Accidentally" drop the table.
3. Restore to a timestamp **one second before** the drop (cheatsheet §5).
4. Confirm all 3 rows exist on the new instance, and that the new endpoint is different from the original.
5. Repoint your test application config to the new endpoint.

---

## Lab 7: Build a Custom Parameter Group

**Goal:** feel the difference between dynamic and static parameters.

1. Create `production-mysql8-pg` (cheatsheet §6).
2. Change `max_connections` (dynamic) — confirm it applies with no reboot required.
3. Change `innodb_buffer_pool_size` (static) — confirm status shows **pending-reboot**.
4. Attach the group to your Lab 1 instance — confirm the *entire instance* now shows pending-reboot, even though you haven't changed anything else.
5. Reboot and verify both parameters are live.

---

## Lab 8: Encrypt an Existing Unencrypted Database

**Goal:** work around the "can't encrypt in place" limitation.

1. Confirm your Lab 1 instance is unencrypted.
2. Snapshot → copy the snapshot with `--kms-key-id` → restore a new encrypted instance (cheatsheet §7).
3. Verify: `aws rds describe-db-instances --query 'DBInstances[].StorageEncrypted'`.
4. Cut your application over to the new encrypted endpoint.

---

## Lab 9: Enforce SSL/TLS In Transit

1. Download the RDS CA bundle.
2. Set `rds.force_ssl = 1` (Postgres) or `REQUIRE SSL` per user (MySQL) via your parameter group.
3. Attempt an unencrypted connection — confirm it's rejected.
4. Connect with `--ssl-mode=REQUIRED` (MySQL) or `sslmode=require` (Postgres) — confirm success.

---

## Lab 10: Set Up IAM Database Authentication

1. Enable IAM auth on the instance (cheatsheet §8).
2. Create the mapped DB user (SQL in cheatsheet §8).
3. Attach the `rds-db:connect` IAM policy to your EC2 instance role.
4. From the EC2 instance, generate a token and connect:
   ```bash
   mysql -h prod-mysql-db.abcdefg.ap-south-1.rds.amazonaws.com \
     --user=app_iam_user --password=$(aws rds generate-db-auth-token ...) \
     --enable-cleartext-plugin --ssl-mode=REQUIRED
   ```
5. Wait 16 minutes, try to open a **new** connection with the same token — confirm it's rejected. Confirm an **already-open** session from step 4 is unaffected.

---

## Lab 11: Deploy RDS Proxy in Front of a Lambda Workload

1. Create an RDS Proxy targeting your instance (cheatsheet §9).
2. Deploy a simple Lambda function that queries the DB through the **proxy endpoint**.
3. Load-test with 200+ concurrent Lambda invocations (e.g., via a quick script hitting the function URL in parallel).
4. Compare `DatabaseConnections` on the DB (should stay low/flat) vs. concurrent Lambda executions (should spike) in CloudWatch.
5. Deliberately use a prepared statement or session variable — check the `DatabaseConnectionsCurrentlySessionPinned` proxy metric to see pinning happen.

---

## Lab 12: Blue/Green Major Version Upgrade

1. Create a Blue/Green Deployment targeting a newer major engine version (cheatsheet §11).
2. Run your test suite against the **Green** environment's endpoint.
3. Once validated, execute the switchover.
4. Confirm the switchover completed with minimal downtime and your application automatically resumed against production traffic.

---

## Scenario A: The "Too Many Connections" Outage

**Situation:** A high-traffic retail app in Hyderabad scales out via hundreds of Lambda functions during a flash sale. The app tier crashes with:
```
ERROR 1040 (08004): Too many connections
```
Running standard AWS RDS MySQL.

**Mission:** fix it without upsizing the instance.

<details>
<summary>Solution</summary>

Deploy **Amazon RDS Proxy** in front of the MySQL instance (see [§11](#11-rds-proxy) in README, [§9](#9-rds-proxy) in cheatsheet, and Lab 11).

- Point the Lambda functions at the **proxy endpoint**, not the DB endpoint directly.
- The proxy pools and multiplexes a small number of persistent backend connections across thousands of concurrent Lambda invocations.
- Optionally combine with **IAM Database Authentication** for credential-free connections, offloaded to the proxy layer to protect DB CPU.
- No instance resize needed — this is a connection-management problem, not a compute problem.

**Watch for:** prepared statements or session-level variables causing **connection pinning**, which would silently degrade the fix under peak load — monitor pinning metrics in CloudWatch.
</details>

---

## Scenario B: The Catastrophic Accidental Drop

**Situation:** A junior DBA runs `DROP TABLE customer_orders;` against **Production** PostgreSQL at exactly **16:10:05 PM** instead of staging. Automated Backups are enabled with a **14-day retention window**. The company must not lose any transactions from earlier that day.

**Mission:** what mechanism do you use, what time do you restore to, and what must you tell the app team afterward?

<details>
<summary>Solution</summary>

**Mechanism:** **Point-in-Time Recovery (PITR)** — automated backups continuously ship transaction logs to S3 every 5 minutes, enabling second-level recovery within the 14-day window.

**Restore target:** **16:10:04 PM** (one second *before* the drop) — this captures every transaction up to, but not including, the destructive statement.

**Execution:**
```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-postgres-db \
  --target-db-instance-identifier prod-postgres-recovered \
  --restore-time "2026-07-10T16:10:04.000Z"
```

**Critical follow-up step:** this always provisions a **brand-new instance with a brand-new endpoint** — it never overwrites the existing (now-corrupted) database. The application team **must update their connection string/config to point to the new endpoint** before traffic can resume. Plan a short, coordinated cutover window and communicate the new endpoint clearly before declaring the incident resolved.
</details>
