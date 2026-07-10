# Troubleshooting: EFS & FSx

Common failure points from the labs above, why they happen, and how to fix them.

---

## Amazon EFS

### 1. `mount.nfs4: Connection timed out`

**Cause:** The EFS security group isn't allowing inbound NFS (port 2049) from your EC2 instance's security group or IP range.

**Fix:**
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-<efs-sg-id> \
  --protocol tcp --port 2049 \
  --source-group sg-<ec2-sg-id>
```
Also confirm the instance and the mount target's subnet don't have a Network ACL blocking port 2049 in either direction.

### 2. `mount: unknown filesystem type 'efs'`

**Cause:** `amazon-efs-utils` isn't installed, so the `-t efs` mount type isn't recognized.

**Fix:**
```bash
sudo dnf install -y amazon-efs-utils
# or, on Ubuntu/Debian:
sudo apt-get install -y amazon-efs-utils
```

### 3. `Failed to resolve "fs-xxxx.efs.<region>.amazonaws.com"`

**Cause:** DNS resolution for the EFS mount target isn't working — common when mounting from a peered VPC, a different account, or on-premises over Direct Connect/VPN.

**Fix:** Either enable `enableDnsSupport` / `enableDnsHostnames` on the VPC, set up private hosted zone / Route 53 resolver rules for cross-VPC DNS, or mount directly using the mount target's **IP address** instead of the DNS name:
```bash
sudo mount -t nfs4 -o nfsvers=4.1 10.0.1.10:/ /mnt/efs
```

### 4. `Permission denied` when writing to `/mnt/efs`

**Cause:** POSIX file permissions on the EFS root, or the mount is using an **EFS Access Point** with a restrictive POSIX user/group/permissions configuration.

**Fix:**
```bash
sudo chmod 777 /mnt/efs        # quick test only — scope this down for real workloads
```
For access points, check the `PosixUser` and `RootDirectory > CreationInfo` settings match the UID/GID your application runs as.

### 5. Mount succeeds but writes are unexpectedly slow

**Cause:** Usually one of:
- **Performance mode** mismatch — General Purpose has an operations/sec ceiling; very high parallel small-file operations may need **Max I/O**.
- **Provisioned throughput** set too low, or on **Bursting mode** the burst credit balance is depleted.

**Fix:** Check `CloudWatch` metrics `PermittedThroughput` and `BurstCreditBalance`; switch to **Elastic throughput mode** to remove manual tuning entirely.

### 6. Files not moving to IA/Archive as expected

**Cause:** Lifecycle policies only trigger after the configured "last accessed" window (e.g. 30/90 days) — a file that was touched (even just read/stat'd) resets the clock. Lifecycle transitions also run on a periodic internal schedule, not instantly at the exact day boundary.

**Fix:** Confirm the policy with:
```bash
aws efs describe-lifecycle-configuration --file-system-id fs-xxxx
```

---

## Amazon FSx (for Lustre)

### 1. `mount.lustre: … Connection refused` or hangs

**Cause:** Security group isn't allowing inbound **TCP 988** (and the ephemeral port range Lustre also uses) from the client's security group.

**Fix:** Add an inbound rule for port 988 (and 1021–1023 if using older Lustre clients) from the EC2 security group.

### 2. `mount.lustre: mount … No such device`

**Cause:** The `lustre-client` package isn't installed, or its version doesn't match the FSx server's Lustre version.

**Fix:**
```bash
sudo dnf install -y lustre-client
```
Check the FSx console for the exact supported client version if the mount still fails after install — Amazon Linux 2023's default repo build usually matches, but custom AMIs may need the version pinned manually.

### 3. `input.txt` from S3 doesn't show up under `/mnt/fsx`

**Cause:** The **Data Repository Association / Import path** wasn't set at file-system creation, or auto-import wasn't configured, so metadata was never linked.

**Fix:** Verify the association:
```bash
aws fsx describe-data-repository-associations --file-system-id fs-xxxx
```
If missing, you'll need to create a Data Repository Association (or recreate the file system with the import path set, for older Scratch file systems that don't support adding one after creation).

### 4. `output.txt` written locally never appears back in S3

**Cause:** FSx doesn't auto-export by default (unless auto-export is enabled on the association) — you must run an explicit export task.

**Fix:**
```bash
aws fsx create-data-repository-task \
  --file-system-id fs-xxxx \
  --type EXPORT_TO_REPOSITORY \
  --paths output.txt \
  --report Enabled=true,Format=REPORT_CSV_20191124,Path=s3://your-bucket/reports/

aws fsx describe-data-repository-tasks --filters Name=file-system-id,Values=fs-xxxx
```
Check the task's `Lifecycle` field — it should move from `PENDING` → `EXECUTING` → `SUCCEEDED`. A `FAILED` status will include a `FailureDetails.Message` you can act on.

### 5. Scratch file system data disappeared after a hardware failure

**Cause:** **Scratch** deployment type has **no replication** — it's designed for temporary, easily-reproducible data only, by design.

**Fix:** Use **Persistent** deployment type (SSD or HDD) for anything that needs durability guarantees and automatic failover.

---

## General AWS Networking Checks (Both Services)

```bash
# Confirm the client instance and the file system are in the same VPC
aws ec2 describe-instances --instance-ids i-xxxx --query 'Reservations[].Instances[].VpcId'
aws efs describe-mount-targets --file-system-id fs-xxxx --query 'MountTargets[].VpcId'

# Test raw TCP reachability before blaming the mount command
nc -zv <target-ip-or-dns> 2049   # EFS
nc -zv <target-ip-or-dns> 988    # FSx for Lustre
```

If `nc` can't even open the port, the problem is **networking** (security group/NACL/route table) — not the mount command itself. Fix networking first, then retry the mount.

---

## Cost / Billing Gotchas

| Symptom | Likely Cause |
|---|---|
| Unexpected EFS bill spike | Files transitioned to IA/Archive and are now being **read frequently**, triggering per-GB access fees — consider moving hot data back to Standard |
| Unexpected FSx bill | Storage capacity is billed continuously once provisioned, **regardless of usage** — delete unused Scratch file systems promptly after a lab/demo |
| Data transfer charges | Cross-AZ traffic between an EC2 instance and a mount target/file server in a **different** AZ incurs standard cross-AZ data transfer charges |
