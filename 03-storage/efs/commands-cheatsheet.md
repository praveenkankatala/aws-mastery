# Commands Cheat Sheet: EFS & FSx

Every command used across both labs, plus the AWS CLI equivalent of each console step so you can script the entire setup.

---

## Amazon EFS

### Security Group (CLI)

```bash
# Create the security group
aws ec2 create-security-group \
  --group-name efs-sg \
  --description "Allow NFS for EFS" \
  --vpc-id vpc-xxxxxxxx

# Allow inbound NFS (2049) — restrict the CIDR in production
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxx \
  --protocol tcp \
  --port 2049 \
  --cidr 0.0.0.0/0
```

### Create the File System (CLI)

```bash
aws efs create-file-system \
  --creation-token my-shared-efs \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted

# One mount target per AZ/subnet
aws efs create-mount-target \
  --file-system-id fs-0abcd1234efgh5678 \
  --subnet-id subnet-xxxxxxxx \
  --security-groups sg-xxxxxxxx

# List file systems / mount targets
aws efs describe-file-systems
aws efs describe-mount-targets --file-system-id fs-0abcd1234efgh5678
```

### Lifecycle Management (CLI)

```bash
aws efs put-lifecycle-configuration \
  --file-system-id fs-0abcd1234efgh5678 \
  --lifecycle-policies TransitionToIA=AFTER_30_DAYS TransitionToArchive=AFTER_90_DAYS
```

### Launch EC2 Instances (CLI)

```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxx \
  --count 2 \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-xxxxxxxx \
  --subnet-id subnet-xxxxxxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=EFS-Demo-Node}]'
```

### Mount / Use EFS (on the instance, shell)

```bash
# Install the EFS mount helper
sudo dnf update -y
sudo dnf install -y amazon-efs-utils

# Create local mount point
sudo mkdir -p /mnt/efs

# Mount with TLS encryption in transit
sudo mount -t efs -o tls fs-0abcd1234efgh5678:/ /mnt/efs

# Confirm the mount
df -h

# Write a file
cd /mnt/efs
sudo touch demo.txt
echo "Hello from EC2 Instance 1!" | sudo tee demo.txt

# Read it from another instance
cd /mnt/efs
ls -la
cat demo.txt

# Unmount before terminating
cd ~
sudo umount /mnt/efs
```

### Clean Up (CLI)

```bash
aws ec2 terminate-instances --instance-ids i-xxxx i-yyyy
aws efs delete-mount-target --mount-target-id fsmt-xxxxxxxx
aws efs delete-file-system --file-system-id fs-0abcd1234efgh5678
```

---

## Amazon FSx (for Lustre)

### S3 Bucket Setup (CLI)

```bash
aws s3 mb s3://fsx-lustre-demo-bucket
echo "Processing data with FSx for Lustre!" > input.txt
aws s3 cp input.txt s3://fsx-lustre-demo-bucket/
```

### Create the File System (CLI)

```bash
aws fsx create-file-system \
  --file-system-type LUSTRE \
  --storage-capacity 1200 \
  --subnet-ids subnet-xxxxxxxx \
  --security-group-ids sg-xxxxxxxx \
  --lustre-configuration \
      DeploymentType=SCRATCH_2,\
ImportPath=s3://fsx-lustre-demo-bucket,\
ExportPath=s3://fsx-lustre-demo-bucket/export

aws fsx describe-file-systems
```

### Mount / Use FSx for Lustre (on the instance, shell)

```bash
# Install the Lustre client
sudo dnf update -y
sudo dnf install -y lustre-client

# Create local mount point
sudo mkdir -p /mnt/fsx

# Mount (get DNS name + mount name from `aws fsx describe-file-systems`)
sudo mount -t lustre -o noatime,flock File-System-DNS-Name@tcp:/Mount-Name /mnt/fsx

# Confirm S3 data is visible
cd /mnt/fsx
ls -la

# Write new data
echo "Analysis complete." | sudo tee output.txt

# Unmount before terminating
sudo umount /mnt/fsx
```

### Export Data Back to S3 (CLI)

```bash
aws fsx create-data-repository-task \
  --file-system-id fs-xxxxxxxxxxxxxxxxx \
  --type EXPORT_TO_REPOSITORY \
  --paths output.txt \
  --report Enabled=true,Format=REPORT_CSV_20191124,Path=s3://fsx-lustre-demo-bucket/reports/

aws fsx describe-data-repository-tasks
```

### Clean Up (CLI)

```bash
aws ec2 terminate-instances --instance-ids i-xxxx
aws fsx delete-file-system --file-system-id fs-xxxxxxxxxxxxxxxxx
aws s3 rm s3://fsx-lustre-demo-bucket --recursive
aws s3 rb s3://fsx-lustre-demo-bucket
```

---

## Quick Reference: Ports & Protocols

| Service | Port | Protocol |
|---|---|---|
| EFS | 2049 (TCP) | NFSv4 |
| FSx for Windows File Server | 445 (TCP) | SMB |
| FSx for NetApp ONTAP | 111, 635, 2049 (NFS) / 445 (SMB) / 3260 (iSCSI) | NFS, SMB, iSCSI |
| FSx for OpenZFS | 2049 (TCP) | NFS |
| FSx for Lustre | 988 (TCP) | Lustre client |

## Quick Reference: Useful Diagnostic Commands

```bash
df -h                     # confirm a mount is active and see used/free space
mount | grep efs          # show current EFS mounts and their options
mount | grep lustre        # show current Lustre mounts and their options
sudo systemctl status amazon-efs-mount-watchdog   # EFS mount watchdog status
lfs df -h /mnt/fsx          # Lustre-specific space/usage report
nc -zv <mount-target-ip> 2049   # test NFS port reachability
nc -zv <fsx-dns-name> 988       # test Lustre port reachability
```
