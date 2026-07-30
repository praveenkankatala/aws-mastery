# AWS Migration — Troubleshooting

Real error messages, what actually causes them, and how to fix them. Organized by service so you can jump straight to the thing that's broken.

> **The 60-second triage that solves a surprising number of problems:**
> ```bash
> aws sts get-caller-identity        # right account?
> echo $AWS_REGION                   # right region? (mismatched regions cause a LOT of this)
> aws mgn describe-source-servers --query 'items[].dataReplicationInfo.dataReplicationError'
> aws dms describe-connections --query 'Connections[].[EndpointIdentifier,Status,LastFailureMessage]'
> # then: is it DNS, is it routing, is it a security group, or is it IAM?
> ```
> Nine times out of ten it's one of those four.

---

## Contents

- [1. Discovery and Migration Hub](#1-discovery-and-migration-hub)
- [2. MGN — agent installation](#2-mgn--agent-installation)
- [3. MGN — replication problems](#3-mgn--replication-problems)
- [4. MGN — launch and boot problems](#4-mgn--launch-and-boot-problems)
- [5. MGN — post-launch and application problems](#5-mgn--post-launch-and-application-problems)
- [6. Windows-specific migration issues](#6-windows-specific-migration-issues)
- [7. Linux-specific migration issues](#7-linux-specific-migration-issues)
- [8. DMS — connectivity and setup](#8-dms--connectivity-and-setup)
- [9. DMS — full load failures](#9-dms--full-load-failures)
- [10. DMS — CDC and lag problems](#10-dms--cdc-and-lag-problems)
- [11. DMS — data validation mismatches](#11-dms--data-validation-mismatches)
- [12. Schema conversion problems](#12-schema-conversion-problems)
- [13. DataSync problems](#13-datasync-problems)
- [14. Snow Family problems](#14-snow-family-problems)
- [15. Networking and connectivity](#15-networking-and-connectivity)
- [16. DNS and cutover problems](#16-dns-and-cutover-problems)
- [17. IAM and permissions](#17-iam-and-permissions)
- [18. Performance problems after migration](#18-performance-problems-after-migration)
- [19. Cost surprises](#19-cost-surprises)
- [20. Quota and limit errors](#20-quota-and-limit-errors)
- [21. Licensing and activation](#21-licensing-and-activation)
- [22. Container and refactor problems](#22-container-and-refactor-problems)
- [23. Rollback situations](#23-rollback-situations)
- [24. Where to find logs](#24-where-to-find-logs)
- [25. Escalation checklist](#25-escalation-checklist)

---

## 1. Discovery and Migration Hub

### Agent shows `UNHEALTHY` or `SHUTDOWN`

**Symptoms:** `describe-agents` returns `health: UNHEALTHY`, or the agent disappears from the console.

**Causes and fixes:**

| Cause | Check | Fix |
|---|---|---|
| No outbound HTTPS to the ADS endpoint | `curl -sI https://arsenal-discovery.<region>.amazonaws.com` | Open 443 outbound; add a proxy config if needed |
| Wrong region in the install command | `grep region /etc/opt/aws/discovery/*` | Reinstall with the correct region (must match the Migration Hub **home region**) |
| Invalid or deleted access keys | `aws iam list-access-keys --user-name ads-agent-user` | Create new keys, reinstall the agent |
| Clock skew > 5 minutes | `timedatectl` / `w32tm /query /status` | Fix NTP. Signature v4 fails outside a 5-minute window. |
| Agent service stopped | `systemctl status aws-discovery-daemon` | `systemctl restart aws-discovery-daemon` |
| Disk full | `df -h /var` | Free space; the agent needs room for its buffer |

```bash
# Linux agent logs
sudo tail -100 /var/log/aws/discovery/aws-discovery-daemon.log
sudo tail -100 /var/log/aws/discovery/codedeploy-agent-deployments.log

# Behind a proxy
sudo vi /etc/opt/aws/discovery/proxy.conf   # set http_proxy / https_proxy, then restart
```

### `Data collection is not started`

The agent installed fine but is registered without collecting.

```bash
AGENTS=$(aws discovery describe-agents --query 'agentsInfo[].agentId' --output text)
aws discovery start-data-collection-by-agent-ids --agent-ids $AGENTS
aws discovery describe-agents --query 'agentsInfo[].{Id:agentId,Coll:collectionStatus}'
```

### No dependency data / no connections showing

**Cause:** almost always that not enough time has passed, or you're using the **agentless** collector, which doesn't do process-level dependency mapping.

**Fix:** run for 2–4 weeks minimum. Deploy **agents** (not just the agentless collector) on servers where you need dependency detail. Generate representative traffic if you're in a lab.

### `HomeRegionNotSetException`

```
An error occurred (HomeRegionNotSetException) when calling the ListConfigurations operation
```

**Fix:**

```bash
aws migrationhub-config create-home-region-control \
  --home-region <region> --target Type=ACCOUNT,Id=$(aws sts get-caller-identity --query Account --output text)
aws migrationhub-config get-home-region
```

Then make every discovery/Migration Hub call **in that home region**, regardless of where you're migrating to.

### `InvalidParameterException: Home region cannot be changed`

You can't change it once set. Options: use a different AWS account, or accept it and continue (you can still migrate into any region — only the tracking data lives in the home region).

### Agentless collector shows no VMs

| Check | Fix |
|---|---|
| vCenter credentials | Read-only is enough, but it must have access to the datacenter/cluster objects |
| Collector can reach vCenter on 443 | Fix the firewall between the appliance and vCenter |
| Collector registered to AWS | Re-enter the AWS credentials in the collector web UI |
| vCenter version supported | Check the current supported range in the docs |

---

## 2. MGN — agent installation

### `Failed to fetch the installer` / TLS errors

```bash
curl -sI https://aws-application-migration-service-<region>.s3.<region>.amazonaws.com/latest/linux/aws-replication-installer-init
# 403/404 → wrong region in the URL
# timeout  → no outbound 443, or a proxy is intercepting TLS
```

If a corporate proxy does TLS inspection, add the proxy CA to the OS trust store, or allow-list the AWS endpoints.

### `Installation failed: Missing required package`

The agent needs a compiler toolchain and kernel headers to build its kernel module on some distributions.

```bash
# RHEL/CentOS/Amazon Linux
sudo yum install -y gcc make perl kernel-devel-$(uname -r) wget dhclient
# Debian/Ubuntu
sudo apt-get install -y gcc make perl linux-headers-$(uname -r) wget
```

If `kernel-devel` for your exact running kernel isn't available in any repo, either update the kernel and reboot, or the OS is too old — consider a different R (rebuild rather than rehost).

### `Unsupported operating system` / `Kernel version not supported`

**Cause:** the OS or kernel is outside MGN's supported matrix (very old CentOS 5/6, unusual kernels, some hardened distributions).

**Options, in order of preference:**
1. Patch the OS to a supported version, then retry.
2. Use **VM Import/Export** on an exported OVA/VHD instead of live replication.
3. Rebuild the server from scratch on a modern AMI and migrate only the data and application.
4. Reclassify as **Retain** until it can be rebuilt.

### `Agent installation failed: not enough free disk space`

The agent needs roughly 2 GB free on the root/system volume plus room for its own files.

```bash
df -h /
sudo journalctl --vacuum-size=200M
sudo yum clean all / sudo apt-get clean
```

### Windows: `The installation failed with exit code 1603` or the service won't start

| Cause | Fix |
|---|---|
| Not running as Administrator | Right-click → Run as administrator, or use an elevated PowerShell |
| .NET Framework missing | Install .NET Framework 4.6+ |
| Antivirus blocking the driver | Add an exclusion for `C:\Program Files (x86)\AWS Replication Agent\` and the driver |
| BitLocker enabled | Suspend or decrypt before installing (see [§6](#6-windows-specific-migration-issues)) |
| Pending reboot | Reboot, then install |
| VSS broken | `vssadmin list writers` — any writer in a failed state must be fixed first |

### Agent installs but the server never appears in the console

```bash
# 1. Region mismatch is the most common cause
grep -i region /var/lib/aws-replication-agent/*.cfg
aws mgn describe-source-servers --region <the-region-the-agent-was-installed-with>

# 2. IAM policy on the agent user
aws iam list-attached-user-policies --user-name mgn-agent-user
# must include AWSApplicationMigrationAgentPolicy

# 3. Endpoint reachability
curl -sI https://mgn.<region>.amazonaws.com

# 4. The service wasn't initialized in that region
aws mgn initialize-service --region <region>
```

---

## 3. MGN — replication problems

### State stuck in `INITIATING`

```bash
aws mgn describe-source-servers --filters sourceServerIDs=<s-id> \
  --query 'items[0].dataReplicationInfo.{State:dataReplicationState,Err:dataReplicationError,Msgs:dataReplicationInitiation.steps}'
```

Read the `steps` array — it names the exact step that's failing.

| Failing step | Meaning | Fix |
|---|---|---|
| `CREATE_SECURITY_GROUP` | No permission or SG quota reached | Check IAM; raise the SG-per-VPC quota |
| `LAUNCH_REPLICATION_SERVER` | No capacity, bad subnet, no IP addresses | Change instance type/AZ; check the staging subnet has free IPs |
| `BOOT_REPLICATION_SERVER` | Replication server can't reach AWS services | Staging subnet needs a route to the internet (NAT/IGW) or S3 + MGN VPC endpoints |
| `AUTHENTICATE_WITH_SERVICE` | IAM/instance profile problem | Re-run `aws mgn initialize-service` to recreate roles |
| `DOWNLOAD_REPLICATION_SOFTWARE` | No S3 access from the staging subnet | Add an S3 gateway endpoint or NAT route |
| `CREATE_STAGING_DISKS` | EBS quota, or an unsupported volume size | Check EBS volume-count and storage quotas |
| `PAIR_REPLICATION_SERVER_WITH_AGENT` | Port 1500 blocked | Allow TCP 1500 from the source to the staging SG |
| `CONNECT_AGENT_TO_REPLICATION_SERVER` | Same as above, or routing asymmetry | Verify both directions; check NACLs (they're stateless!) |

### `Stalled` / `BACKLOG` growing / `Lag duration` increasing

Replication can't keep up with the rate of change.

```bash
aws mgn describe-source-servers --filters sourceServerIDs=<s-id> --query \
 'items[0].dataReplicationInfo.{State:dataReplicationState,Lag:lagDuration,
Disks:replicatedDisks[].{Dev:deviceName,Total:totalStorageBytes,Replicated:replicatedStorageBytes,Backlog:backloggedStorageBytes}}'
```

| Cause | Fix |
|---|---|
| Bandwidth throttle too low | `aws mgn update-replication-configuration --source-server-id <s-id> --bandwidth-throttling 0` (0 = unlimited) |
| Replication server too small | Bump to `t3.large`/`m5.large`; use a **dedicated** replication server for this source |
| Source disk write rate exceeds link capacity | Reschedule heavy batch jobs; increase bandwidth; consider a bulk seed + shorter catch-up |
| Staging disk type too slow | Change `default-large-staging-disk-type` from `ST1` to `GP3` |
| Network congestion / packet loss | Test with `iperf3`; check DX/VPN utilization and any MSS clamping issue |
| Antivirus scanning every block | Exclude the agent directory |

```bash
# Bump the replication server for one source
aws mgn update-replication-configuration --source-server-id <s-id> \
  --replication-server-instance-type m5.large --use-dedicated-replication-server true \
  --bandwidth-throttling 0
```

### `RESCAN` or repeated `INITIAL_SYNC` restarts

**Causes:** source rebooted during initial sync (this is normal, it resumes); disk added or resized on the source; agent upgraded; the staging volume was deleted.

**Fix:** let it finish. If it loops repeatedly:

```bash
aws mgn retry-data-replication --source-server-id <s-id>
# If it still loops, check for source-side disk changes:
lsblk    # compare against what MGN thinks it's replicating
```

Adding a disk to the source mid-replication requires a rescan. Avoid changing source disk layout during a migration wave.

### `Disconnected from service` / `Agent not seen`

```bash
# On the source
sudo systemctl status aws-replication-agent
sudo tail -50 /var/lib/aws-replication-agent/agent.log.0
curl -sI https://mgn.<region>.amazonaws.com
nc -zv <replication-server-private-ip> 1500
```

If the replication server was terminated by something else (a cleanup script, an SCP, an auto-scaling rule), MGN will recreate it — but check whether a **Config rule or automation is terminating untagged instances**. That's a classic self-inflicted wound: your governance automation kills MGN's staging servers.

### `Error: Insufficient disk space on the staging area`

```bash
aws mgn describe-source-servers --filters sourceServerIDs=<s-id> \
  --query 'items[0].sourceProperties.disks'
aws ec2 describe-volumes --filters Name=tag:Name,Values="*Application Migration Service*" \
  --query 'Volumes[].{Id:VolumeId,Size:Size,State:State}'
```

MGN sizes staging volumes from the source disk size. If the source disk grew, or a snapshot-space quota is hit, you'll see this. Check EBS storage quotas per region and snapshot limits.

### Replication is very slow but nothing is "wrong"

Realistic expectations, and how to speed it up:

| Lever | Effect |
|---|---|
| Remove bandwidth throttle | Direct and large |
| Dedicated replication server, larger type | Large for multi-disk servers |
| Replicate fewer disks (exclude scratch/swap/temp) | Large — use `--devices` at install time |
| GP3 staging disks instead of ST1 | Moderate |
| Multiple servers in parallel vs sequential | Total wave time |
| Compress/clean the source first (logs, temp files) | Moderate; also cheaper |

```bash
# Reinstall the agent replicating only the disks you need
sudo ./aws-replication-installer-init --region <region> --devices /dev/sda,/dev/sdb --no-prompt
```

---

## 4. MGN — launch and boot problems

### Test/cutover instance launches but never becomes reachable

The single most useful command in this whole document:

```bash
aws ec2 get-console-output --instance-id i-<id> --output text | tail -100
aws ec2 get-console-screenshot --instance-id i-<id> --query ImageData --output text | base64 -d > screen.jpg
```

That screenshot tells you instantly whether it's a boot failure, a filesystem check, a Windows recovery screen, or a network problem.

### Instance boots to a black screen / `no bootable device` / UEFI shell

| Cause | Fix |
|---|---|
| **Boot mode mismatch** (most common) | Source is UEFI but the template says `LEGACY_BIOS`, or vice versa. Set it correctly and relaunch. |
| GPT disk with BIOS boot mode | Use `UEFI` |
| Bootloader on a disk that wasn't replicated | Replicate all disks, including small boot/EFI partitions |
| Corrupted bootloader from a mid-write snapshot | Relaunch from a different point-in-time snapshot |

```bash
# Check the source
# Linux:
[ -d /sys/firmware/efi ] && echo UEFI || echo BIOS
# Windows PowerShell:
Confirm-SecureBootUEFI

# Fix the template
aws mgn update-launch-configuration-template \
  --launch-configuration-template-id <lct-id> --boot-mode UEFI
```

### Linux instance boots into emergency mode / `dracut` shell

**Cause:** `/etc/fstab` references devices that don't exist on EC2 — by `/dev/sdX` name, or a UUID from a disk that wasn't replicated, or an NFS/CIFS mount that isn't reachable.

**Fix:** mount the root volume on a rescue instance and edit `fstab`:

```bash
# 1. Stop the failed instance, detach its root volume
aws ec2 stop-instances --instance-ids i-<broken>
aws ec2 detach-volume --volume-id vol-<root>

# 2. Attach to a working instance
aws ec2 attach-volume --volume-id vol-<root> --instance-id i-<rescue> --device /dev/sdf

# 3. Mount and fix
sudo mkdir /mnt/rescue && sudo mount /dev/nvme1n1p1 /mnt/rescue
sudo vi /mnt/rescue/etc/fstab
#   - use UUIDs, not /dev/sdX
#   - add 'nofail' to every non-essential mount (NFS, CIFS, data disks)
#   - comment out any share that won't exist in AWS yet
sudo umount /mnt/rescue

# 4. Reattach as root and start
aws ec2 detach-volume --volume-id vol-<root>
aws ec2 attach-volume --volume-id vol-<root> --instance-id i-<broken> --device /dev/xvda
aws ec2 start-instances --instance-ids i-<broken>
```

**Prevention:** add `nofail` to non-critical fstab entries on the source **before** migrating. This one change prevents most rehost boot failures.

### `The instance is missing the ENA/NVMe driver`

MGN's conversion normally handles this. When it doesn't:

```bash
# On the source, BEFORE migrating (Linux)
modinfo ena && echo "ENA present"
lsinitrd | grep -E 'ena|nvme'
# RHEL/CentOS: rebuild initramfs to include the drivers
sudo dracut -f --add-drivers "ena nvme nvme-core"

# Check the AMI/instance attribute afterwards
aws ec2 describe-instances --instance-ids i-<id> --query 'Reservations[].Instances[].EnaSupport'
aws ec2 modify-instance-attribute --instance-id i-<id> --ena-support
```

For Windows: install the AWS ENA and NVMe drivers on the source before migrating, or launch on an older instance family (e.g. `t2`/`m4`) that doesn't require ENA, then upgrade after installing drivers.

### `InsufficientInstanceCapacity` during launch

```
An error occurred (InsufficientInstanceCapacity) when calling the RunInstances operation
```

| Fix | Note |
|---|---|
| Try a different AZ | Change the subnet in the launch template |
| Try a different instance type/family | `m6i` ↔ `m5` ↔ `m6a` are usually interchangeable |
| Retry later | Capacity fluctuates, especially for large or older types |
| Use an On-Demand Capacity Reservation | Best practice for a **planned cutover** — reserve capacity days in advance |

```bash
aws ec2 create-capacity-reservation --instance-type m6i.2xlarge \
  --instance-platform Linux/UNIX --availability-zone <az> --instance-count 10 \
  --end-date-type limited --end-date <cutover-date+1d>
```

**Do this for every production cutover.** Discovering there's no capacity at 23:00 on a Saturday is entirely avoidable.

### Launched instance has the wrong instance type or subnet

MGN launches via an **EC2 launch template** it creates per source server. Your settings must be on the **default version** of that template.

```bash
aws ec2 describe-launch-templates --query \
 'LaunchTemplates[?contains(LaunchTemplateName,`<s-id>`)].[LaunchTemplateId,DefaultVersionNumber]' --output text

aws ec2 create-launch-template-version --launch-template-id lt-<id> --source-version 1 \
  --launch-template-data '{"InstanceType":"m6i.large","IamInstanceProfile":{"Name":"MigratedInstanceProfile"},
   "NetworkInterfaces":[{"DeviceIndex":0,"SubnetId":"subnet-<app>","Groups":["sg-<app>"],"AssociatePublicIpAddress":false,"DeleteOnTermination":true}]}'
aws ec2 modify-launch-template --launch-template-id lt-<id> --default-version 2
```

Also check `--target-instance-type-right-sizing-method`: if it's `BASIC`, MGN overrides your instance type. Set it to `NONE` when you want explicit control.

```bash
aws mgn update-launch-configuration --source-server-id <s-id> \
  --target-instance-type-right-sizing-method NONE
```

### Test launch succeeded but I can't SSH/RDP

Work down this list in order — it's almost always one of these:

```bash
# 1. Is it actually running and passing status checks?
aws ec2 describe-instance-status --instance-ids i-<id> \
  --query 'InstanceStatuses[].{Inst:InstanceStatus.Status,Sys:SystemStatus.Status}'

# 2. Security group — does it allow you in?
aws ec2 describe-security-groups --group-ids sg-<id> --query 'SecurityGroups[].IpPermissions'

# 3. NACLs — remember these are STATELESS, you need both directions
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=subnet-<id> \
  --query 'NetworkAcls[].Entries'

# 4. Route table — private subnet with no NAT and no endpoints = no SSM either
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=subnet-<id> \
  --query 'RouteTables[].Routes'

# 5. Use Session Manager instead of SSH (needs the SSM agent + instance profile + endpoint/NAT)
aws ssm describe-instance-information --filters Key=InstanceIds,Values=i-<id>
aws ssm start-session --target i-<id>

# 6. Reachability Analyzer will just tell you
aws ec2 create-network-insights-path --source <your-instance-or-igw> --destination i-<id> \
  --destination-port 22 --protocol tcp
aws ec2 start-network-insights-analysis --network-insights-path-id nip-<id>
aws ec2 describe-network-insights-analyses --network-insights-analysis-ids nia-<id> \
  --query 'NetworkInsightsAnalyses[0].{Found:NetworkPathFound,Why:Explanations}'
```

Note: the migrated instance keeps the **source's** SSH keys and local accounts — the EC2 key pair you select is largely irrelevant for a rehosted Linux box. Use the credentials you used on-premises.

### `finalize-cutover` failed / staging resources not cleaned up

```bash
aws mgn describe-jobs --filters jobIDs=<job-id> --query 'items[0].participatingServers'
aws mgn describe-job-log-items --job-id <job-id>
# Retry
aws mgn finalize-cutover --source-server-id <s-id>
```

If it keeps failing, check for a resource that can't be deleted (a snapshot still copying, a volume attached to something, an SCP denying the delete). Then remove leftovers manually:

```bash
aws ec2 describe-volumes --filters Name=status,Values=available \
  Name=tag:Name,Values="*Application Migration Service*" --query 'Volumes[].VolumeId' --output text
```

---

## 5. MGN — post-launch and application problems

### The app is running but can't reach its database / dependencies

**This is the number one post-cutover issue, and it's almost always hardcoded configuration.**

```bash
# Find the hardcoded references
sudo grep -rInE '([0-9]{1,3}\.){3}[0-9]{1,3}' /etc /opt /var/www 2>/dev/null | grep -v '127.0.0.1' | head -30
sudo grep -rIn 'old-hostname' /etc /opt 2>/dev/null

# Windows
Select-String -Path C:\inetpub\wwwroot\*.config,C:\apps\*\*.ini `
  -Pattern '\b\d{1,3}(\.\d{1,3}){3}\b'
```

Fix properly rather than by patching IPs:
1. Create a **Route 53 private hosted zone** with stable internal names (`db.orders.internal`).
2. Point the app at names, not IPs.
3. Better still, move the config into **Parameter Store / Secrets Manager**.

Also check hybrid DNS: can the migrated instance resolve on-prem names?

```bash
dig +short db01.corp.local
cat /etc/resolv.conf                                   # should be the VPC resolver (x.x.x.2)
aws route53resolver list-resolver-rules
aws route53resolver list-resolver-rule-associations
```

### Services didn't start after cutover

```bash
systemctl --failed
journalctl -p err -b --no-pager | tail -50
systemctl list-unit-files --state=enabled | grep <yourapp>
```

Common reasons: the service waits on a mount that isn't there (see fstab, above), depends on a license server it can't reach, binds to a specific IP that no longer exists, or starts before the network is fully up.

```bash
# Bind-to-specific-IP problem
sudo grep -rn 'bind' /etc/<app>/  # change to 0.0.0.0 or the new IP
```

### Scheduled jobs aren't running

```bash
crontab -l; ls -l /etc/cron.d/; systemctl list-timers --all
# Windows
Get-ScheduledTask | Where-Object State -ne 'Disabled' | Select TaskName, State, LastRunTime, LastTaskResult
```

Windows scheduled tasks with **stored passwords** frequently break after migration (the credential blob is tied to the machine). Re-enter credentials, or better, move to gMSA / an EventBridge + SSM alternative.

⚠️ **A real hazard:** cron jobs and scheduled tasks start running on the **test instance** too. If a job sends customer emails, posts to a payment API, or writes to a shared database, you've just caused a production incident from a "safe" test. **Launch test instances into an isolated subnet with no route to production**, and disable outbound-affecting jobs in the post-launch actions.

### Duplicate hostname / duplicate SID on the network

Because a rehost is a clone, the migrated server has the same hostname, and on Windows the same machine SID. During a parallel-run test both exist.

- Keep test instances isolated (no domain access, no shared DNS).
- Only join the domain in the **cutover** launch, not the test launch.
- If both must coexist, rename one before joining anything.

### Application performance is worse than on-prem

See [§18](#18-performance-problems-after-migration) — but check these three first:

1. **Instance type undersized** by MGN's `BASIC` right-sizing (it's conservative and doesn't understand your workload).
2. **EBS type/IOPS** — a source with local NVMe or a fast SAN moved onto gp3 baseline will feel slow. Provision IOPS/throughput to match.
3. **The database is still on-prem** while the app moved — every query now crosses the WAN. This is the wave-planning mistake from [README §11](README.md#11-wave-planning-and-the-migration-factory).

---

## 6. Windows-specific migration issues

### BitLocker prevents or corrupts replication

```powershell
Get-BitLockerVolume
Suspend-BitLocker -MountPoint "C:" -RebootCount 0     # suspend before replication
# or fully decrypt:
Disable-BitLocker -MountPoint "C:"
Get-BitLockerVolume -MountPoint "C:" | Select VolumeStatus, EncryptionPercentage
```

Re-enable encryption on the target using **EBS encryption** instead — that's the cloud-native equivalent and it's transparent to the OS.

### Windows won't activate on the target

```powershell
slmgr /dlv
Get-Service AWSLiteAgent, AmazonSSMAgent
# Point at the AWS KMS activation endpoint
slmgr /skms 169.254.169.250:1688
slmgr /ato
```

| Cause | Fix |
|---|---|
| Route to `169.254.169.250` blocked | Ensure the instance can reach the link-local KMS endpoint |
| BYOL flag set incorrectly | If `osByol=true` but you don't have licence mobility, relaunch with `osByol=false` |
| EC2Launch/EC2Config missing | Install the correct agent for the Windows version |
| Non-AWS-provided Windows image with a retail key | Re-key or use licence-included |

### Instance stuck at "Getting Windows ready" or a recovery screen

```bash
aws ec2 get-console-screenshot --instance-id i-<id> --query ImageData --output text | base64 -d > screen.jpg
```

| Screen | Meaning | Fix |
|---|---|---|
| Recovery / "Your PC needs to be repaired" | Boot mode mismatch or bootloader damage | Fix boot mode; try an earlier PIT snapshot |
| CHKDSK running | Filesystem inconsistency from the snapshot | Let it finish; if it loops, use an earlier snapshot |
| "Getting devices ready" for a long time | Driver enumeration | Normal on first boot — give it 15 minutes |
| Blue screen `INACCESSIBLE_BOOT_DEVICE` | Storage driver missing | Install AWS NVMe/ENA drivers on the source first |

### Domain trust broken after migration

```powershell
Test-ComputerSecureChannel -Verbose
Reset-ComputerMachinePassword -Server dc01.corp.local -Credential (Get-Credential)
# or rejoin
Remove-Computer -UnjoinDomainCredential (Get-Credential) -Force -Restart
Add-Computer -DomainName corp.local -Credential (Get-Credential) -OUPath "OU=AWS,DC=corp,DC=local" -Restart
nltest /sc_verify:corp.local
```

Also verify: the instance can reach the DCs (TCP/UDP 53, 88, 135, 389, 445, 464, 636, 3268-3269, plus the RPC dynamic range 49152–65535), and the DNS servers in the DHCP option set point at AD DNS.

### Static IP configuration causes no network on boot

The OS kept its on-prem static IP, which doesn't exist in the VPC.

```powershell
# Fix via Session Manager or EC2 Serial Console, then:
Get-NetAdapter | Set-NetIPInterface -Dhcp Enabled
Remove-NetIPAddress -IPAddress <old-static-ip> -Confirm:$false
Remove-NetRoute -DestinationPrefix 0.0.0.0/0 -Confirm:$false
Restart-Service dhcp
ipconfig /renew
```

**Prevention:** set the source NIC to DHCP before the cutover launch, or use MGN's `copy-private-ip` with a matching VPC CIDR.

### MSDTC / RPC / SQL cluster traffic fails

Distributed transactions and RPC use dynamic high ports. Security groups usually block them.

```bash
aws ec2 authorize-security-group-ingress --group-id sg-<app> \
  --protocol tcp --port 49152-65535 --source-group sg-<app>
aws ec2 authorize-security-group-ingress --group-id sg-<app> \
  --protocol tcp --port 135 --source-group sg-<app>
```

Better: restrict the RPC dynamic range on the servers themselves, then open only that narrower range.

---

## 7. Linux-specific migration issues

### SELinux blocks the app after migration

```bash
getenforce
sudo ausearch -m avc -ts recent | audit2why | head -40
# Relabel if contexts were lost
sudo touch /.autorelabel && sudo reboot
# Or fix a specific path
sudo restorecon -Rv /var/www/html
sudo semanage port -a -t http_port_t -p tcp 8080
```

### Network interface named differently (`eth0` vs `ens5`)

Old configs reference `eth0`; the new instance may present `ens5` or similar.

```bash
ip -br link
# Persist predictable naming, or update the config
sudo sed -i 's/eth0/ens5/g' /etc/sysconfig/network-scripts/ifcfg-*
# Or disable predictable names in the kernel cmdline: net.ifnames=0 biosdevname=0
```

### Cloud-init overwrites configuration on boot

On a rehosted server, cloud-init may reset the hostname, SSH keys, or `/etc/hosts`.

```bash
sudo cloud-init status --long
# Preserve hostname
sudo sed -i 's/^preserve_hostname: false/preserve_hostname: true/' /etc/cloud/cloud.cfg
# Or disable modules you don't want
sudo vi /etc/cloud/cloud.cfg   # remove set_hostname, update_hostname, users-groups
```

### `/etc/hosts` still resolves old on-prem IPs

```bash
cat /etc/hosts        # migrated verbatim; clean out stale entries
```

Replace with Route 53 private hosted zone records so this never happens again.

### Swap or large temp volumes wasting replication bandwidth

Exclude them at install time:

```bash
sudo ./aws-replication-installer-init --region <region> --devices /dev/sda --no-prompt
# Then recreate swap on the target
sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Old kernel can't use ENA, so only small instance types work

```bash
uname -r
modinfo ena
sudo yum update kernel -y && sudo reboot     # then verify
```

If updating the kernel isn't possible, launch on an `m4`/`t2` type first, install drivers, then resize.

---

## 8. DMS — connectivity and setup

### `Test connection failed: Application-Status: 1020912, Application-Message: Cannot connect`

Work through this in order:

```bash
# 1. Is the endpoint even reachable from the replication instance's subnet?
aws dms describe-replication-instances --query \
 'ReplicationInstances[].{Id:ReplicationInstanceIdentifier,Subnets:ReplicationSubnetGroup.Subnets[].SubnetIdentifier,IP:ReplicationInstancePrivateIpAddress}'

# 2. Security group on the DATABASE must allow the DMS instance's private IP / SG
aws ec2 describe-security-groups --group-ids sg-<db> --query 'SecurityGroups[].IpPermissions'

# 3. Route from the DMS subnet to the source (NAT, VPN, DX, or peering)
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<dms-subnet>

# 4. NACLs — stateless, need both directions
# 5. Credentials
aws dms describe-endpoints --query 'Endpoints[].[EndpointIdentifier,Username,ServerName,Port]' --output table
```

Then read the actual failure message:

```bash
aws dms describe-connections --filters Name=endpoint-arn,Values=<ep-arn> \
  --query 'Connections[].{Status:Status,Msg:LastFailureMessage}'
```

| Message contains | Meaning | Fix |
|---|---|---|
| `Cannot connect` / timeout | Network or SG | Routing + security group |
| `Access denied for user` | Credentials or grants | Recreate the DMS user with the right grants |
| `Unknown database` | Wrong database name | Fix the endpoint's `DatabaseName` |
| `SSL` / certificate errors | TLS mismatch | Set `SslMode`, import the CA with `aws dms import-certificate` |
| `ORA-01017` | Oracle bad credentials | Case-sensitive passwords; check the profile |
| `Login failed for user` (SQL Server) | SQL auth disabled or wrong user | Enable mixed-mode auth; verify the login |

### `Replication instance cannot be created: subnet group must span at least 2 AZs`

```bash
aws dms create-replication-subnet-group \
  --replication-subnet-group-identifier dms-subnets \
  --replication-subnet-group-description "DMS" \
  --subnet-ids subnet-<az-a> subnet-<az-b>
```

### `The IAM Role arn:aws:iam::...:role/dms-vpc-role is not configured properly`

```bash
aws iam create-role --role-name dms-vpc-role --assume-role-policy-document \
 '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"dms.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name dms-vpc-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSVPCManagementRole
```

The role name must be **exactly** `dms-vpc-role` (and `dms-cloudwatch-logs-role` for logging). DMS looks them up by name.

### Endpoint works from the console but the task can't connect

The console tests from a different path than the replication instance uses. Always use `test-connection` with the **replication instance ARN**, not the console's generic test.

---

## 9. DMS — full load failures

### Task status `failed` immediately

```bash
aws dms describe-replication-tasks --filters Name=replication-task-arn,Values=<arn> \
  --query 'ReplicationTasks[0].{Status:Status,Reason:StopReason,LastFailure:LastFailureMessage}'

# The real detail is in CloudWatch Logs
aws logs tail dms-tasks-<replication-instance-id> --follow
aws logs filter-log-events --log-group-name dms-tasks-<ri-id> \
  --filter-pattern "ERROR" --max-items 50
```

### `No tables were found at task initialization`

**Cause:** the table mapping doesn't match anything. Almost always a case-sensitivity or schema-name problem.

```bash
# Oracle stores names UPPERCASE; PostgreSQL lowercase; MySQL depends on the platform
# Wrong:
"object-locator": {"schema-name": "appdb", "table-name": "%"}
# Right for Oracle:
"object-locator": {"schema-name": "APPDB", "table-name": "%"}
```

For SQL Server, the schema is typically `dbo`, and the database is set on the endpoint, not in the mapping.

### `ERROR: relation "x" already exists` / duplicate key on full load

**Cause:** `TargetTablePrepMode` doesn't match your intent.

| Mode | Behaviour | When to use |
|---|---|---|
| `DO_NOTHING` | Leaves existing tables and data | You pre-created the schema with SCT — **the usual choice for heterogeneous** |
| `DROP_AND_CREATE` | Drops and recreates | Homogeneous, DMS-created schema, repeatable reload |
| `TRUNCATE_BEFORE_LOAD` | Keeps the structure, empties data | Reloading into a schema you control |

### `LOB column data is truncated` / task very slow on LOB tables

```json
"TargetMetadata": {
  "SupportLobs": true,
  "FullLobMode": false,          // true = correct but very slow
  "LimitedSizeLobMode": true,
  "LobMaxSize": 64,              // KB — must be >= your largest LOB
  "LobChunkSize": 64
}
```

Find your real maximum first:

```sql
-- MySQL
SELECT MAX(LENGTH(notes))/1024 AS max_kb FROM orders;
-- PostgreSQL
SELECT MAX(octet_length(notes))/1024 AS max_kb FROM orders;
-- Oracle
SELECT MAX(DBMS_LOB.GETLENGTH(notes))/1024 FROM orders;
```

Set `LobMaxSize` above that. If some LOBs are genuinely enormous, migrate those tables in a separate task with `FullLobMode: true`, or move the blobs to S3 and store references.

### `Tables errored: N` — one or more tables suspended

```bash
aws dms describe-table-statistics --replication-task-arn <arn> \
  --query 'TableStatistics[?TableState==`Table error`].[SchemaName,TableName,TableState]' --output table

# Look in the control tables DMS created on the target
# SELECT * FROM dms_control.awsdms_suspended_tables;
# SELECT * FROM dms_control.awsdms_apply_exceptions;
```

Then reload only the affected tables:

```bash
aws dms reload-tables --replication-task-arn <arn> \
  --tables-to-reload SchemaName=appdb,TableName=orders
```

| Typical cause | Fix |
|---|---|
| Data type overflow (e.g. `NUMBER` → `INT`) | Widen the target column |
| NOT NULL violation | Source has NULLs the target forbids; relax or clean |
| Foreign key violation during load | Disable FKs during load, re-enable after |
| Character set / collation mismatch | Set target charset to `utf8mb4` / UTF-8; `HandleCollationDiff: true` |
| No primary key | Add one, or accept full-load-only for that table |
| Reserved word as a column name | Add a transformation rule to rename |

### Full load is extremely slow

```json
"FullLoadSettings": {
  "MaxFullLoadSubTasks": 16,        // parallel tables
  "CommitRate": 50000,
  "CreatePkAfterFullLoad": true     // create indexes AFTER the load
}
```

Plus:
- **Parallel-load a huge table** by ranges (see the `table-settings` rule in the cheat sheet).
- **Scale the replication instance** — check `CPUUtilization`, `FreeableMemory`, `FreeStorageSpace` in CloudWatch.
- **Tune the target temporarily**: RDS Multi-AZ off, backup retention 0, larger instance class, `innodb_flush_log_at_trx_commit=2`, `sync_binlog=0`. Revert before cutover.
- **Drop secondary indexes** on the target before the load, recreate after. This is often a 3–5× difference.

```bash
aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CPUUtilization \
  --dimensions Name=ReplicationInstanceIdentifier,Value=dms-lab \
  --statistics Average --period 300 \
  --start-time $(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)
```

### `ERROR: Not enough free disk space on the replication instance`

```bash
aws dms modify-replication-instance --replication-task-arn <arn> --allocated-storage 200 --apply-immediately
```

Also reduce log verbosity — `LOGGER_SEVERITY_DETAILED_DEBUG` fills disks fast. Use it to diagnose, then turn it back down.

---

## 10. DMS — CDC and lag problems

### CDC never starts after full load

| Cause | Check | Fix |
|---|---|---|
| Migration type is `full-load` only | `describe-replication-tasks` → `MigrationType` | Recreate with `full-load-and-cdc` |
| Source logging not configured | See the checks below | Enable it, then restart the task |
| DMS user lacks replication privileges | `SHOW GRANTS` | Grant `REPLICATION CLIENT`, `REPLICATION SLAVE` (MySQL), `rds_replication` (PG), LogMiner grants (Oracle) |
| Binlogs/archive logs already purged | Retention too short | Increase retention **before** starting |

```sql
-- MySQL
SHOW VARIABLES LIKE 'log_bin';            -- ON
SHOW VARIABLES LIKE 'binlog_format';      -- ROW
SHOW VARIABLES LIKE 'binlog_row_image';   -- FULL
SHOW BINARY LOGS;
CALL mysql.rds_set_configuration('binlog retention hours', 72);

-- PostgreSQL
SHOW wal_level;                            -- logical
SELECT * FROM pg_replication_slots;
SHOW max_replication_slots;

-- Oracle
SELECT log_mode, supplemental_log_data_min, supplemental_log_data_all FROM v$database;

-- SQL Server
SELECT name, recovery_model_desc, is_cdc_enabled FROM sys.databases WHERE name='appdb';
```

### `CDCLatencySource` or `CDCLatencyTarget` climbing

**Source latency high** = DMS can't read changes fast enough:
- Long-running transactions on the source hold the log position open. Find and address them.
- Very high change volume — scale the replication instance.
- Oracle LogMiner is slow → switch to **Binary Reader** (`useLogminerReader=N` in extra connection attributes).
- Source I/O saturated.

**Target latency high** = DMS can't apply changes fast enough:
- Missing indexes on the target for the columns in UPDATE/DELETE `WHERE` clauses. **This is the most common cause by far** — without an index, every update is a full table scan.
- Enable batch apply:

```json
"TargetMetadata": {"BatchApplyEnabled": true},
"ChangeProcessingTuning": {
  "BatchApplyPreserveTransaction": false,   // faster, but transaction order not preserved
  "BatchApplyTimeoutMin": 1, "BatchApplyTimeoutMax": 30,
  "MemoryLimitTotal": 2048, "MemoryKeepTime": 60,
  "CommitTimeout": 1, "MinTransactionSize": 1000
}
```

- Scale up the target database.
- Triggers on the target firing on every applied row — disable during migration.

```bash
aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencyTarget \
  --dimensions Name=ReplicationInstanceIdentifier,Value=dms-lab Name=ReplicationTaskIdentifier,Value=<task-id> \
  --statistics Average Maximum --period 300 \
  --start-time $(date -u -d '6 hours ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --query 'Datapoints[].{T:Timestamp,Avg:Average,Max:Maximum}' --output table
```

### Changes to some tables aren't replicating

**Cause:** no primary key or unique index. DMS can full-load such tables but cannot reliably apply updates and deletes.

```sql
-- Find them
SELECT t.table_name FROM information_schema.tables t
LEFT JOIN information_schema.table_constraints c
  ON t.table_name = c.table_name AND c.constraint_type = 'PRIMARY KEY'
WHERE t.table_schema = 'appdb' AND c.constraint_name IS NULL;
```

**Fixes:** add a primary key on the source (best); or add a unique index; or accept full-load-only for those tables and reload during the cutover window.

### `The replication slot already exists` / slot leaking (PostgreSQL)

```sql
SELECT slot_name, active, restart_lsn, pg_size_pretty(
  pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots;

-- Drop an abandoned slot (only if you're sure the task is gone)
SELECT pg_drop_replication_slot('dms_slot_name');
```

⚠️ **An inactive slot prevents WAL from being recycled and will eventually fill the source disk.** If you delete a DMS task, check for and remove its slot.

### `ORA-01555: snapshot too old` / archive log missing

```sql
ALTER SYSTEM SET undo_retention = 10800;                    -- 3 hours
-- and increase archive log retention so DMS can catch up
EXEC rdsadmin.rdsadmin_util.set_configuration('archivelog retention hours', 48);  -- RDS Oracle
```

### After cutover, the target's auto-increment/sequence values are wrong

DMS migrates data, not sequence state. Your first insert on the target collides with an existing key.

```sql
-- MySQL
SELECT auto_increment FROM information_schema.tables WHERE table_schema='appdb' AND table_name='orders';  -- on source
ALTER TABLE appdb.orders AUTO_INCREMENT = <source_value + 1000>;                                          -- on target

-- PostgreSQL
SELECT setval(pg_get_serial_sequence('orders','id'), (SELECT COALESCE(MAX(id),0)+1000 FROM orders), false);

-- Oracle
ALTER SEQUENCE orders_seq RESTART START WITH <max+1000>;

-- SQL Server
DBCC CHECKIDENT ('orders', RESEED, <max+1000>);
```

**Put this in the cutover runbook as an explicit step.** It's forgotten constantly and shows up as duplicate-key errors the moment users come back.

---

## 11. DMS — data validation mismatches

### `ValidationState: Mismatched records`

```bash
aws dms describe-table-statistics --replication-task-arn <arn> \
  --query 'TableStatistics[?ValidationFailedRecords>`0`].[TableName,ValidationFailedRecords,ValidationSuspendedRecords,ValidationState]' --output table
```

Then look in the validation failure table on the target:

```sql
SELECT * FROM dms_control.awsdms_validation_failures_v1 ORDER BY FAILURE_TIME DESC LIMIT 50;
```

**Common benign causes (data is actually fine):**

| Cause | Explanation |
|---|---|
| Timestamp precision | Source has microseconds, target rounds to milliseconds |
| Float/double representation | Binary floating point differs; use `NUMERIC`/`DECIMAL` for money |
| Trailing whitespace | `CHAR` vs `VARCHAR` padding rules differ |
| Collation / case sensitivity | Set `HandleCollationDiff: true` |
| Empty string vs NULL | Oracle treats `''` as NULL; PostgreSQL doesn't |
| Rows changed during validation | Ongoing CDC — re-validate when writes are frozen |
| Timezone handling | `TIMESTAMP` vs `TIMESTAMP WITH TIME ZONE` |

**Causes that are real problems:**

| Cause | Fix |
|---|---|
| Truncated LOBs | Increase `LobMaxSize`, reload the table |
| Character set corruption (mojibake) | Set target to `utf8mb4`/UTF-8 and reload |
| Rows missing entirely | Reload the table; investigate the apply exceptions |
| Numeric overflow | Widen the target column, reload |

### Independent verification (always do this too)

Don't rely solely on DMS validation. Run your own checks on the business numbers:

```sql
-- Source and target, compare the outputs
SELECT COUNT(*)                    AS rows,
       SUM(amount)                 AS total_value,
       COUNT(DISTINCT customer_id) AS customers,
       MIN(created_at)             AS earliest,
       MAX(created_at)             AS latest
FROM orders;

-- Per-status breakdown catches partial-reload problems
SELECT status, COUNT(*), SUM(amount) FROM orders GROUP BY status ORDER BY status;

-- Then eyeball 10 specific records the business cares about,
-- including unicode names, apostrophes, NULLs, negative amounts,
-- and the oldest and newest rows.
```

### Validation is very slow or times out

```json
"ValidationSettings": {
  "ThreadCount": 10,
  "PartitionSize": 20000,
  "FailureMaxCount": 10000,
  "RecordFailureDelayLimitInMinutes": 0,
  "SkipLobColumns": true
}
```

Validating multi-TB LOB tables row by row is rarely worth it — skip LOB columns and validate those separately with checksums.

---

## 12. Schema conversion problems

### Conversion percentage is much lower than expected

That's information, not a failure. It means significant code work. Read the action items by severity and turn "significant" items into estimated days on your plan.

### Common conversion blockers and their answers

| Blocker | Approach |
|---|---|
| Oracle packages | Convert to PostgreSQL schemas + functions, or use the SCT extension pack |
| `CONNECT BY` hierarchical queries | Rewrite as recursive CTEs (`WITH RECURSIVE`) |
| `DBMS_*` built-ins | SCT extension pack provides substitutes; some need rewriting |
| Autonomous transactions | Redesign (PostgreSQL has no direct equivalent) |
| T-SQL stored procedures | Convert to PL/pgSQL, or keep T-SQL with **Babelfish** |
| SQL Server Agent jobs | pg_cron, or EventBridge + Lambda/ECS |
| Linked servers | postgres_fdw, or move the join into the application |
| `MERGE` | `INSERT ... ON CONFLICT DO UPDATE` |
| Case-sensitive identifiers | Convert everything to lowercase with a transformation rule — do this consistently or you'll chase quoting bugs for weeks |
| Oracle `ROWNUM` | `LIMIT` / `ROW_NUMBER()` |
| `NVARCHAR` byte-vs-char lengths | Check that data fits; UTF-8 multi-byte characters need more bytes |
| Empty string = NULL (Oracle) | Explicitly decide the semantics and test |

### SCT can't connect to the source

- Install the correct JDBC driver and point SCT at it (Settings → Global Settings → Drivers).
- Java version must match SCT's requirement.
- For Oracle, the user needs `SELECT_CATALOG_ROLE` and various `SELECT ANY DICTIONARY` grants.

### Extension pack failed to apply

```bash
aws dms start-metadata-model-export-to-target --migration-project-identifier <id> \
  --selection-rules file://rules.json --overwrite-extension-pack
```

The target user needs permission to create schemas and functions. On RDS, some operations require the `rds_superuser` role.

---

## 13. DataSync problems

### `Task execution failed: unable to connect to the source location`

```bash
aws datasync describe-task-execution --task-execution-arn <arn> \
  --query '{Status:Status,Result:Result}'
```

| `ErrorCode` | Cause | Fix |
|---|---|---|
| `AGENT_OFFLINE` | Agent VM down or lost connectivity | Check the agent VM; `describe-agent` should show `ONLINE` |
| `MOUNT_FAILED` | NFS export doesn't allow the agent's IP | Add the agent IP to `/etc/exports`, `exportfs -ra` |
| `PERMISSION_DENIED` | Agent lacks read access | For NFS use `no_root_squash` or map a suitable UID; for SMB give the service account read + read-ACL |
| `SMB_AUTHENTICATION` | Bad credentials or wrong domain | Verify `DOMAIN\user`; check for account lockout |
| `LOCATION_NOT_FOUND` | Wrong subdirectory | Path is relative to the export root, not the filesystem root |

### Agent won't activate

```bash
curl "http://<agent-ip>/?activationRegion=<region>"    # must return an activationKey
```

| Cause | Fix |
|---|---|
| Agent can't reach the DataSync endpoint | Open outbound 443 to `datasync.<region>.amazonaws.com` |
| Using a VPC endpoint but didn't specify it | Pass `--vpc-endpoint-id`, `--subnet-arns`, `--security-group-arns` to `create-agent` |
| Activation key expired | Keys are short-lived — regenerate and activate immediately |
| Wrong region | The activation region must match where you run `create-agent` |
| Agent VM under-resourced | Give it at least 4 vCPU / 32 GB RAM (80 GB disk) for real workloads |

### Transfer is much slower than the link should allow

| Cause | Fix |
|---|---|
| `BytesPerSecond` throttle set | `aws datasync update-task --options BytesPerSecond=-1` |
| Millions of small files | The bottleneck is metadata operations, not bandwidth. Split into multiple tasks by directory and run them in parallel. |
| `VerifyMode: POINT_IN_TIME_CONSISTENT` on a huge dataset | Use `ONLY_FILES_TRANSFERRED` for intermediate runs; do a full verify on the final run |
| Agent undersized | Larger agent instance/VM; more vCPU directly helps |
| Source storage is the bottleneck | Check the NAS's own IOPS and CPU |
| Single task, single agent | Multiple tasks across multiple agents, partitioned by folder |

```bash
# Parallelise by top-level directory
for D in finance hr archive legal; do
  aws datasync create-task --source-location-arn $SRC_LOC --destination-location-arn $DST_LOC \
    --name "sync-$D" --includes FilterType=SIMPLE_PATTERN,Value="/$D/*"
done
```

### Files transferred but permissions or ACLs are wrong

```bash
# NFS/POSIX
--options '{"PosixPermissions":"PRESERVE","Uid":"INT_VALUE","Gid":"INT_VALUE","Mtime":"PRESERVE","Atime":"BEST_EFFORT"}'

# SMB → FSx: preserve NTFS ACLs
--options '{"SecurityDescriptorCopyFlags":"OWNER_DACL_SACL"}'
```

`OWNER_DACL_SACL` requires the DataSync user to have `SeSecurityPrivilege` on the source. Without it, use `OWNER_DACL` and expect to lose audit ACLs.

Note: S3 has no POSIX permissions. Metadata is stored as object metadata and only round-trips correctly if S3 is an intermediate hop between two file systems.

### `FilesSkipped` is non-zero

```bash
aws logs filter-log-events --log-group-name /datasync/migration \
  --filter-pattern "skipped" --max-items 50
```

Usual reasons: open/locked files (Windows), files changing mid-transfer, path length > 1,024 characters, special characters, or symlinks pointing outside the transfer scope. Re-run the task — an incremental run usually picks them up once they're closed.

---

## 14. Snow Family problems

| Problem | Cause | Fix |
|---|---|---|
| Can't unlock the device | Wrong manifest or unlock code, or the manifest is from a different job | Re-download both: `get-job-manifest` and `get-job-unlock-code` |
| S3 adapter rejects writes | Service not started, or wrong endpoint/port | `snowballEdge start-service --service-id s3`; use `http://<ip>:8080` |
| Very slow copy | Single-threaded copy over 1 GbE | Use 10 GbE/25 GbE, run multiple parallel copy processes, or use DataSync on the device |
| `Access Denied` writing to the device | Using your normal AWS credentials | Use `snowballEdge get-secret-access-key` and a dedicated CLI profile |
| Import job partially failed | Bad blocks, or objects with invalid key names | Read the job completion report in S3; re-transfer only the failed objects online |
| Device arrived locked/damaged | Shipping | Contact AWS Support; don't force it |
| Data landed but is stale by weeks | Expected — it's a point-in-time copy | Run the delta: DMS CDC from the recorded position, or a DataSync incremental |

**The mistake that costs the most:** not recording the database log position (binlog file+offset / LSN / SCN) at the exact moment you copied to the device. Without it you cannot start CDC at the right point and you'll either lose data or duplicate it. Write it down, in the runbook, twice.

---

## 15. Networking and connectivity

### On-prem can't reach AWS (or vice versa)

Diagnose in this order — it's nearly always one of these five:

```bash
# 1. Route table has a route to the on-prem CIDR
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=subnet-<id> \
  --query 'RouteTables[].Routes'

# 2. Security group allows the traffic (stateful — inbound rule is enough)
aws ec2 describe-security-groups --group-ids sg-<id> --query 'SecurityGroups[].IpPermissions'

# 3. NACL allows BOTH directions (stateless!) including ephemeral ports 1024-65535
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=subnet-<id> \
  --query 'NetworkAcls[].Entries'

# 4. VPN tunnels are UP / DX BGP is established
aws ec2 describe-vpn-connections --query \
 'VpnConnections[].{Id:VpnConnectionId,State:State,Tunnels:VgwTelemetry[].{Status:Status,Msg:StatusMessage}}'
aws directconnect describe-virtual-interfaces --query \
 'virtualInterfaces[].{Name:virtualInterfaceName,State:virtualInterfaceState,BGP:bgpPeers[0].bgpPeerState}'

# 5. Transit Gateway route table has the right propagation and association
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id tgw-rtb-<id> \
  --filters Name=state,Values=active

# Then just ask Reachability Analyzer
aws ec2 create-network-insights-path --source <src> --destination <dst> --destination-port <port> --protocol tcp
aws ec2 start-network-insights-analysis --network-insights-path-id nip-<id>
aws ec2 describe-network-insights-analyses --network-insights-analysis-ids nia-<id> \
  --query 'NetworkInsightsAnalyses[0].{Found:NetworkPathFound,Why:Explanations}'
```

Also confirm the **on-prem side** has a route back to the VPC CIDR. Asymmetric routing is common and confusing: packets arrive but replies vanish.

### Overlapping CIDR ranges

**Symptom:** you can't route between on-prem `10.0.0.0/8` and your new VPC `10.20.0.0/16` because the on-prem supernet already covers it.

There is no clean fix. Options:
- **Re-IP the VPC** (easiest if caught early — this is why you plan CIDRs first).
- **Private NAT gateway** to translate addresses between overlapping spaces.
- **PrivateLink** for specific service-to-service access, which sidesteps routing entirely.
- **Transit Gateway with static routes** for narrow, non-overlapping subsets.

**Prevention:** reserve a dedicated, documented supernet for AWS before creating a single VPC.

### VPN tunnel flapping

| Cause | Fix |
|---|---|
| IKE/IPsec parameter mismatch | Compare with the downloaded configuration file for your device |
| Dead peer detection timing out | Set DPD to a longer interval; keep traffic flowing |
| No traffic → tunnel idles down | Configure a keepalive or monitoring ping |
| Only one tunnel configured | Configure **both** tunnels; AWS may fail over at any time for maintenance |
| NAT device in front of the customer gateway | Enable NAT-T; use the correct public IP |

### Large file transfers hang or reset (MTU / MSS issues)

Classic symptom: small requests work fine, large transfers stall.

```bash
# Find the working MTU
ping -M do -s 1472 <remote-ip>     # 1472 + 28 = 1500
ping -M do -s 8972 <remote-ip>     # jumbo frames
tracepath <remote-ip>
```

Fixes: clamp MSS on the on-prem router (`tcp-mss 1350` or similar); set the instance MTU to 1500 rather than 9001 when traffic crosses a VPN; note that Transit Gateway supports 8500-byte packets for VPC traffic but only 1500 over VPN.

### Instance in a private subnet can't reach AWS APIs

```bash
# Either a NAT route:
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=subnet-<id> \
  --query 'RouteTables[].Routes[?NatGatewayId!=null]'

# Or interface endpoints with private DNS enabled:
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=vpc-<id> \
  --query 'VpcEndpoints[].{Svc:ServiceName,Type:VpcEndpointType,DNS:PrivateDnsEnabled,State:State}' --output table

# Endpoint SG must allow 443 from the subnet CIDR
aws ec2 describe-security-groups --group-ids sg-<endpoints> --query 'SecurityGroups[].IpPermissions'

# And the VPC must have DNS support + hostnames enabled for private DNS to work
aws ec2 describe-vpc-attribute --vpc-id vpc-<id> --attribute enableDnsSupport
aws ec2 describe-vpc-attribute --vpc-id vpc-<id> --attribute enableDnsHostnames
```

---

## 16. DNS and cutover problems

### Users still hitting the old server after DNS change

| Cause | Fix |
|---|---|
| TTL was too high when you changed it | Lower TTL to 60s **at least 24–48 hours before** cutover (i.e. more than the old TTL ago) |
| Client/OS DNS caching | `ipconfig /flushdns`, `systemd-resolve --flush-caches`, restart browsers |
| Java applications cache DNS forever by default | Set `networkaddress.cache.ttl=60` in `java.security` |
| Connection pools hold old IPs | Restart the application, not just the DNS |
| Hardcoded IPs in client configs | Find and fix (grep the estate) |
| Corporate resolver with its own long cache | Ask the network team to flush |
| `/etc/hosts` overrides | Check on every client that's misbehaving |

```bash
aws route53 get-change --id <change-id> --query 'ChangeInfo.Status'    # INSYNC
for R in 8.8.8.8 1.1.1.1 9.9.9.9 <corp-resolver>; do echo -n "$R: "; dig +short @$R app.example.com; done
```

### AWS resources can't resolve on-prem names (or vice versa)

```bash
# AWS → on-prem: outbound resolver endpoint + forwarding rule
aws route53resolver list-resolver-endpoints
aws route53resolver list-resolver-rules
aws route53resolver list-resolver-rule-associations   # rule must be associated with the VPC

# on-prem → AWS: inbound resolver endpoint, and on-prem DNS forwards the AWS zone to its IPs
aws route53resolver list-resolver-endpoints --filters Name=Direction,Values=INBOUND
```

Test from the instance:

```bash
dig +short db01.corp.local
dig +short @<inbound-endpoint-ip> app.aws.corp.local
cat /etc/resolv.conf     # should be the VPC resolver (VPC base + 2)
```

**Set hybrid DNS up during Mobilize, not during a cutover.** It's the dependency that silently breaks everything else.

### Certificate errors after cutover

| Cause | Fix |
|---|---|
| Cert was on the server, traffic now terminates at the ALB | Import the cert to ACM, or request a new ACM cert with DNS validation |
| Incomplete chain | Include intermediates when importing |
| SAN doesn't cover the new hostname | Reissue with the correct SANs |
| Client pinning the old certificate | Coordinate with the client owners in advance |
| Internal CA not trusted by the new instance | Install the corporate root CA in the OS trust store |

### ALB health checks failing so no traffic flows

```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn> \
  --query 'TargetHealthDescriptions[].{Id:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason,Desc:TargetHealth.Description}' --output table
```

| Reason | Meaning | Fix |
|---|---|---|
| `Target.Timeout` | No response in time | SG must allow the ALB's SG on the health check port; app may be slow to start |
| `Target.FailedHealthChecks` | Wrong status code or path | Check `--matcher HttpCode` and the health check path actually exists |
| `Target.NotRegistered` | Not registered | `register-targets` |
| `Elb.RegistrationInProgress` | Just wait | Normal for ~30s |
| `Target.ResponseCodeMismatch` | App returns 302/401 on `/` | Add a dedicated unauthenticated `/health` endpoint |
| `Target.NotInUse` | Wrong AZ or subnet | The ALB must have a subnet in the target's AZ |

---

## 17. IAM and permissions

### `AccessDeniedException: User is not authorized to perform...`

```bash
# What exactly is being denied? CloudTrail tells you precisely.
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<Action> \
  --max-items 5 --query 'Events[].CloudTrailEvent' --output text | jq '.errorMessage, .userIdentity.arn'

# Simulate before you deploy
aws iam simulate-principal-policy --policy-source-arn <role-or-user-arn> \
  --action-names mgn:StartCutover dms:StartReplicationTask \
  --resource-arns "*"
```

Also check for an **SCP** blocking it at the organization level — the error looks identical to a missing IAM permission but no IAM change will fix it:

```bash
aws organizations list-policies-for-target --target-id <account-id> --filter SERVICE_CONTROL_POLICY
```

### Required roles for each migration service

| Service | Roles needed |
|---|---|
| MGN | `AWSApplicationMigrationAgentPolicy` on the agent user; service roles created by `initialize-service`; `AWSApplicationMigrationReplicationServerRole`; `AWSApplicationMigrationConversionServerRole` |
| DMS | `dms-vpc-role`, `dms-cloudwatch-logs-role`, `dms-access-for-endpoint` (exact names) |
| DataSync | A role for S3 access (`BucketAccessRoleArn`); the agent needs no role |
| ADS | `AWSApplicationDiscoveryAgentAccess` on the agent user |
| VM Import | A role literally named `vmimport` with the correct external-ID trust policy |
| AWS Backup | `AWSBackupDefaultServiceRole` |

### Agent authentication failures

```
Failed to authenticate with the service. Please check your credentials.
```

- Keys deleted, deactivated, or rotated → create new keys and reinstall the agent.
- Clock skew > 5 minutes → fix NTP.
- Keys are for a different account than the service was initialized in.
- An SCP or permissions boundary is blocking the action.

### KMS `AccessDeniedException` on encrypted volumes

The key policy must allow the service-linked roles to use the key:

```bash
aws kms get-key-policy --key-id <key-id> --policy-name default
```

Add `kms:Encrypt`, `kms:Decrypt`, `kms:ReEncrypt*`, `kms:GenerateDataKey*`, `kms:DescribeKey`, `kms:CreateGrant` for the MGN/DMS/EBS service roles. Missing `CreateGrant` is a frequent, confusing failure.

---

## 18. Performance problems after migration

### Systematic diagnosis

```bash
# 1. Is the instance the bottleneck?
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-<id> --statistics Average Maximum --period 300 \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)

# 2. Burst credits exhausted? (t-family)
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUCreditBalance \
  --dimensions Name=InstanceId,Value=i-<id> --statistics Minimum --period 300 \
  --start-time $(date -u -d '6 hours ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)

# 3. EBS throttling?
for M in VolumeReadOps VolumeWriteOps VolumeQueueLength BurstBalance; do
  echo "== $M"; aws cloudwatch get-metric-statistics --namespace AWS/EBS --metric-name $M \
    --dimensions Name=VolumeId,Value=vol-<id> --statistics Average --period 300 \
    --start-time $(date -u -d '3 hours ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --query 'Datapoints[-5:].[Timestamp,Average]' --output text
done

# 4. Network limits?
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name NetworkPacketsIn \
  --dimensions Name=InstanceId,Value=i-<id> --statistics Sum --period 60 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)
```

### The usual culprits

| Symptom | Cause | Fix |
|---|---|---|
| Fast then slow, repeating | `t3`/`t2` burst credits exhausted | Move to `m`/`c`/`r` family, or enable unlimited mode (costs more) |
| Disk-bound app much slower | Source had local NVMe or a fast SAN; target is gp3 baseline | Increase gp3 IOPS/throughput, use io2, or an instance store family (`i4i`, `m6id`) |
| Every DB query slower | App in AWS, DB still on-prem → WAN round-trip per query | Move them together, or add caching, or batch the queries |
| Slow only at peak | Undersized instance | Right-size using p95 with headroom |
| Sporadic latency spikes | Cross-AZ traffic between tiers | Keep chatty tiers in the same AZ (accept the AZ risk knowingly, or use read replicas) |
| Network throughput capped | Instance type's network baseline | Larger instance, or enable ENA/placement groups |
| DB slow after replatform | Parameter group defaults differ from your tuned on-prem config | Compare parameters one by one; recreate your tuning |
| Query plans changed | Statistics not gathered on the new engine | `ANALYZE` / `UPDATE STATISTICS` after the load |
| Windows file share slow | Insufficient FSx throughput capacity | Increase throughput capacity |

### Database-specific post-migration performance

```sql
-- PostgreSQL: statistics and vacuum after a bulk load
ANALYZE VERBOSE;
VACUUM (ANALYZE, VERBOSE);
SELECT * FROM pg_stat_user_tables WHERE last_analyze IS NULL;

-- MySQL
ANALYZE TABLE orders, customers;
SHOW ENGINE INNODB STATUS\G

-- Find the missing indexes DMS didn't create
-- (compare against the source's index list — DMS doesn't migrate secondary indexes
--  unless it created the schema)
SHOW INDEX FROM orders;   -- MySQL
SELECT * FROM pg_indexes WHERE tablename='orders';  -- PostgreSQL
```

Enable **Performance Insights** and let it tell you where the time goes rather than guessing.

### "Everything is slower" but metrics look fine

Compare against the baseline you captured pre-migration ([README §13.2](README.md#132-the-baseline-you-must-capture-before-you-touch-anything)). If you didn't capture one, you're now in an argument you can't win — capture one on the source before the *next* wave.

Also check the obvious human factors: users are now going through a VPN they weren't before; the ALB adds a hop; TLS termination moved; a proxy was inserted.

---

## 19. Cost surprises

### The bill is higher than the business case

Work through this list — it's almost always several of these together:

| Cause | Detect | Fix |
|---|---|---|
| No right-sizing (lift-and-shift the waste) | Compute Optimizer findings = `Overprovisioned` | Resize from p95 utilization |
| MGN staging resources left running | Volumes/instances tagged "Application Migration Service" still present | `finalize-cutover` on every server |
| Orphaned EBS volumes and snapshots | `describe-volumes --filters Name=status,Values=available` | Delete after the dark period |
| Duplicate snapshots from MGN PIT policy | Snapshot count exploding | Reduce PIT retention days |
| DMS replication instances still running | `describe-replication-instances` | Delete after cutover |
| No Savings Plans | Cost Explorer shows 100% On-Demand | Purchase after 2–4 weeks of stable usage |
| Data transfer: cross-AZ and egress | Cost Explorer → usage type `DataTransfer-Regional-Bytes` | Co-locate chatty tiers; VPC endpoints; CloudFront |
| NAT Gateway data processing | Often surprisingly large | S3/DynamoDB gateway endpoints are free; interface endpoints for the rest |
| Non-prod running 24/7 | Instances with `Environment=dev` running at 03:00 | Instance Scheduler / SSM automation — ~65% saving |
| gp2 instead of gp3 | `describe-volumes --filters Name=volume-type,Values=gp2` | Modify to gp3 |
| Over-provisioned RDS | Low `CPUUtilization`, high `FreeableMemory` | Downsize; consider Aurora Serverless v2 for spiky workloads |
| Multi-AZ on non-prod | RDS `MultiAZ: true` in dev | Single-AZ for non-prod |
| Unattached Elastic IPs | `describe-addresses` with null association | Release |
| Untagged resources with no owner | Resource Groups Tagging API | Enforce tags with Config + SCP |
| CloudWatch Logs retention = never expire | Log group storage growing forever | Set retention policies |

```bash
# Where is the money going?
aws ce get-cost-and-usage --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=USAGE_TYPE \
  --query 'ResultsByTime[0].Groups|sort_by(@,&to_number(Metrics.UnblendedCost.Amount))[-15:]' --output table

# Migration leftovers sweep
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[].{Id:VolumeId,GB:Size,Created:CreateTime}' --output table
aws ec2 describe-instances --filters Name=tag:Name,Values="*Application Migration Service*" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name]' --output text
aws dms describe-replication-instances --query 'ReplicationInstances[].[ReplicationInstanceIdentifier,ReplicationInstanceClass]' --output text
```

### Unexpected data transfer charges during migration

Replication traffic over the **public internet** or through a **NAT gateway** costs money per GB. Over Direct Connect it's much cheaper.

- Use `--data-plane-routing PRIVATE_IP` with VPN/DX where possible.
- Create S3 and MGN/DMS **VPC endpoints** so replication doesn't traverse NAT.
- Model the transfer cost for large estates before you start — for 100 TB it's a real line item.

---

## 20. Quota and limit errors

| Error | Quota | Command |
|---|---|---|
| `VcpuLimitExceeded` | Running On-Demand vCPUs per family | `aws service-quotas request-service-quota-increase --service-code ec2 --quota-code L-1216C47A --desired-value 500` |
| `VolumeLimitExceeded` / `MaxIOPSLimitExceeded` | EBS volumes / total IOPS / total TiB per region | Service Quotas → EBS |
| `AddressLimitExceeded` | 5 Elastic IPs per region by default | Release unused, or request more |
| `VpcLimitExceeded` | 5 VPCs per region | Request more |
| `SecurityGroupLimitExceeded` | 2,500 SGs per VPC / 5 per ENI | Consolidate rules |
| `RulesPerSecurityGroupLimitExceeded` | 60 inbound + 60 outbound | Use prefix lists |
| `SnapshotLimitExceeded` | Snapshots per region | Delete old ones; tune MGN PIT retention |
| `DBInstanceQuotaExceeded` | RDS instances per region (40) | Request more |
| `ReplicationInstanceQuotaExceeded` | DMS replication instances (20) | Delete finished ones |
| `TooManyRequestsException` | API rate limits | Exponential backoff; `aws configure set max_attempts 10` |
| `InstanceLimitExceeded` (MGN) | Max concurrent source servers | Migrate in smaller batches |

```bash
# Audit before a wave — do this a week ahead, increases take days
for Q in L-1216C47A L-34B43A08 L-D18FCD1D; do
  aws service-quotas get-service-quota --service-code ec2 --quota-code $Q \
    --query '{Name:Quota.QuotaName,Value:Quota.Value}' --output text
done
aws service-quotas list-requested-service-quota-change-history --service-code ec2 \
  --query 'RequestedQuotas[].{Quota:QuotaName,Desired:DesiredValue,Status:Status}' --output table
```

---

## 21. Licensing and activation

| Problem | Cause | Fix |
|---|---|---|
| Windows won't activate | Can't reach the AWS KMS endpoint | Allow `169.254.169.250:1688`; `slmgr /skms 169.254.169.250:1688 && slmgr /ato` |
| BYOL Windows won't run | BYOL requires Dedicated Hosts + licence mobility | Use licence-included (`osByol=false`), or set up a Dedicated Host |
| SQL Server edition mismatch | Licence-included AMI edition differs from the source | Migrate to RDS, or bring the licence on a Dedicated Host |
| Oracle licence cost explodes | vCPU-based counting differs from on-prem socket licensing | Model it before committing; this is what drives Aurora conversion |
| Third-party app licence tied to hardware (MAC/UUID/dongle) | The MAC and UUID change | Contact the vendor for a re-host key; some allow MAC-preserving configs; USB dongles may make the workload **Retain** |
| RHEL/SUSE subscription invalid | Source used an on-prem subscription | Use Cloud Access / BYOS, or migrate to an AWS-provided AMI |
| Per-socket licensing product | Cloud has no sockets | Renegotiate to per-core or per-user |

Use **AWS License Manager** to track and enforce entitlements across the migrated estate rather than tracking them in a spreadsheet.

---

## 22. Container and refactor problems

| Problem | Cause | Fix |
|---|---|---|
| A2C `inventory` finds nothing | Unsupported app type, or not running | A2C handles Java/.NET; the app must be running when you analyze |
| A2C container won't start | Missing dependencies or a wrong base image | Edit `analysis.json` — base image, env vars, included files — and re-containerize |
| Container works locally, fails on ECS | Missing env vars/secrets, or no IAM task role | Check the CloudWatch log stream for the task; verify Parameter Store/Secrets ARNs |
| `CannotPullContainerError` | No route to ECR, or missing permissions | ECR + ECR API + S3 endpoints (or NAT); `AmazonECSTaskExecutionRolePolicy` on the execution role |
| Task keeps restarting | Health check failing, or OOM | `describe-tasks` → `stoppedReason`; raise memory; fix the health check path/timeouts |
| Sessions lost when a task recycles | Session state was on local disk | Externalise to ElastiCache or DynamoDB — this is required, not optional |
| App writes to the local filesystem | Containers are ephemeral | Use EFS volumes, or refactor to S3 |
| Windows container image is enormous | Windows base images are large | Use the smallest suitable base; expect slower pulls; consider porting to .NET on Linux |
| Refactor Spaces route not working | Route inactive, or the service can't be reached | `list-routes` → check `ActivationState`; verify TGW attachments and the service endpoint |
| Latency increased after containerizing | Extra hops (ALB → API Gateway → VPC Link) | Simplify the path; measure each hop |

---

## 23. Rollback situations

### Rehost rollback (easy)

```bash
# 1. Revert DNS to the source
aws route53 change-resource-record-sets --hosted-zone-id <ZID> --change-batch file://revert-dns.json
# 2. Start services on the source
ssh source-server 'sudo systemctl start nginx mysql'
# 3. Stop, DO NOT terminate, the target
aws ec2 stop-instances --instance-ids i-<target>
# 4. Replication is still running if you haven't finalized — you can retry the cutover later
aws mgn describe-source-servers --filters sourceServerIDs=<s-id> \
  --query 'items[0].dataReplicationInfo.dataReplicationState'
```

If you already ran `finalize-cutover`, replication has stopped and the staging area is gone. You'd need to reinstall the agent and re-replicate. **This is why you finalize after hypercare, not immediately after cutover.**

### Database rollback (hard — plan it in advance)

Once writes have landed on the new database, rolling back means moving those writes back.

**Option 1 — reverse DMS task (set this up BEFORE cutover):**

```bash
aws dms create-replication-task --replication-task-identifier reverse-aurora-to-mysql \
  --source-endpoint-arn <new-target-as-source> --target-endpoint-arn <old-source-as-target> \
  --replication-instance-arn <ri-arn> --migration-type cdc \
  --table-mappings file://mappings.json
# Start it immediately after cutover; it becomes your rollback path
aws dms start-replication-task --replication-task-arn <reverse-arn> --start-replication-task-type start-replication
```

**Option 2 — restore the pre-cutover snapshot and manually replay:** identify all writes since cutover, export them, replay on the source. Slow, error-prone, and sometimes the only option. Have the queries pre-written.

**Option 3 — accept forward-fix only.** For many systems this is the honest answer: past the point of no return, you fix forward. **Say so explicitly in the runbook** so nobody assumes a rollback exists.

### Refactor rollback (easy, by design)

```bash
# Set the new service's weight to 0; all traffic returns to the monolith
aws elbv2 modify-listener --listener-arn <arn> --default-actions '[{
 "Type":"forward","ForwardConfig":{"TargetGroups":[
   {"TargetGroupArn":"<monolith-tg>","Weight":100},
   {"TargetGroupArn":"<new-tg>","Weight":0}]}}]'
```

This is the entire argument for the strangler fig pattern.

### Post-rollback checklist

```
[ ] Users notified that the rollback is complete and the old system is live
[ ] Source is serving traffic and validated (run the smoke tests against it)
[ ] Target stopped but PRESERVED for investigation
[ ] Any writes that landed on the target identified and reconciled
[ ] Data integrity confirmed on the source
[ ] Root cause documented while it's fresh
[ ] Runbook updated with what you learned
[ ] Next attempt scheduled with the fix in place — and the fix tested first
```

---

## 24. Where to find logs

| Component | Location |
|---|---|
| MGN agent (Linux) | `/var/lib/aws-replication-agent/agent.log.0` (and `.1`, `.2`) |
| MGN agent (Windows) | `C:\Program Files (x86)\AWS Replication Agent\agent.log.0` |
| MGN jobs | `aws mgn describe-job-log-items --job-id <id>` |
| MGN post-launch actions | SSM Run Command history / the SSM document's output |
| Discovery agent (Linux) | `/var/log/aws/discovery/aws-discovery-daemon.log` |
| Discovery agent (Windows) | `C:\ProgramData\AWS Discovery\Logs\` |
| DMS task | CloudWatch Logs group `dms-tasks-<replication-instance-id>` |
| DMS control tables | On the target: `awsdms_apply_exceptions`, `awsdms_suspended_tables`, `awsdms_validation_failures_v1`, `awsdms_status` |
| DataSync | The CloudWatch log group you set on the task |
| EC2 boot | `aws ec2 get-console-output` / `get-console-screenshot` |
| EC2Launch (Windows) | `C:\ProgramData\Amazon\EC2Launch\log\` |
| cloud-init (Linux) | `/var/log/cloud-init.log`, `/var/log/cloud-init-output.log` |
| SSM agent | `/var/log/amazon/ssm/` · `C:\ProgramData\Amazon\SSM\Logs\` |
| RDS | `aws rds describe-db-log-files` + `download-db-log-file-portion`; CloudWatch Logs exports |
| VPC traffic | VPC Flow Logs |
| API calls (who did what) | CloudTrail |
| ALB access | S3 access-log bucket (enable it — off by default) |

```bash
# Fast log commands worth memorising
aws logs tail dms-tasks-<ri-id> --follow --format short
aws logs filter-log-events --log-group-name dms-tasks-<ri-id> --filter-pattern "ERROR"
aws ec2 get-console-output --instance-id i-<id> --output text | tail -80
aws mgn describe-job-log-items --job-id <id> --query 'items[].{T:logDateTime,E:event,D:eventData}' --output table
aws rds download-db-log-file-portion --db-instance-identifier appdb-target \
  --log-file-name error/postgresql.log --output text | tail -100
```

---

## 25. Escalation checklist

Before opening a support case (or asking a colleague), gather this. Having it ready typically turns a multi-day back-and-forth into a single response.

```
IDENTIFIERS
  Account ID:                    Region:
  Migration Hub home region:
  Source server ID (MGN):        Job ID:
  DMS task ARN:                  Replication instance:
  Instance ID(s):                Volume ID(s):

WHAT'S HAPPENING
  Exact error message (copy-paste, don't paraphrase):
  What you were doing when it happened:
  When it started (UTC timestamp):
  Is it consistent or intermittent?
  How many servers/tasks affected — one, or all?

WHAT CHANGED
  Anything modified in the last 24h? (SG, IAM, quota, agent version, OS patch)
  Did this ever work? When did it last work?

SOURCE ENVIRONMENT
  OS + version + kernel:         Boot mode (BIOS/UEFI):
  Disk layout and total size:    Hypervisor (or physical):
  Agent version:

WHAT YOU'VE ALREADY CHECKED
  [ ] Security groups            [ ] NACLs (both directions)
  [ ] Route tables               [ ] DNS resolution
  [ ] IAM permissions + SCPs     [ ] Service quotas
  [ ] Endpoint reachability      [ ] Clock sync
  [ ] Relevant logs (attach them)

BUSINESS CONTEXT
  Is a cutover window at risk? When?
  Is production affected right now?
  Rollback available? Yes/No — describe it.
```

### General debugging discipline

1. **Read the actual error.** AWS migration errors are unusually specific. `describe-*` on the failing resource almost always names the failing step.
2. **Change one thing at a time.** Otherwise you won't know what fixed it, and you'll hit it again next wave.
3. **Check region and account first.** A startling share of "impossible" problems are a wrong `AWS_REGION`.
4. **Is it DNS, routing, security groups, or IAM?** Four questions that cover most connectivity failures. NACLs are stateless — check both directions.
5. **Reachability Analyzer beats guessing** for any "A can't talk to B".
6. **Console output and screenshots** solve boot problems in seconds. Use them before anything else.
7. **Test in isolation.** A test launch in an isolated subnet tells you whether the problem is the instance or the environment.
8. **Write down what you find.** Wave 2 will hit the same thing. This document exists because someone did that.

---

*Back to → [README.md](README.md) · [commands-cheatsheet.md](commands-cheatsheet.md) · [hands-on-labs.md](hands-on-labs.md)*
