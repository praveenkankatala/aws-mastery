# EC2 — Commands Cheatsheet

Quick-reference commands grouped by topic. Pair with `README.md` for the "why" behind each one.

## Table of Contents
- [Instance Metadata (IMDS)](#instance-metadata-imds)
- [Instance Status Checks](#instance-status-checks)
- [User Data / Bootstrapping](#user-data--bootstrapping)
- [CloudFormation User Data](#cloudformation-user-data)
- [AMI Lifecycle](#ami-lifecycle)
- [Placement Groups](#placement-groups)
- [Auto-Recovery (CloudWatch)](#auto-recovery-cloudwatch)

---

## Instance Metadata (IMDS)

```bash
# --- IMDSv1 (legacy, no auth — avoid in production) ---
curl http://169.254.169.254/latest/meta-data/instance-id

# --- IMDSv2 (modern, session-token based) ---
# Step 1: request a token (valid for 6 seconds here)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 6")

# Step 2: use the token to query any metadata path
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/local-ipv4

# --- Enforce IMDSv2 only + fix the container hop-limit gotcha ---
aws ec2 modify-instance-metadata-options \
    --instance-id i-0123456789abcdef0 \
    --http-tokens required \
    --http-put-response-hop-limit 2
```

---

## Instance Status Checks

```bash
# Describe status checks for an instance
aws ec2 describe-instance-status --instance-ids i-0123456789abcdef0

# Stop / start to force migration off a bad physical host
# (Stop releases the host; Start schedules a new one — required for a
# System Status Check failure, since a reboot alone keeps the same host)
aws ec2 stop-instances --instance-ids i-0123456789abcdef0
aws ec2 start-instances --instance-ids i-0123456789abcdef0

# Pull the raw console/system log for OS-level troubleshooting
aws ec2 get-console-output --instance-id i-0123456789abcdef0
```

---

## User Data / Bootstrapping

```bash
# Minimal Linux bootstrap script (User Data field)
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "Hello from User Data" > /var/www/html/index.html

# Inspect why a bootstrap script failed (instance still shows healthy!)
cat /var/log/cloud-init-output.log
```

---

## CloudFormation User Data

```yaml
UserData:
  Fn::Base64:
    Fn::Sub: |
      #!/bin/bash
      yum update -y
      yum install -y httpd
      systemctl start httpd
      systemctl enable httpd
      echo "<h1>Welcome to the ${EnvironmentName} Web Server!</h1>" >> /var/www/html/index.html
```

---

## AMI Lifecycle

```bash
# Create ("bake") an AMI from a running instance
aws ec2 create-image \
  --instance-id i-0123456789abcdef0 \
  --name "Production-Gold-Image" \
  --no-reboot   # omit this flag to let AWS safely stop/start for consistency

# List your AMIs
aws ec2 describe-images --owners self

# Copy an AMI to another region (DR / global deployment)
aws ec2 copy-image \
  --source-region ap-south-1 \
  --source-image-id ami-0123456789example \
  --region us-east-1 \
  --name "Production-Gold-Image-DR-Copy"

# Share an AMI with another AWS account
aws ec2 modify-image-attribute \
  --image-id ami-0123456789example \
  --launch-permission "Add=[{UserId=111122223333}]"

# Retirement — step 1: deregister (stops new launches)
aws ec2 deregister-image --image-id ami-0123456789example

# Retirement — step 2: delete the underlying snapshot (stops storage billing!)
aws ec2 describe-snapshots --owner-ids self \
  --filters "Name=description,Values='*ami-0123456789example*'"
aws ec2 delete-snapshot --snapshot-id snap-0123456789example
```

---

## Placement Groups

```bash
# Create a Cluster placement group (low latency, HPC)
aws ec2 create-placement-group --group-name hpc-cluster --strategy cluster

# Create a Spread placement group (max isolation, 7 instances/AZ max)
aws ec2 create-placement-group --group-name critical-nodes --strategy spread

# Create a Partition placement group (large distributed systems)
aws ec2 create-placement-group --group-name kafka-partitions \
  --strategy partition --partition-count 4

# Launch an instance into a placement group
aws ec2 run-instances \
  --image-id ami-0123456789example \
  --instance-type c8g.2xlarge \
  --count 4 \
  --placement "GroupName=hpc-cluster"
```

---

## Auto-Recovery (CloudWatch)

```bash
# Create an alarm that triggers EC2 auto-recovery on a system status check failure
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-System-Status-Check-Failed" \
  --namespace "AWS/EC2" \
  --metric-name "StatusCheckFailed_System" \
  --statistic Minimum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --dimensions "Name=InstanceId,Value=i-0123456789abcdef0" \
  --alarm-actions "arn:aws:automate:ap-south-1:ec2:recover"
```
