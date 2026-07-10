# EC2 — Troubleshooting Guide

Symptom-first playbooks for the most common EC2 failure modes.

## Table of Contents
- [Instance Unreachable / Failing Status Checks](#instance-unreachable--failing-status-checks)
- [User Data / Bootstrap Script "Didn't Work"](#user-data--bootstrap-script-didnt-work)
- [IMDS / Container Can't Read Metadata](#imds--container-cant-read-metadata)
- [MAC-Locked Software License Broke After Instance Replacement](#mac-locked-software-license-broke-after-instance-replacement)
- [Placement Group Launch Failures](#placement-group-launch-failures)
- [AMI Still Being Billed After "Deleting" It](#ami-still-being-billed-after-deleting-it)
- [CloudFormation Stack Fails Silently Despite Instance Running](#cloudformation-stack-fails-silently-despite-instance-running)

---

## Instance Unreachable / Failing Status Checks

**First, identify which tier failed:**

| Console shows | Meaning | Fix |
|---|---|---|
| `System status check: failed` | AWS physical host/hardware problem | **Stop → Start** the instance (not reboot) to force migration to new hardware |
| `Instance status check: failed` | OS/kernel/network config problem inside your VM | Inspect logs, use Serial Console, or apply the rescue-volume pattern |

### If it's a System Status Check failure
```bash
aws ec2 stop-instances --instance-ids i-0123456789abcdef0
aws ec2 start-instances --instance-ids i-0123456789abcdef0
```
⚠️ Any **Instance Store** data is wiped during this cycle — back up first if possible.

### If it's an Instance Status Check failure
1. **Get the system log** (Console: `Actions → Monitor and troubleshoot → Get system log`, or `aws ec2 get-console-output`). Look for `Kernel panic`, `Failed to mount`, or broken network config lines.
2. **Try the EC2 Serial Console** for raw terminal access when SSH/network is completely down.
3. **Rescue-volume pattern** if the OS is unrecoverable in place:
   - Stop the broken instance.
   - Detach its root EBS volume.
   - Attach it as a *secondary* disk on a healthy rescue instance in the same AZ.
   - Mount it, fix the broken config/logs, unmount.
   - Reattach it as `/dev/sda1` on the original instance and start it.

### Prevent recurrence
Set up **CloudWatch Auto-Recovery** on `StatusCheckFailed_System` so AWS migrates automatically without a manual page:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2-Auto-Recovery" \
  --namespace "AWS/EC2" \
  --metric-name "StatusCheckFailed_System" \
  --statistic Minimum --period 60 --evaluation-periods 2 \
  --threshold 0 --comparison-operator GreaterThanThreshold \
  --dimensions "Name=InstanceId,Value=i-0123456789abcdef0" \
  --alarm-actions "arn:aws:automate:ap-south-1:ec2:recover"
```

---

## User Data / Bootstrap Script "Didn't Work"

**Symptom:** instance shows `2/2 checks passed`, but your app never came up.

This is expected — a healthy boot only means the OS started, not that your script succeeded.

1. Check the cloud-init output log:
   ```bash
   cat /var/log/cloud-init-output.log
   ```
2. Common root causes:
   - **Missing shebang** (`#!/bin/bash`) as the very first line.
   - Script assumed **it runs once per boot** — it only runs on first launch by default, so a Stop/Start cycle won't re-trigger it.
   - Script silently exceeded the **16 KB size limit** and got truncated.
   - A command inside the script failed partway through, but nothing was checking exit codes — the rest of the script (and the "healthy" status) proceeded anyway.
3. For CloudFormation-driven launches, prefer `cfn-init` + `cfn-signal` over a raw shell script — a failed `cfn-signal` will correctly fail (and roll back) the stack instead of reporting false success.

---

## IMDS / Container Can't Read Metadata

**Symptom:** `curl` to `169.254.169.254` times out from inside a Docker container or Kubernetes pod, but works fine on the host.

**Cause:** the default IMDSv2 response hop limit is `1`. Crossing into a container network bridge adds a hop, so the packet gets dropped — a deliberate security measure against credential theft from compromised containers.

**Fix (only if the container genuinely needs metadata access):**
```bash
aws ec2 modify-instance-metadata-options \
    --instance-id i-0123456789abcdef0 \
    --http-put-response-hop-limit 2
```

**Symptom:** IMDSv1 calls suddenly stop working after a security hardening pass.

**Cause:** IMDSv2 was enforced (`--http-tokens required`). Update your scripts/SDKs to use the token-based flow:
```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 6")
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

---

## MAC-Locked Software License Broke After Instance Replacement

**Symptom:** licensed enterprise software (SAP, Oracle, legacy CAD tools) suddenly reports an invalid license after an instance stop/start, termination, or hardware failover.

**Cause:** the license was bound to the primary ENI's MAC address, which changes when the instance moves to new physical hardware or a replacement instance is launched.

**Fix (going forward):**
1. Create a **standalone ENI** independent of any instance's lifecycle.
2. Attach it as a **secondary** interface.
3. Re-register the license against that ENI's MAC address.
4. On future instance replacement, simply detach/reattach that same ENI — the MAC (and thus the license) follows the ENI, not the instance.

**Reminder:** after attaching a second ENI, Linux won't route return traffic through it automatically — configure policy-based routing (`iproute2`) if the second interface needs to actually serve traffic.

---

## Placement Group Launch Failures

**Symptom:** `InsufficientCapacity` error when launching into a **Cluster** placement group.

**Likely causes:**
- Mixed instance types/sizes in the same Cluster group — the physical rack can't fit dissimilar hardware profiles.
- Adding instances to an already-full rack after the group was initially populated.

**Fixes:**
- Use the **same instance type** for every node in a Cluster group.
- **Launch all required instances in one API call** rather than incrementally.
- If a stopped instance in a Cluster/Partition group fails to restart, the rack may no longer have room — remove it from the placement group to start it elsewhere, then re-plan capacity.

**Symptom:** can't add an 8th instance to a **Spread** placement group in one AZ.

**Cause:** Spread groups are hard-capped at **7 running instances per AZ**. Use additional AZs, or switch to a **Partition** group if you need to scale beyond 7 while retaining hardware-level isolation.

---

## AMI Still Being Billed After "Deleting" It

**Symptom:** you deregistered an AMI, but storage costs didn't drop.

**Cause:** deregistering an AMI removes the *launch pointer* only — the underlying EBS snapshot(s) remain and continue to incur storage charges until explicitly deleted.

**Fix:**
```bash
# Find the snapshot tied to the AMI
aws ec2 describe-snapshots --owner-ids self \
  --filters "Name=description,Values='*ami-0123456789example*'"

# Delete it
aws ec2 delete-snapshot --snapshot-id snap-0123456789example
```

**Prevention for fleets on Auto Scaling:** use **AMI Deprecation** (with a future date) instead of hard deregistration, so in-flight scaling actions aren't broken while you migrate consumers to the new image — then deregister and clean up snapshots once nothing references the old AMI.

---

## CloudFormation Stack Fails Silently Despite Instance Running

**Symptom:** the EC2 instance is up and passing status checks, but the application inside it never actually configured correctly, and CloudFormation reports `CREATE_COMPLETE` anyway.

**Cause:** a raw shell script in `UserData` has no way to report failure back to CloudFormation — the stack only tracks whether the *instance* launched, not whether your *script* succeeded.

**Fix:** move configuration logic into `AWS::CloudFormation::Init` metadata and invoke it via `cfn-init`, then call `cfn-signal` to report real success/failure:
- A failed `cfn-signal` (or a timeout waiting for one) causes CloudFormation to correctly mark the resource `CREATE_FAILED` and roll back the stack, instead of a false-positive `CREATE_COMPLETE`.
