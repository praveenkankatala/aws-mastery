# EBS — Hands-On Labs

Guided, step-by-step demos you can run in your own AWS account. Each lab builds practical muscle memory for a concept in `README.md`.

## Table of Contents
- [Lab 1: Attach an EBS Volume to an EC2 Instance](#lab-1-attach-an-ebs-volume-to-an-ec2-instance)
- [Lab 2: Move an EBS Volume to a Second Instance](#lab-2-move-an-ebs-volume-to-a-second-instance)
- [Lab 3: Migrate an EBS Volume from AZ-1 to AZ-2](#lab-3-migrate-an-ebs-volume-from-az-1-to-az-2)
- [Lab 4: Discover and Mount an Instance Store Volume](#lab-4-discover-and-mount-an-instance-store-volume)
- [Lab 5: Prove the Instance Store Ephemeral Lifecycle](#lab-5-prove-the-instance-store-ephemeral-lifecycle)
- [Lab 6: Resize a Live Volume with Zero Downtime](#lab-6-resize-a-live-volume-with-zero-downtime)
- [Lab 7: Encrypt an Existing Unencrypted Volume](#lab-7-encrypt-an-existing-unencrypted-volume)

---

## Lab 1: Attach an EBS Volume to an EC2 Instance

**Goal:** create a new data disk and mount it to a running server in the same AZ.

### Step 1 — Create the volume
1. **EC2 → Elastic Block Store → Volumes → Create Volume**.
2. Choose `gp3`, size `20 GiB`.
3. **Critical:** set the **Availability Zone** to match your target instance exactly (e.g. `ap-south-1a`).
4. Click **Create volume**.

### Step 2 — Attach it
1. Select the new volume → **Actions → Attach volume**.
2. Choose your running instance in the **Instance** field.
3. Leave the default device name (e.g. `/dev/sdf`).
4. Click **Attach volume**.

### Step 3 — Mount it inside Linux (SSH required)
```bash
# 1. Verify the disk is visible (look for xvdf or nvme1n1)
lsblk

# 2. Confirm it's a fresh, empty disk
sudo file -s /dev/xvdf

# 3. Create an ext4 filesystem
sudo mkfs -t ext4 /dev/xvdf

# 4. Create a mount point and mount it
sudo mkdir /data
sudo mount /dev/xvdf /data

# 5. Verify
df -h
```

---

## Lab 2: Move an EBS Volume to a Second Instance

**Goal:** cleanly detach a volume from one instance and reattach it to another (standard EBS is single-attach unless using io2 Multi-Attach).

### Step 1 — Detach cleanly from the first instance
```bash
# On the FIRST instance — unmount before detaching
sudo umount /data
```
Then in the console: **Volumes → select volume → Actions → Detach volume**. Wait for state = `available`.

### Step 2 — Attach to the second instance
1. Select the same volume → **Actions → Attach volume**.
2. Choose the **second** EC2 instance (must be in the same AZ).
3. Click **Attach volume**.

### Step 3 — Mount on the second instance
```bash
# Filesystem already exists from Lab 1 — DO NOT run mkfs again (it wipes data!)
sudo mkdir /data
sudo mount /dev/xvdf /data
# Your data from the first instance should now be visible here
ls -la /data
```

---

## Lab 3: Migrate an EBS Volume from AZ-1 to AZ-2

**Goal:** move data across Availability Zones using a snapshot as the bridge (EBS volumes can't attach cross-AZ directly).

### Step 1 — Snapshot the volume in AZ-1
1. **Volumes** → select the live volume in `ap-south-1a`.
2. **Actions → Create snapshot** → description `Migration-Snapshot-to-AZ2`.
3. Go to **Snapshots** and wait for status `pending → completed`.

### Step 2 — Create a new volume in AZ-2 from the snapshot
1. Select the completed snapshot → **Actions → Create volume from snapshot**.
2. **The critical change:** set **Availability Zone** to `ap-south-1b`.
3. Click **Create volume**.

### Step 3 — Attach and mount in AZ-2
1. Select the new volume → **Actions → Attach volume** → choose the instance in AZ-2.
2. SSH in and mount it the same way as Lab 2, Step 3.
3. Once verified, you can safely delete the old AZ-1 volume and the temporary snapshot to keep costs clean.

---

## Lab 4: Discover and Mount an Instance Store Volume

**Goal:** identify and prepare local ephemeral storage on a Nitro instance.

> **Prerequisite:** launch an instance family that includes local storage — look for a `d` in the name (e.g. `c8gd.xlarge`, `c6gd.medium`, `m6id.large`). Standard types like `t3.micro` won't show local drives.

```bash
# 1. List all block devices
lsblk
# nvme0n1 -> EBS root volume (has a partition + mountpoint /)
# nvme1n1 -> Instance Store (~size varies, no partition, no mountpoint yet)

# 2. Confirm which is which by model name
sudo nvme list
# Model column: "Amazon Elastic Block Store" vs "Amazon EC2 NVMe Instance Storage"

# 3. Confirm the instance store disk is empty
sudo file -s /dev/nvme1n1

# 4. Format it
sudo mkfs -t ext4 /dev/nvme1n1

# 5. Mount it
sudo mkdir /mnt/scratch-cache
sudo mount /dev/nvme1n1 /mnt/scratch-cache/

# 6. Verify
df -h /mnt/scratch-cache/
```

### Make the mount survive reboots (not stop/start)
```bash
sudo nano /etc/fstab
# append:
/dev/nvme1n1    /mnt/scratch-cache    ext4    defaults,nofail    0    2
```
The `nofail` flag is essential — if the device name shifts or fails to initialize on a new physical host, the OS still finishes booting instead of hanging.

---

## Lab 5: Prove the Instance Store Ephemeral Lifecycle

**Goal:** empirically verify that Instance Store data survives a reboot but not a stop/start.

### Phase 1 — Create test data
```bash
cd /mnt/scratch-cache
echo "This data is sitting on physical local hardware." > hardware_test.txt
cat hardware_test.txt
```

### Phase 2 — Reboot test (data should survive)
```bash
sudo reboot
```
Wait ~30–60 seconds, SSH back in:
```bash
cat /mnt/scratch-cache/hardware_test.txt
```
**Expected:** file content is intact — a reboot keeps you on the same physical host.

### Phase 3 — Stop/Start test (data should be wiped)
In the console: **Instance state → Stop instance** → wait for `stopped` → **Instance state → Start instance**.

```bash
cat /mnt/scratch-cache/hardware_test.txt
```
**Expected:** `No such file or directory`. Running `lsblk` will show `/dev/nvme1n1` isn't even mounted — the drive came back completely blank because you landed on a brand-new physical host.

### Automate recovery for this exact scenario (User Data)
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

## Lab 6: Resize a Live Volume with Zero Downtime

**Goal:** grow a `gp3` volume from 20 GiB to 40 GiB while the instance keeps running, using Elastic Volumes.

### Step 1 — Modify in the console
1. **Volumes** → select your attached volume → **Actions → Modify volume**.
2. Change size `20 → 40 GiB` (optionally change type or IOPS/throughput too).
3. Click **Modify** and confirm. Volume state shows `modifying (optimizing)` while the instance keeps running normally.

### Step 2 — Extend the filesystem inside Linux
```bash
# 1. Confirm the kernel sees the new hardware size
lsblk
# xvdf shows 40G, but xvdf1 partition is still 20G

# 2. Grow the partition (note the space between device and partition number)
sudo growpart /dev/xvdf 1
lsblk   # xvdf1 should now show 40G

# 3. Check filesystem type
df -Th

# 4a. If ext4 (Ubuntu/Debian):
sudo resize2fs /dev/xvdf1

# 4b. If xfs (Amazon Linux/RHEL) — use the mount PATH, not the device name:
sudo xfs_growfs /data

# 5. Verify
df -h /data
```

> ⚠️ **6-hour rule:** you must wait ~6 hours before modifying the same volume again.
> ⚠️ **To shrink a volume:** Elastic Volumes can't do this — create a new, smaller volume and manually `rsync` data across instead.

---

## Lab 7: Encrypt an Existing Unencrypted Volume

**Goal:** convert a legacy unencrypted volume to an encrypted one (cannot be done in place).

```bash
# 1. Snapshot the unencrypted live volume
aws ec2 create-snapshot --volume-id vol-unencrypted-example \
  --description "Pre-encryption-backup"

# 2. Copy the snapshot with encryption enabled
aws ec2 copy-snapshot \
  --source-region ap-south-1 \
  --source-snapshot-id snap-unencrypted-example \
  --region ap-south-1 \
  --encrypted \
  --kms-key-id alias/aws/ebs \
  --description "Encrypted-copy"

# 3. Create a new volume from the encrypted snapshot copy (same AZ as your instance)
aws ec2 create-volume \
  --availability-zone ap-south-1a \
  --snapshot-id snap-encrypted-copy-example \
  --volume-type gp3

# 4. Detach the old unencrypted volume, attach the new encrypted volume
aws ec2 detach-volume --volume-id vol-unencrypted-example
aws ec2 attach-volume \
  --volume-id vol-encrypted-new \
  --instance-id i-0123456789abcdef0 \
  --device /dev/sdf
```
Mount the new volume the same way as Lab 1, Step 3 (skip `mkfs` if you want to preserve the copied data — the filesystem already exists from the source snapshot).

**Bonus check:** confirm any *new* snapshot taken from this volume is automatically encrypted too — that inheritance is a core property of EBS encryption.
