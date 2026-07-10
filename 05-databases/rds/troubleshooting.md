# AWS RDS — Troubleshooting Guide

Common production incidents, their root cause, and the fix. Cross-referenced with [`README.md`](./README.md) for the underlying concept.

## Table of Contents
- [1. `ERROR 1040: Too many connections`](#1-error-1040-too-many-connections)
- [2. Application reads stale data right after a write](#2-application-reads-stale-data-right-after-a-write)
- [3. Instance stuck in "pending-reboot"](#3-instance-stuck-in-pending-reboot)
- [4. Can't enable encryption on an existing DB](#4-cant-enable-encryption-on-an-existing-db)
- [5. Restore operation fails / can't find the data](#5-restore-operation-fails--cant-find-the-data)
- [6. Backups vanished after deleting an instance](#6-backups-vanished-after-deleting-an-instance)
- [7. RDS Proxy isn't actually reducing connection count](#7-rds-proxy-isnt-actually-reducing-connection-count)
- [8. IAM auth token rejected](#8-iam-auth-token-rejected)
- [9. Multi-AZ failover took longer than expected](#9-multi-az-failover-took-longer-than-expected)
- [10. Storage modification is "stuck"](#10-storage-modification-is-stuck)
- [11. Can't SSH into the RDS instance](#11-cant-ssh-into-the-rds-instance)
- [12. Major version upgrade breaks the application](#12-major-version-upgrade-breaks-the-application)
- [Incident Playbooks](#incident-playbooks)

---

## 1. `ERROR 1040: Too many connections`

**Symptom:** app tier crashes under load (often Lambda-driven spikes) with the DB refusing new connections.

**Root cause:** each engine has a hard connection ceiling tied to instance RAM; every short-lived Lambda invocation opens its own direct connection (~8MB memory + CPU context-switch cost each).

**Fix:**
- Deploy **RDS Proxy** in front of the instance and repoint the app to the proxy endpoint. It pools/multiplexes a small number of persistent backend connections across many app-side sessions.
- Do **not** default to resizing the instance — that's expensive and treats the symptom, not the cause.
- If pinning is neutralizing the pool (see #7), fix the pinning trigger first.

---

## 2. Application reads stale data right after a write

**Symptom:** a record just inserted/updated on the primary doesn't appear (or shows old values) when read moments later.

**Root cause:** **replica lag** — Read Replicas use **asynchronous** replication, so there's always a small time gap between a primary write and its appearance on the replica.

**Fix:**
- For read-your-own-write consistency, route that specific read back to the **primary**, not the replica.
- Monitor `ReplicaLag` in CloudWatch and alert if it exceeds your app's tolerance.
- Design the app to expect eventual consistency on replica reads by default.

---

## 3. Instance stuck in "pending-reboot"

**Symptom:** you changed a parameter, but it hasn't taken effect, and the console shows "pending reboot."

**Root cause:** either the parameter is **static** (requires reboot by design), or you attached a **new parameter group entirely** — which always forces pending-reboot status regardless of whether individual parameters are dynamic or static.

**Fix:** schedule a maintenance window and run `aws rds reboot-db-instance`. For zero-surprise changes, only touch dynamic parameters during business hours; batch static-parameter changes into a planned reboot.

---

## 4. Can't enable encryption on an existing DB

**Symptom:** the "Enable Encryption" toggle is greyed out or the API rejects `--storage-encrypted` on `modify-db-instance`.

**Root cause:** encryption at rest can only be set **at creation time** — AWS does not support in-place encryption toggling.

**Fix:** snapshot → copy the snapshot with `--kms-key-id` set → restore a new instance from the encrypted copy → cut the application over to the new endpoint → decommission the old unencrypted instance.

---

## 5. Restore operation fails / can't find the data

**Symptom:** after a PITR or snapshot restore, the app can't connect, or people assume the restore "did nothing."

**Root cause:** restores **never overwrite** an existing instance. Every restore provisions a **brand-new instance with a brand-new endpoint** — the data absolutely is there, but the application is still pointed at the old endpoint.

**Fix:** always update the app's connection string/config to the new endpoint immediately after any restore, and treat that as a mandatory last step of the runbook, not an afterthought.

---

## 6. Backups vanished after deleting an instance

**Symptom:** team deletes an old instance to save cost, later needs a backup from it — and finds nothing.

**Root cause:** **Automated backups are deleted along with the instance by default** unless you explicitly choose to retain a final snapshot on deletion.

**Fix:** before deleting any instance, either take a **manual snapshot** (never expires, survives deletion) or use the "retain automated backups" option during the delete workflow. Make this a required step in your decommissioning checklist.

---

## 7. RDS Proxy isn't actually reducing connection count

**Symptom:** you deployed RDS Proxy, but `DatabaseConnections` on the underlying instance is still high.

**Root cause:** **Connection Pinning** — prepared statements, session-level variable changes (e.g., altering time zone mid-session), or temporary tables lock a proxy session to one specific backend connection, disabling multiplexing for that session.

**Fix:** audit your driver/ORM for prepared-statement behavior and avoid session-level mutations where possible. Monitor `DatabaseConnectionsCurrentlySessionPinned` in CloudWatch to quantify how much pinning is occurring.

---

## 8. IAM auth token rejected

**Symptom:** `Access denied for user` even though the IAM policy looks correct.

**Root cause checklist:**
- Token is older than **15 minutes** (regenerate — but note this only affects the initial connection, not already-open sessions).
- `iam_database_authentication_enabled` isn't actually set on the instance.
- The DB user wasn't mapped correctly (`GRANT rds_iam` for Postgres / `AWSAuthenticationPlugin` for MySQL missing).
- The IAM policy's resource ARN uses the wrong **DB Resource ID** (`dbi-xxxxxx`, not the instance identifier) or wrong DB username.

**Fix:** verify each item in order above; regenerate the token last, since an expired token is the most common false lead when the real issue is a missing IAM policy or DB-side mapping.

---

## 9. Multi-AZ failover took longer than expected

**Symptom:** failover exceeded the expected 30–60 second window.

**Root cause:** classic Multi-AZ Instance failover time depends on DNS propagation and connection retry logic in the app/driver; heavy transaction replay at failover time can also extend it.

**Fix:** for tighter failover SLAs, consider **Multi-AZ Cluster** (semi-synchronous quorum, failover under ~35s) or add **RDS Proxy**, which holds connections in a queue during failover instead of letting the app's connection drop and retry cold.

---

## 10. Storage modification is "stuck"

**Symptom:** you tried to resize storage or change storage type again, and the API rejects the request.

**Root cause:** changing storage type or scaling disk space triggers a background **optimization process that can take several hours**, during which further storage modifications are blocked.

**Fix:** plan storage changes during low-traffic windows and avoid stacking multiple storage modifications back-to-back; check `DBInstanceStatus` / `PendingModifiedValues` before attempting another change.

---

## 11. Can't SSH into the RDS instance

**Symptom:** need OS-level access to install an agent or inspect the filesystem — not possible on standard RDS.

**Root cause:** standard RDS deliberately blocks OS/root access as part of the managed-service model.

**Fix:** if OS access is a genuine, vendor-mandated requirement (common for legacy Oracle/SQL Server ERPs), migrate that specific workload to **RDS Custom**, which grants root access with an AWS-monitored guardrail — rather than falling back to fully self-managed EC2 for everything.

---

## 12. Major version upgrade breaks the application

**Symptom:** app errors after an engine major-version bump (e.g., PostgreSQL 14 → 15).

**Root cause:** major version upgrades include structural/behavioral changes and are **never applied automatically** for exactly this reason — something in your query patterns, extensions, or driver assumptions broke compatibility.

**Fix:** always validate major upgrades via a **Blue/Green Deployment** first — upgrade and fully test the "Green" copy against your real test suite before switching production traffic over, rather than upgrading in place.

---

## Incident Playbooks

### Playbook: Sudden connection exhaustion under load
1. Check `DatabaseConnections` in CloudWatch — confirm it's maxed.
2. If Lambda/serverless-driven → deploy RDS Proxy immediately, repoint app.
3. Check for connection pinning if a proxy is already in place.
4. Post-incident: add `DatabaseConnections` alerting at 80% of max before it becomes an outage.

### Playbook: Accidental destructive DML/DDL in production
1. Freeze further writes to the affected table if possible.
2. Identify the exact timestamp of the destructive statement (query logs / Performance Insights).
3. Kick off PITR targeting **one second before** that timestamp.
4. Validate data on the new instance before cutover.
5. Repoint the application to the new endpoint; communicate the cutover window to stakeholders.
6. Post-incident: review IAM/DB permissions — should this user have had production write access at all?
