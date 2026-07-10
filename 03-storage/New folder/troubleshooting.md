# EBS — Troubleshooting Guide

Symptom-first playbooks for the most common EBS/Instance Store failure modes.

## Table of Contents
- [Attach Fails: "Volume and Instance Not in Same AZ"](#attach-fails-volume-and-instance-not-in-same-az)
- [Data Corruption After Moving a Volume Between Instances](#data-corruption-after-moving-a-volume-between-instances)
- [New Production DB Restored from Snapshot Is Slow / Hangs Under Load](#new-production-db-restored-from-snapshot-is-slow--hangs-under-load)
- [Application Feels Slow Despite a Fast EBS Volume](#application-feels-slow-despite-a-fast-ebs-volume)
- [Can't Modify a Volume ("Modification Already in Progress")](#cant-modify-a-volume-modification-already-in-progress)
- [Disk Shows New Size in Console but OS Still Reports Old Size](#disk-shows-new-size-in-console-but-os-still-reports-old-size)
- [Instance Store Data Disappeared Unexpectedly](#instance-store-data-disappeared-unexpectedly)
- [Instance Store Mount Missing After Reboot](#instance-store-mount-missing-after-reboot)
- [AMI/Snapshot Storage Still Being Billed](#amisnapshot-storage-still-being-billed)
- [Can't Encrypt an Existing Volume Directly](#cant-encrypt-an-existing-volume-directly)
- [Snapshot Can't Be Shared to Another Account](#snapshot-cant-be-shared-to-another-account)

---

## Attach Fails: "Volume and Instance Not in Same AZ"

**Cause:** EBS volumes are architecturally locked to a single Availability Zone and can only attach to instances in that exact same AZ.

**Fix:**
- If the volume is brand new: delete it and recreate in the correct AZ.
- If the volume already has data you need: use a **snapshot** as a bridge to create a new volume in the target AZ (see `hands-on-labs.md` → Lab 3).

---

## Data Corruption After Moving a Volume Between Instances

**Symptom:** filesystem errors or missing data after detaching a volume from one instance and attaching it to another.

**Cause:** the volume was detached while still mounted and actively being written to, or `mkfs` was accidentally re-run on the second instance (wiping the existing filesystem).

**Fix / prevention:**
```bash
# ALWAYS unmount cleanly before detaching
sudo umount /data

# THEN detach via console/CLI, wait for state = available, THEN attach elsewhere

# On the destination instance — mount only, never re-format an existing filesystem
sudo mkdir /data
sudo mount /dev/xvdf /data
```

---

## New Production DB Restored from Snapshot Is Slow / Hangs Under Load

**Symptom:** a volume built from a snapshot reports `available` immediately, but the first wave of production traffic causes severe latency or connection drops.

**Cause:** **lazy loading / first-touch latency** — data physically lives in S3 until each block is read for the first time, and a flood of simultaneous first-reads under load overwhelms performance.

**Fixes:**
- **Pre-warm** before cutting over traffic:
  ```bash
  sudo dd if=/dev/xvdf of=/dev/null bs=1M
  ```
  (Slow — can take hours on large volumes, bounded by the instance's EBS throughput limit.)
- **Enable Fast Snapshot Restore (FSR)** on the source snapshot/AZ ahead of time so restored volumes deliver full provisioned performance immediately — the right fix for Auto Scaling Groups and fast DR failovers.

---

## Application Feels Slow Despite a Fast EBS Volume

**Symptom:** you provisioned high IOPS/throughput on a `gp3`/`io2` volume, but the app doesn't see the expected performance.

**Cause:** the **EC2 instance type**, not the volume, is the bottleneck. Every instance type has a maximum EBS bandwidth limit (e.g. a `t3.medium` may cap around 312 MiB/s regardless of what the volume can do).

**Fix:** cross-check your instance's max EBS throughput against the volume's provisioned throughput, and size the instance type up if the instance is capping performance the volume is already paying for.

---

## Can't Modify a Volume ("Modification Already in Progress")

**Symptom:** attempting to resize/change type on a volume fails or is rejected.

**Cause:** the **6-hour rule** — once a volume modification is initiated, that specific volume is locked from further modification for a minimum of 6 hours.

**Fix:** wait out the cooldown window before attempting another modification. Plan resize operations in advance rather than iteratively.

**Related:** Elastic Volumes only supports **growing** a volume. To shrink one, you must create a new, smaller volume and manually copy/`rsync` the data — there's no in-place shrink path.

---

## Disk Shows New Size in Console but OS Still Reports Old Size

**Symptom:** `aws ec2 modify-volume` succeeded and the console shows the larger size, but `df -h` inside the instance still reports the old capacity.

**Cause:** AWS expanding the underlying virtual disk does **not** automatically expand the partition or filesystem — that's an OS-level action you must perform explicitly.

**Fix (in order):**
```bash
# 1. Confirm the kernel sees the new size
lsblk

# 2. Grow the partition (note the space before the partition number)
sudo growpart /dev/xvdf 1

# 3. Check filesystem type
df -Th

# 4. Expand the filesystem — command depends on type
sudo resize2fs /dev/xvdf1        # ext4
sudo xfs_growfs /data            # xfs (use the mount path, not the device)

# 5. Verify
df -h /data
```

---

## Instance Store Data Disappeared Unexpectedly

**Symptom:** files that were on `/mnt/scratch-cache` (or similar) are gone after an instance stop/start cycle, hardware failure, or termination — even though the instance is back up and healthy.

**Cause:** this is expected behavior, not a bug. Instance Store is ephemeral:

| Action | Data Fate |
|---|---|
| OS reboot | Survives |
| Stop → Start | **Wiped** — new physical host |
| Terminate | **Wiped** — disks zeroed for the next customer |
| Hardware failure | **Wiped** — no automatic replication |

**Fix:** never store data on Instance Store that you can't afford to lose or regenerate. For anything that must persist, use **EBS** instead, or design the application to self-replicate (e.g. Cassandra/MongoDB cluster-level redundancy) or rebuild from a persistent source (e.g. cache repopulated from the primary DB).

---

## Instance Store Mount Missing After Reboot

**Symptom:** the instance store volume was mounted and working, but after a plain OS reboot, `/mnt/scratch-cache` is empty or the mount point doesn't exist.

**Cause:** mounting a device manually does not persist across reboots unless it's registered in `/etc/fstab`.

**Fix:**
```bash
echo "/dev/nvme1n1  /mnt/scratch-cache  ext4  defaults,nofail  0  2" | sudo tee -a /etc/fstab
```
The `nofail` option is critical — without it, if the instance ever lands on a host where the device name shifts or isn't ready, the OS can hang at boot waiting for a missing device.

**Better long-term fix:** automate format+mount in **User Data** so a fresh/wiped instance store is always reconfigured correctly on every boot, regardless of device path quirks:
```bash
#!/bin/bash
if ! blkid /dev/nvme1n1; then
    mkfs -t ext4 /dev/nvme1n1
fi
mkdir -p /mnt/scratch-cache
mount /dev/nvme1n1 /mnt/scratch-cache
chmod 777 /mnt/scratch-cache
```

---

## AMI/Snapshot Storage Still Being Billed

**Symptom:** you deleted or deregistered something, but EBS/snapshot storage charges keep appearing.

**Cause:** deregistering an **AMI** only removes the launch pointer — the **underlying snapshot** is a separate resource and keeps billing until explicitly deleted.

**Fix:**
```bash
aws ec2 describe-snapshots --owner-ids self \
  --filters "Name=description,Values='*ami-0123456789example*'"
aws ec2 delete-snapshot --snapshot-id snap-0123456789example
```

---

## Can't Encrypt an Existing Volume Directly

**Symptom:** looking for an "encrypt this volume" toggle on a live unencrypted volume — it doesn't exist.

**Cause:** **you cannot encrypt or decrypt a live EBS volume in place.** Encryption state can only change during a snapshot-copy operation.

**Fix — the required workflow:**
1. Snapshot the unencrypted volume.
2. Copy that snapshot with encryption enabled (choose a KMS key).
3. Create a new volume from the encrypted snapshot copy.
4. Detach the old volume, attach the new encrypted one.

---

## Snapshot Can't Be Shared to Another Account

**Symptom:** attempting to share a snapshot (or an AMI built from it) with another AWS account fails or the option is missing.

**Cause:** the snapshot/volume was encrypted with the **AWS Managed Key** (`aws/ebs`), which explicitly does **not** support cross-account sharing.

**Fix:** re-encrypt using a **Customer Managed Key (CMK)** via the snapshot-copy workflow above, and grant the target account's IAM principal permission to use that CMK — only CMK-encrypted resources can be shared across accounts.
