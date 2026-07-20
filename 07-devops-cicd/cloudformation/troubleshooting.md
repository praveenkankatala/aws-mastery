# AWS CloudFormation — Troubleshooting Guide

Every entry follows **What** (the symptom/error) → **Why** (root cause) → **How** (the fix). 

> 🔍 **Golden rule:** the answer is almost always in the stack events. Find the **first** `*_FAILED` event (everything after it is cascade/rollback noise):
>
> ```bash
> aws cloudformation describe-stack-events --stack-name my-stack \
>   --query "StackEvents[?contains(ResourceStatus,'FAILED')].[LogicalResourceId,ResourceStatusReason]" \
>   --output table
> ```

---

## 1. Stack Creation Failures

### 1.1 `ROLLBACK_COMPLETE` — stack can't be updated

**What:** initial create failed, everything rolled back, and now `update-stack` returns *"Stack ... is in ROLLBACK_COMPLETE state and can not be updated."*
**Why:** a stack whose **first creation** failed has no known-good state to update from. `ROLLBACK_COMPLETE` after a *create* is a terminal state.
**How:**
```bash
aws cloudformation delete-stack --stack-name my-stack
aws cloudformation wait stack-delete-complete --stack-name my-stack
# fix the template, then create again
```
To debug *before* rollback wipes the evidence, create with `--on-failure DO_NOTHING` (or `--disable-rollback`), inspect the failed resources, then delete.

### 1.2 `Requires capabilities : [CAPABILITY_IAM]` (InsufficientCapabilitiesException)

**What:** create/update rejected mentioning capabilities.
**Why:** the template creates IAM resources; AWS requires explicit acknowledgment that you understand the security impact. Named IAM resources need the stronger flag; macros need `AUTO_EXPAND`.
**How:** add the flag it names:
```bash
--capabilities CAPABILITY_IAM            # unnamed IAM resources
--capabilities CAPABILITY_NAMED_IAM      # IAM resources with explicit names
--capabilities CAPABILITY_AUTO_EXPAND    # templates using transforms/macros
```

### 1.3 `Resource already exists` / `AlreadyExistsException`

**What:** e.g., *"my-bucket already exists"* during create.
**Why:** you hardcoded a physical name (BucketName, RoleName, etc.) that exists — in your account, or for S3, **anywhere in the world**. Or a previous stack retained the resource.
**How:**
- Remove the explicit name and let CloudFormation generate one (best practice), or
- Namespace names: `!Sub "${AWS::StackName}-logs"`, or
- If the resource should be managed by this stack, **import** it (`--change-set-type IMPORT`).

### 1.4 Template validation errors

**What:** `ValidationError: Template format error / Unresolved resource dependencies / Invalid template property`.
**Why (common causes):**
- YAML indentation errors, tabs instead of spaces
- `!Ref` to a logical ID or parameter that doesn't exist (typo)
- Property placed at resource level instead of under `Properties`
- Short-form functions nested illegally (`!Base64 !Sub ...` → use `Fn::Base64: !Sub`)
**How:** run `cfn-lint template.yaml` — it pinpoints line numbers and explains most of these before AWS ever sees the template.

### 1.5 `Circular dependency between resources`

**What:** create fails immediately with a circular dependency error listing resource names.
**Why:** A references B and B references A — classically two security groups referencing each other's ingress rules inline.
**How:** break the cycle by extracting the referencing piece into a separate resource:
```yaml
# Instead of inline SecurityGroupIngress on both SGs:
SGAIngressFromB:
  Type: AWS::EC2::SecurityGroupIngress
  Properties:
    GroupId: !GetAtt SGA.GroupId
    SourceSecurityGroupId: !GetAtt SGB.GroupId
    IpProtocol: tcp
    FromPort: 443
    ToPort: 443
```
Also check for unnecessary `DependsOn` creating loops.

### 1.6 `Template may not exceed 51200 bytes`

**What:** create rejected on size.
**Why:** inline (`--template-body`) templates cap at ~51 KB.
**How:** upload to S3 and use `--template-url` (limit 1 MB), or split into nested stacks. If you're near the **500 resources** limit, splitting is mandatory anyway.

### 1.7 IAM permission errors — `API: ec2:CreateVpc User ... is not authorized`

**What:** a resource fails with an explicit *not authorized* reason.
**Why:** CloudFormation calls services **as the caller** (or the stack's service role). Whichever principal is used lacks the permission — possibly blocked by an SCP or permissions boundary.
**How:**
- Grant the missing permission to the user/role, or
- Use a dedicated service role: `--role-arn arn:aws:iam::<acct>:role/cfn-deployer` (then users only need `cloudformation:*` + `iam:PassRole` on that role).
- Decode cryptic authorization failures: `aws sts decode-authorization-message --encoded-message <msg>`.

---

## 2. Update & Rollback Failures

### 2.1 `UPDATE_ROLLBACK_FAILED` — the scariest state

**What:** an update failed, the rollback *also* failed; stack is frozen.
**Why:** rollback couldn't restore a resource — someone changed/deleted it manually outside CloudFormation, a dependency is gone, insufficient permissions mid-rollback, or a nested stack is out of sync.
**How:**
```bash
# Fix the underlying resource manually if possible, then:
aws cloudformation continue-update-rollback --stack-name my-stack

# If a specific resource can never be restored, skip it:
aws cloudformation continue-update-rollback --stack-name my-stack \
  --resources-to-skip BrokenResource NestedStack.ResourceInside
```
⚠️ Skipped resources become unmanaged/inconsistent — reconcile them (fix manually + drift-detect, or re-import) afterwards. **Never** try to work around this state by deleting the stack first; resolve the rollback, then decide.

### 2.2 `No updates are to be performed`

**What:** `update-stack` errors even though "nothing changed" is exactly what you expected.
**Why:** the submitted template + parameters are byte-identical in effect to the current state. It's an error, not a warning — annoying in pipelines.
**How:** in CI use `aws cloudformation deploy ... --no-fail-on-empty-changeset`, or create a change set and check `Status: FAILED` + reason *"didn't contain changes"* before executing.

### 2.3 Update unexpectedly **replaced** a resource (data loss!)

**What:** after an update, a resource has a new physical ID; data/endpoint changed.
**Why:** you changed an **immutable property** (e.g., RDS `DBInstanceIdentifier`, EC2 subnet, most `*Name` properties) — CloudFormation creates a new resource, then deletes the old one.
**How (prevention — this one you can't undo):**
- **Always** run a change set and check the `Replacement` column before prod updates.
- Set `UpdateReplacePolicy: Retain/Snapshot` on stateful resources so the old resource/data survives replacement.
- Protect crown jewels with a **stack policy** denying `Update:Replace`/`Update:Delete`.

### 2.4 Update blocked by stack policy

**What:** *"Action denied by stack policy"*.
**Why:** the stack policy explicitly denies updates to that resource — working as intended.
**How:** if the change is legitimate, pass a temporary override:
```bash
aws cloudformation update-stack ... \
  --stack-policy-during-update-body '{"Statement":[{"Effect":"Allow","Action":"Update:*","Principal":"*","Resource":"*"}]}'
```
The permanent policy is restored automatically after the update.

### 2.5 Stack stuck `IN_PROGRESS` for a very long time

**What:** `UPDATE_IN_PROGRESS`/`CREATE_IN_PROGRESS` for 30+ minutes with no new events.
**Why (usual suspects):**
- A **custom resource** Lambda that never sent its response (waits up to ~1–3 hours before timing out)
- A `CreationPolicy`/WaitCondition waiting for a cfn-signal that will never come
- Genuinely slow resources (RDS Multi-AZ, CloudFront, ACM DNS validation can take 20–40+ min)
**How:**
- Check which resource is pending in events; if it's `Custom::*`, check the Lambda's CloudWatch logs — ensure it *always* calls `cfnresponse.send()` even in `except` blocks.
- `cancel-update-stack` works for updates; creates must time out or be abandoned via console "delete".
- For the future: set `--timeout-in-minutes` and realistic `Timeout` values in CreationPolicies.

---

## 3. Deletion Failures

### 3.1 `DELETE_FAILED` — S3 bucket not empty

**What:** *"The bucket you tried to delete is not empty."*
**Why:** CloudFormation will not delete non-empty buckets (versioned buckets also hide delete markers/old versions).
**How:**
```bash
aws s3 rm s3://bucket-name --recursive        # plus versions if versioned:
aws s3api delete-objects --bucket bucket-name \
  --delete "$(aws s3api list-object-versions --bucket bucket-name \
  --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)"
aws cloudformation delete-stack --stack-name my-stack   # retry
```
Long-term: a small **custom resource** that empties the bucket on delete, or `DeletionPolicy: Retain` and clean up out-of-band.

### 3.2 `DELETE_FAILED` — security group / ENI dependency

**What:** *"resource sg-xxx has a dependent object"* or ENI deletion timeouts.
**Why:** something outside the stack (or a Lambda-in-VPC's lingering ENI) still uses the SG/subnet.
**How:** find the dependency:
```bash
aws ec2 describe-network-interfaces --filters Name=group-id,Values=sg-xxx
```
Detach/delete the dependent, retry. Lambda ENIs can take several minutes to release after function deletion — retrying later often just works. Last resort: `delete-stack --retain-resources <LogicalId>` (only valid from `DELETE_FAILED`), then clean up manually.

### 3.3 Delete blocked — `Export ... is in use by another stack`

**What:** deletion (or an update that removes an Output export) fails naming the export.
**Why:** cross-stack protection: importing stacks would break.
**How:**
```bash
aws cloudformation list-imports --export-name the-export-name
```
Update/delete the **importing** stacks first (replace `!ImportValue` with a parameter or literal), then delete this one.

### 3.4 Delete blocked — termination protection

**What:** *"Stack cannot be deleted while TerminationProtection is enabled."*
**How:** deliberate two-step: 
```bash
aws cloudformation update-termination-protection --stack-name my-stack --no-enable-termination-protection
aws cloudformation delete-stack --stack-name my-stack
```

### 3.5 Retained resources surprise

**What:** stack deleted successfully but resources (and costs) remain.
**Why:** `DeletionPolicy: Retain`/`Snapshot` on those resources — by design.
**How:** this is expected; list them from the final `DELETE_COMPLETE`/`DELETE_SKIPPED` events and clean up manually if truly unwanted.

---

## 4. EC2 Bootstrapping Issues (cfn-init / cfn-signal)

### 4.1 `Failed to receive N resource signal(s) within the specified duration`

**What:** instance/ASG creation rolls back on signal timeout.
**Why:** cfn-signal never ran, or UserData crashed before reaching it, or the instance has **no network path** to the CloudFormation endpoint (private subnet without NAT/VPC endpoint), or the timeout is simply too short, or (for ASGs) `Count` exceeds instances actually launched.
**How:**
1. Recreate with rollback disabled, SSH/SSM into the instance.
2. Read, in order: `/var/log/cloud-init-output.log`, `/var/log/cfn-init.log`, `/var/log/cfn-init-cmd.log`.
3. Use `#!/bin/bash -xe` in UserData so failures are loud and early.
4. Ensure `cfn-signal` runs with `-e $?` **immediately after** `cfn-init`.
5. Private subnets: add a NAT gateway or a `com.amazonaws.<region>.cloudformation` interface VPC endpoint.
6. Bump `Timeout: PT15M` → `PT30M` for slow installs.

### 4.2 cfn-init succeeded but config didn't apply on **update**

**What:** you changed `AWS::CloudFormation::Init` metadata, updated the stack, nothing happened on the instance.
**Why:** metadata changes don't touch running instances unless **cfn-hup** is installed and running — UserData only executes at first boot.
**How:** install/enable cfn-hup in your bootstrap (config in `/etc/cfn/cfn-hup.conf` + hook re-running cfn-init), or bake changes into the Launch Template and roll instances via `UpdatePolicy: AutoScalingRollingUpdate`.

---

## 5. Cross-Cutting Issues

### 5.1 Drift — "the template says X, reality says Y"

**What:** stack behaves unexpectedly; drift detection shows `MODIFIED`/`DELETED` resources.
**Why:** console/CLI changes outside CloudFormation.
**How:** decide the source of truth: revert manual changes, **or** update the template to codify them, **or** re-import deleted-and-recreated resources. Then update the stack and re-run drift detection until `IN_SYNC`. Prevent recurrence with IAM (deny direct mutations in prod) and scheduled drift checks.

### 5.2 `Rate exceeded` (Throttling)

**What:** intermittent throttling errors during big deployments/StackSet operations.
**Why:** too many concurrent API calls in the account (CloudFormation + everything else shares service limits).
**How:** CloudFormation retries automatically most of the time; for chronic cases reduce parallelism (StackSet `MaxConcurrentCount`), split giant stacks, stagger pipeline deployments, or request a service quota increase.

### 5.3 Change set shows changes you didn't make (`Transform` templates)

**What:** SAM/macro templates show large diffs for tiny edits.
**Why:** the transform re-expands the whole template; generated logical IDs can shift.
**How:** expected behavior — review the *processed* template (`get-template --template-stage Processed`) to compare like-with-like.

### 5.4 Parameter validation failures

**What:** *"Parameter 'X' must match pattern..."*, or a value rejected despite looking right.
**Why:** `AllowedPattern`/`AllowedValues`/length constraints; or an AWS-typed parameter (e.g., `AWS::EC2::KeyPair::KeyName`) referencing something that doesn't exist **in this region**.
**How:** read the `ConstraintDescription`; confirm region; for lists, pass CLI values comma-separated with the whole thing quoted: `ParameterKey=Subnets,ParameterValue=\"subnet-a,subnet-b\"`.

### 5.5 Nested stack failed — root just says "Embedded stack ... was not successfully created"

**What:** the root stack's error is useless.
**Why:** the real error lives in the **child** stack's events.
**How:** `describe-stack-events` on the child's stack ID (visible in the root's events / console). Fix in the child template, redeploy **the root**. Never operate directly on nested stacks.

### 5.6 StackSet instances stuck `OUTDATED` / operation `FAILED`

**What:** some accounts didn't take the update.
**Why:** missing/mis-trusted execution role in the target account, SCP blocking, or failure tolerance stopped the rollout.
**How:** `describe-stack-set-operation` + `list-stack-set-operation-results` for per-account reasons; verify `AWSCloudFormationStackSetExecutionRole` exists and trusts the admin account; re-run with corrected roles.

---

## 6. Debugging Workflow (memorize this)

```
1. describe-stack-events → find the FIRST *_FAILED event
2. Read ResourceStatusReason word by word (the answer is usually literal)
3. Classify: permission? | limit/quota? | bad property? | dependency? | external state?
4. Reproduce fast: --on-failure DO_NOTHING, small isolated template
5. Fix → cfn-lint → change set → execute
6. If state is wedged: continue-update-rollback / retain-resources / import
7. Post-fix: drift detect, add the guard (policy, lint rule, cfn-guard) that
   would have caught it — turn every incident into a pipeline check
```

---

*Related: [`commands-cheatsheet.md`](commands-cheatsheet.md) §11 for recovery commands · [`hands-on-labs.md`](hands-on-labs.md) Lab 6 deliberately breaks a stack so you can practice all of this safely.*
