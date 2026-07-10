# EBS — Commands Cheatsheet

Quick-reference commands grouped by topic. Pair with `README.md` for the "why" behind each one.

## Table of Contents
- [Create & Attach a Volume](#create--attach-a-volume)
- [Mount / Format a Volume (Linux)](#mount--format-a-volume-linux)
- [Detach / Move Between Instances](#detach--move-between-instances)
- [Cross-AZ Migration (via Snapshot)](#cross-az-migration-via-snapshot)
- [Instance Store Discovery & Mounting](#instance-store-discovery--mounting)
- [Snapshots](#snapshots)
- [Modify (Resize) a Live Volume](#modify-resize-a-live-volume)
- [Encryption](#encryption)

---

## Create & Attach a Volume

```bash
# Create a gp3 volume in a specific AZ (must match your instance's AZ)
aws ec2 create-volume \
  --availability-zone ap-south-1a \
  --size 20 \
  --volume-type gp3

# Attach it to a running instance
aws ec2 attach-volume \
  --volume-id vol-0123456789example \
  --instance-id i-0123456789abcdef0 \
  --device /dev/sdf
```

---

## Mount / Format a Volume (Linux)

```bash
# 1. Verify the disk is attached (look for xvdf or nvme1n1)
lsblk

# 2. Check if it already has a filesystem (fresh volumes report 'data')
sudo file -s /dev/xvdf

# 3. Create an ext4 filesystem (⚠️ only on a brand-new, empty volume)
sudo mkfs -t ext4 /dev/xvdf

# 4. Create a mount point
sudo mkdir /data

# 5. Mount it
sudo mount /dev/xvdf /data

# 6. Verify
df -h
```

---

## Detach / Move Between Instances

```bash
# On the source instance: unmount cleanly first
sudo umount /data

# Detach via CLI (or Console: Actions -> Detach volume)
aws ec2 detach-volume --volume-id vol-0123456789example

# Wait for state = available, then attach to the second instance
aws ec2 attach-volume \
  --volume-id vol-0123456789example \
  --instance-id i-0987654321fedcba0 \
  --device /dev/sdf

# On the second instance — DO NOT re-run mkfs (that wipes your data!)
sudo mkdir /data
sudo mount /dev/xvdf /data
```

---

## Cross-AZ Migration (via Snapshot)

EBS volumes are AZ-locked, so moving from AZ-1 to AZ-2 requires a snapshot as a bridge.

```bash
# 1. Snapshot the volume in AZ-1
aws ec2 create-snapshot \
  --volume-id vol-0123456789example \
  --description "Migration-Snapshot-to-AZ2"

# 2. Wait for completion
aws ec2 wait snapshot-completed --snapshot-ids snap-0123456789example

# 3. Create a new volume from the snapshot IN THE TARGET AZ
aws ec2 create-volume \
  --availability-zone ap-south-1b \
  --snapshot-id snap-0123456789example \
  --volume-type gp3

# 4. Attach to the instance in AZ-2
aws ec2 attach-volume \
  --volume-id vol-0newexample \
  --instance-id i-instance-in-az2 \
  --device /dev/sdf

# (Optional cleanup once verified)
aws ec2 delete-volume --volume-id vol-0123456789example   # old AZ-1 volume
aws ec2 delete-snapshot --snapshot-id snap-0123456789example
```

---

## Instance Store Discovery & Mounting

```bash
# 1. List block devices — instance store shows up unpartitioned, unmounted
lsblk

# 2. Confirm which device is Instance Store vs EBS (Nitro instances)
sudo nvme list
# Model column shows "Amazon EC2 NVMe Instance Storage" vs "Amazon Elastic Block Store"

# 3. Confirm it's empty
sudo file -s /dev/nvme1n1

# 4. Format
sudo mkfs -t ext4 /dev/nvme1n1

# 5. Mount
sudo mkdir /mnt/scratch-cache
sudo mount /dev/nvme1n1 /mnt/scratch-cache/

# 6. Verify
df -h /mnt/scratch-cache/

# 7. Persist across REBOOTS (not stop/start!) via /etc/fstab
#    nofail prevents boot hangs if the device isn't ready on a new host
echo "/dev/nvme1n1  /mnt/scratch-cache  ext4  defaults,nofail  0  2" | sudo tee -a /etc/fstab
```

### Auto-format-and-mount in User Data (survives stop/start wipes)
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

## Snapshots

```bash
# Create a snapshot
aws ec2 create-snapshot \
  --volume-id vol-0123456789example \
  --description "Nightly-Backup"

# List snapshots for an account
aws ec2 describe-snapshots --owner-ids self

# Copy a snapshot cross-region (DR)
aws ec2 copy-snapshot \
  --source-region ap-south-1 \
  --source-snapshot-id snap-0123456789example \
  --region us-east-1 \
  --description "DR-copy"

# Enable Fast Snapshot Restore in a given AZ
aws ec2 enable-fast-snapshot-restores \
  --availability-zones ap-south-1a \
  --source-snapshot-ids snap-0123456789example

# Pre-warm a restored volume manually (alternative to FSR)
sudo dd if=/dev/xvdf of=/dev/null bs=1M

# Delete a snapshot
aws ec2 delete-snapshot --snapshot-id snap-0123456789example
```

### Data Lifecycle Manager (DLM) — automated policy (conceptual CLI shape)
```bash
aws dlm create-lifecycle-policy \
  --description "Nightly-Prod-Snapshots" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::111122223333:role/AWSDataLifecycleManagerDefaultRole \
  --policy-details file://dlm-policy.json
```

---

## Modify (Resize) a Live Volume

```bash
# 1. Modify size/type/IOPS/throughput without downtime
aws ec2 modify-volume \
  --volume-id vol-0123456789example \
  --size 40 \
  --volume-type gp3

# 2. Check modification progress
aws ec2 describe-volumes-modifications --volume-ids vol-0123456789example
```

### Extend the filesystem inside Linux (required after modify-volume)
```bash
# Confirm the kernel sees the new size
lsblk

# Grow the partition (note the space between device and partition number)
sudo growpart /dev/xvdf 1

# Check filesystem type
df -Th

# If ext4 (Ubuntu/Debian):
sudo resize2fs /dev/xvdf1

# If xfs (Amazon Linux/RHEL) — use the MOUNT PATH, not the device:
sudo xfs_growfs /data

# Verify
df -h /data
```
> ⚠️ **6-hour rule:** once you modify a volume, you must wait ~6 hours before modifying that same volume again.
> ⚠️ **Shrinking:** not supported by Elastic Volumes — create a smaller volume and manually copy/rsync data over.

---

## Encryption

```bash
# Turn on encryption-by-default for a region
aws ec2 enable-ebs-encryption-by-default --region ap-south-1

# Encrypt an existing UNENCRYPTED volume (indirect workflow required):
# 1. Snapshot the unencrypted volume
aws ec2 create-snapshot --volume-id vol-unencrypted-example

# 2. Copy the snapshot with encryption enabled
aws ec2 copy-snapshot \
  --source-region ap-south-1 \
  --source-snapshot-id snap-unencrypted-example \
  --region ap-south-1 \
  --encrypted \
  --kms-key-id alias/aws/ebs \
  --description "Encrypted-copy"

# 3. Create a new volume from the encrypted snapshot copy
aws ec2 create-volume \
  --availability-zone ap-south-1a \
  --snapshot-id snap-encrypted-copy-example \
  --volume-type gp3

# 4. Detach the old unencrypted volume, attach the new encrypted one
aws ec2 detach-volume --volume-id vol-unencrypted-example
aws ec2 attach-volume \
  --volume-id vol-encrypted-new \
  --instance-id i-0123456789abcdef0 \
  --device /dev/sdf
```
