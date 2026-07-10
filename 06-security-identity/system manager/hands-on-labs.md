# AWS SSM — Hands-On Labs

Practical labs to build real muscle memory with AWS Systems Manager. Do them in order — later labs assume earlier ones are done. Each lab lists an **objective**, **prerequisites**, **steps**, **validation**, and **cleanup**.

> 💰 **Cost note:** Most labs use `t3.micro`/`t2.micro` instances, which are free-tier eligible. Always run the cleanup steps to avoid charges.

---

## Lab 0 — Prerequisites: IAM Role + Launch a Manageable Instance

**Objective:** Stand up an EC2 instance that SSM can actually see and control.

**Steps:**
1. Create an IAM role for EC2 with the managed policy `AmazonSSMManagedInstanceCore` attached.
   ```bash
   aws iam create-role \
     --role-name SSMInstanceRole \
     --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

   aws iam attach-role-policy \
     --role-name SSMInstanceRole \
     --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

   aws iam create-instance-profile --instance-profile-name SSMInstanceProfile
   aws iam add-role-to-instance-profile \
     --instance-profile-name SSMInstanceProfile \
     --role-name SSMInstanceRole
   ```
2. Launch an EC2 instance (Amazon Linux 2023 already bundles the SSM Agent) with this instance profile attached, **and no key pair required** since you'll use Session Manager.
   ```bash
   aws ec2 run-instances \
     --image-id <amazon-linux-2023-ami-id> \
     --instance-type t3.micro \
     --iam-instance-profile Name=SSMInstanceProfile \
     --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ssm-lab-instance}]'
   ```
3. Make sure the instance's security group allows **outbound HTTPS (443)** — Session Manager needs this to reach SSM endpoints (no inbound rules needed at all).

**Validation:**
```bash
aws ssm describe-instance-information --filters "Key=tag:Name,Values=ssm-lab-instance"
```
You should see `PingStatus: Online`. If not, jump to [troubleshooting.md](./troubleshooting.md#instance-not-showing-up-in-fleet-manager).

**Cleanup:** Keep this instance running — the rest of the labs use it.

---

## Lab 1 — Verify the Agent & Explore Fleet Manager

**Objective:** Get comfortable reading fleet-wide instance data.

**Steps:**
1. Confirm agent version and status:
   ```bash
   aws ssm describe-instance-information --instance-information-filter-list "key=InstanceIds,valueSet=<instance-id>"
   ```
2. Pull inventory data (installed apps, network config):
   ```bash
   aws ssm list-inventory-entries --instance-id <instance-id> --type-name AWS:Application
   ```

**Validation:** Confirm you can see OS, agent version, and IP address details returned in the JSON.

---

## Lab 2 — Session Manager: Log In Without SSH

**Objective:** Prove you can access a shell with zero open inbound ports and zero SSH keys.

**Steps:**
1. Install the Session Manager plugin locally (see [commands-cheatsheet.md](./commands-cheatsheet.md#session-manager)).
2. Start a session:
   ```bash
   aws ssm start-session --target <instance-id>
   ```
3. Run a few commands inside the session (`whoami`, `uptime`), then exit:
   ```bash
   exit
   ```
4. Try port forwarding to reach a local service (e.g. a database) without exposing it publicly:
   ```bash
   aws ssm start-session \
     --target <instance-id> \
     --document-name AWS-StartPortForwardingSession \
     --parameters '{"portNumber":["3306"],"localPortNumber":["9999"]}'
   ```

**Validation:** You got a shell prompt with no SSH key and no inbound rule. Confirm the session is logged:
```bash
aws ssm describe-sessions --state Active
```

---

## Lab 3 — Parameter Store: Config & Secrets

**Objective:** Store both plain config and an encrypted secret, then read them back.

**Steps:**
1. Store a plaintext config value:
   ```bash
   aws ssm put-parameter --name "/lab/app/log-level" --value "INFO" --type String
   ```
2. Store an encrypted secret:
   ```bash
   aws ssm put-parameter --name "/lab/db/password" --value "Sup3rSecret!" --type SecureString --key-id alias/aws/ssm
   ```
3. Read both back:
   ```bash
   aws ssm get-parameter --name "/lab/app/log-level"
   aws ssm get-parameter --name "/lab/db/password" --with-decryption
   ```
4. From inside a Session Manager shell on your instance, fetch a parameter using the instance's own IAM role (this is how real apps consume config):
   ```bash
   aws ssm get-parameter --name "/lab/app/log-level" --region <region>
   ```

**Validation:** The instance can read the parameter using its attached IAM role — no hardcoded credentials.

**Cleanup:**
```bash
aws ssm delete-parameter --name "/lab/app/log-level"
aws ssm delete-parameter --name "/lab/db/password"
```

---

## Lab 4 — Run Command: Fleet-Wide Execution

**Objective:** Execute a command across multiple instances at once, without logging into any of them.

**Steps:**
1. (Optional) Launch a second lab instance and tag both with `Environment=Lab`.
2. Send a command to all tagged instances:
   ```bash
   aws ssm send-command \
     --targets "Key=tag:Environment,Values=Lab" \
     --document-name "AWS-RunShellScript" \
     --parameters commands="df -h && uptime"
   ```
3. Check results:
   ```bash
   aws ssm list-command-invocations --command-id <command-id> --details
   ```

**Validation:** You see output from every targeted instance in a single call, with per-instance success/failure status.

---

## Lab 5 — Patch Manager: Baseline & Scan

**Objective:** Scan and patch your instance using a managed patch baseline.

**Steps:**
1. Tag the instance for patching:
   ```bash
   aws ec2 create-tags --resources <instance-id> --tags Key=PatchGroup,Value=Lab
   ```
2. Run a patch **scan** (no changes made yet):
   ```bash
   aws ssm send-command \
     --document-name "AWS-RunPatchBaseline" \
     --targets "Key=tag:PatchGroup,Values=Lab" \
     --parameters "Operation=Scan"
   ```
3. Review what's missing:
   ```bash
   aws ssm describe-instance-patch-states --instance-ids <instance-id>
   ```
4. Now actually **install** patches:
   ```bash
   aws ssm send-command \
     --document-name "AWS-RunPatchBaseline" \
     --targets "Key=tag:PatchGroup,Values=Lab" \
     --parameters "Operation=Install"
   ```

**Validation:** Compliance state moves from `NON_COMPLIANT` to `COMPLIANT` (or close to it) after install.

---

## Lab 6 — Automation: Build a Custom AMI

**Objective:** Run a built-in Automation runbook to snapshot your instance into a reusable AMI.

**Steps:**
```bash
aws ssm start-automation-execution \
  --document-name "AWS-CreateImage" \
  --parameters "InstanceId=<instance-id>,ImageName=lab-ami-$(date +%s),NoReboot=false"
```
Track progress:
```bash
aws ssm describe-automation-executions \
  --filters "Key=DocumentNamePrefix,Values=AWS-CreateImage"
```

**Validation:** A new AMI appears in the EC2 console once the automation status shows `Success`.

**Cleanup:** Deregister the AMI and delete its snapshot once you're done.

---

## Lab 7 — AppConfig: Feature Flag Rollout

**Objective:** Deploy a feature flag to a "running application" safely, without redeploying code.

**Steps:**
1. Create the application and profile (see cheatsheet for full commands).
2. Create a flag payload file `flags.json`:
   ```json
   { "newCheckoutFlow": { "enabled": true } }
   ```
3. Deploy it gradually using a linear deployment strategy (e.g. 20% every 5 minutes) rather than all at once.
4. Watch the deployment state move through `BAKING` → `DEPLOYING` → `COMPLETE`.

**Validation:** You can pause/roll back a deployment mid-flight if you simulate an error — this is the core value of AppConfig over a code deploy.

---

## Lab 8 — Parameter Store vs Secrets Manager, Side by Side

**Objective:** Feel the practical difference the comparison table describes.

**Steps:**
1. Store the same fake DB password in both services (see cheatsheet commands for each).
2. Enable rotation on the **Secrets Manager** version — notice it's a couple of CLI calls with a Lambda ARN.
3. Try to enable "native" rotation on the **Parameter Store** version — notice there isn't one; you'd have to hand-write a Lambda + EventBridge schedule yourself.
4. Try replicating each to a second region:
   - Secrets Manager: one CLI call (`replicate-secret-to-regions`).
   - Parameter Store: no native equivalent — you'd script a copy via CLI/CI-CD.

**Validation:** You can now explain, from direct experience, why Secrets Manager costs more but buys you rotation and replication out of the box.

---

## Lab 9 — OpsCenter: Track and Resolve an Issue

**Objective:** Use OpsCenter as a lightweight ticketing system tied to your infrastructure.

**Steps:**
1. Create an OpsItem describing a fake incident:
   ```bash
   aws ssm create-ops-item \
     --title "High memory on lab instance" \
     --description "Memory usage exceeded 85%" \
     --source "ManualLabExercise" \
     --priority 3
   ```
2. List open items and inspect the one you created:
   ```bash
   aws ssm describe-ops-items --ops-item-filters "Key=Status,Values=Open,Operator=Equal"
   ```
3. Resolve it:
   ```bash
   aws ssm update-ops-item --ops-item-id <ops-item-id> --status Resolved
   ```

**Validation:** The item moves from `Open` to `Resolved` and disappears from your open-items filter.

---

## Lab 10 — Capstone: End-to-End Governed Patch Workflow

**Objective:** Combine Change Manager (approval), Automation (execution), and Session Manager (verification) into one realistic workflow — the way a production team would actually patch a fleet.

**Steps:**
1. Submit a change request referencing the `AWS-RunPatchBaseline` runbook, scheduled for a future maintenance window.
2. Have it approved (in-console, under Change Manager, as a second "approver" persona if possible).
3. Once approved, confirm the Automation execution actually ran the patch job.
4. Use Session Manager to manually verify the patch level on the instance:
   ```bash
   aws ssm start-session --target <instance-id>
   # then, inside the session:
   cat /etc/os-release
   sudo yum history | head -5
   ```
5. Close the loop by creating an OpsItem if anything looks wrong, or marking the change complete if it doesn't.

**Validation:** You've now touched all four SSM pillars in a single realistic workflow — this is the story to tell in an interview or on your GitHub profile.

**Cleanup:** Terminate the lab EC2 instance(s), delete any test parameters/secrets, deregister test AMIs, and delete the patch baseline/AppConfig application if no longer needed.
