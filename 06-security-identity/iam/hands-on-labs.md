# AWS IAM — Hands-On Labs

12 progressive labs. Do them in order — each one builds on identities or policies created in the last. Use a **sandbox/personal AWS account**, never production.

> 💰 **Cost note:** every lab in this file uses only Free-Tier-eligible resources (IAM itself is free; the one `t3.micro` EC2 instance in Lab 4 falls under Free Tier for new accounts). Terminate/delete resources at the end of each lab.
>
> 🧹 **Cleanup:** each lab ends with a "Clean up" step. Don't skip it — leftover IAM users/roles are a common source of security debt.

---

## Lab 0 — Prerequisites

1. An AWS account with console + CLI access.
2. AWS CLI v2 installed: `aws --version`
3. Run `aws configure` with an admin identity (only for *building* these labs — not for daily driving).
4. Create a working folder:
   ```bash
   mkdir ~/iam-labs && cd ~/iam-labs
   ```

---

## Lab 1 — Users, Groups, and Your First Policy

**Goal:** experience deny-by-default, then grant access via a Group.

1. Create a user with no permissions:
   ```bash
   aws iam create-user --user-name lab-alice
   aws iam create-access-key --user-name lab-alice
   ```
2. Configure a second CLI profile with those keys: `aws configure --profile lab-alice`
3. Try to list S3 buckets as `lab-alice`:
   ```bash
   aws s3 ls --profile lab-alice
   ```
   → **Expect `AccessDenied`.** This is deny-by-default in action.
4. Create a group and attach an AWS-managed policy:
   ```bash
   aws iam create-group --group-name lab-readonly
   aws iam attach-group-policy --group-name lab-readonly \
     --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
   aws iam add-user-to-group --user-name lab-alice --group-name lab-readonly
   ```
5. Retry step 3 — it now succeeds.
6. **Try a write action** (`aws s3 mb s3://some-new-bucket --profile lab-alice`) → still denied. ReadOnly really means read-only.

**Clean up:**
```bash
aws iam remove-user-from-group --user-name lab-alice --group-name lab-readonly
aws iam detach-group-policy --group-name lab-readonly --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-group --group-name lab-readonly
```
*(Keep `lab-alice` — used in Lab 2.)*

---

## Lab 2 — Reading and Writing a Custom JSON Policy (PARC)

**Goal:** author a policy from scratch using the PARC framework and attach it as **Customer Managed**.

1. Create `custom-s3-policy.json`:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": ["s3:GetObject", "s3:PutObject"],
         "Resource": "arn:aws:s3:::lab-alice-bucket-*/*"
       }
     ]
   }
   ```
2. Create the bucket and the policy:
   ```bash
   aws s3 mb s3://lab-alice-bucket-$RANDOM
   aws iam create-policy --policy-name Lab-CustomS3-Alice --policy-document file://custom-s3-policy.json
   ```
3. Attach it directly to `lab-alice` (inline vs managed — do managed here):
   ```bash
   aws iam attach-user-policy --user-name lab-alice \
     --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/Lab-CustomS3-Alice
   ```
4. Test: `lab-alice` can now `PutObject`/`GetObject` on `lab-alice-bucket-*` but nothing else.
5. **Exercise:** add a `Deny` statement blocking `s3:DeleteObject` explicitly, and prove that Deny wins even with `AmazonS3FullAccess` also attached (attach it temporarily to see explicit-Deny override explicit-Allow).

**Clean up:** detach and delete the policy, empty and delete the bucket.

---

## Lab 3 — IAM Role for EC2 (Service Role + Instance Profile)

**Goal:** give an EC2 instance S3 access with **zero hardcoded credentials**.

1. Create the trust policy and role (see `commands-cheatsheet.md` §4).
2. Create the instance profile and attach the role.
3. Launch a `t3.micro` instance with `--iam-instance-profile Name=EC2-S3-ReadOnly-Profile`.
4. SSH in and run:
   ```bash
   aws s3 ls   # works — no keys configured anywhere on this box!
   curl http://169.254.169.254/latest/meta-data/iam/security-credentials/EC2-S3-ReadOnly-Role
   ```
5. Notice the temporary credentials returned by the metadata service — this is STS working invisibly behind the Instance Profile.

**Clean up:** terminate the instance, remove role from instance profile, delete instance profile, detach policy, delete role.

---

## Lab 4 — Cross-Account Role Assumption

**Goal:** simulate the Account-A → Account-B pattern using two AWS accounts (or two IAM users representing them if you only have one account).

1. In "Account B", create `CrossAccount-DevAccess` role with a trust policy naming Account A's user ARN (see cheatsheet §6).
2. Attach `AmazonS3ReadOnlyAccess` to that role.
3. From Account A's CLI profile, run:
   ```bash
   aws sts assume-role \
     --role-arn arn:aws:iam::<ACCOUNT_B_ID>:role/CrossAccount-DevAccess \
     --role-session-name cross-lab
   ```
4. Export the returned temporary credentials and run `aws s3 ls` — you're now acting inside Account B.
5. **Exercise:** add `"Condition": {"Bool": {"aws:MultiFactorAuthPresent": "true"}}` to the trust policy and confirm the assume-role now fails without an MFA-authenticated session.

**Clean up:** delete the role in Account B.

---

## Lab 5 — Resource-Based Policy on S3

**Goal:** grant access via the *resource* instead of the identity.

1. Create a bucket `shared-lab-bucket-<random>`.
2. Apply the bucket policy from cheatsheet §7, naming a specific external principal.
3. From that external account/user, run `aws s3 ls s3://shared-lab-bucket-<random>` — it works with **zero** identity-based policy granting S3 access on that side.
4. **Exercise:** flip the bucket policy to `"Effect": "Deny"` for a specific principal while that same principal has `AdministratorAccess` attached as an identity policy. Confirm Deny still wins.

**Clean up:** delete the bucket policy, then the bucket.

---

## Lab 6 — Permissions Boundaries

**Goal:** prove a boundary caps effective permissions regardless of what's directly attached.

1. Create a boundary policy allowing only `s3:*` actions (`s3-only-boundary.json`).
2. Create a user with that boundary:
   ```bash
   aws iam create-user --user-name lab-contractor --permissions-boundary arn:aws:iam::<ACCOUNT_ID>:policy/S3OnlyBoundary
   ```
3. Attach `AdministratorAccess` directly to `lab-contractor` (yes, really).
4. Test as `lab-contractor`: `aws ec2 describe-instances` → **still denied**, because the boundary caps it, even though the identity policy says "Admin."
5. Test `aws s3 ls` → succeeds, since S3 is inside the boundary.

**Clean up:** detach policies, delete the user, delete the boundary policy.

---

## Lab 7 — ABAC with Tags

**Goal:** one policy, many users, zero policy edits when onboarding new resources.

1. Tag `lab-alice`: `aws iam tag-user --user-name lab-alice --tags Key=team,Value=Alpha`
2. Launch two EC2 instances, tag one `team=Alpha` and the other `team=Beta`.
3. Attach the ABAC policy from cheatsheet §9 to `lab-alice`.
4. As `lab-alice`, try `ec2:StopInstances` on both instances:
   - Alpha instance → ✅ allowed (tags match)
   - Beta instance → 🚫 denied (tags don't match)
5. **Exercise:** re-tag the Beta instance to `team=Alpha` with zero policy changes and watch access appear automatically.

**Clean up:** terminate both instances, remove the ABAC policy.

---

## Lab 8 — `iam:PassRole` and the Privilege-Escalation Risk

**Goal:** feel *why* PassRole restrictions matter.

1. Create `lab-developer` user with only `ec2:RunInstances`, `ec2:*` EC2 actions, and **no** `iam:PassRole` at all.
2. As `lab-developer`, try to launch an EC2 instance with `--iam-instance-profile Name=EC2-S3-ReadOnly-Profile` → **fails** with `AccessDenied` naming `iam:PassRole`.
3. Attach the scoped `ScopedPassRole` policy from cheatsheet §10 (naming only that specific role ARN).
4. Retry — now it succeeds, but *only* for that one pre-approved role. Try passing a different, more powerful role → still denied.

**Clean up:** terminate the instance, remove the PassRole policy, delete `lab-developer`.

---

## Lab 9 — SCP Guardrail (requires AWS Organizations)

**Goal:** see an org-wide guardrail beat even an `AdministratorAccess` identity policy.

> Requires a management account with AWS Organizations enabled. If you don't have a multi-account setup, read through this lab conceptually — it's still worth understanding the mechanism.

1. Enable SCPs on your Organization root (cheatsheet §11).
2. Create the `RegionLock` SCP restricting activity to `us-east-1`/`ap-south-1`.
3. Attach it to a member account or OU.
4. Log into that member account as an Admin user and try to launch an EC2 instance in `eu-west-1` → **denied**, even with full `AdministratorAccess`.
5. Launch the same instance in `us-east-1` → succeeds.

**Clean up:** detach and delete the SCP.

---

## Lab 10 — IAM Access Analyzer

**Goal:** find external-sharing risk and unused permissions.

1. Create an account analyzer: `aws accessanalyzer create-analyzer --analyzer-name lab-analyzer --type ACCOUNT`
2. Re-create the Lab 5 bucket policy (external sharing) and wait a few minutes.
3. List findings — Access Analyzer should flag the bucket as **publicly/externally shared**.
4. Create an unused-access analyzer and, after a day or two of activity, review which permissions on `lab-alice`'s policies have never actually been used — candidates for removal.

**Clean up:** archive the finding once resolved, delete the analyzers if no longer needed.

---

## Lab 11 — IAM Policy Simulator (Test Before You Deploy)

**Goal:** validate access decisions without touching a live resource.

1. Simulate whether `lab-alice` can `s3:DeleteObject` on a bucket she only has `GetObject`/`PutObject` on:
   ```bash
   aws iam simulate-principal-policy \
     --policy-source-arn arn:aws:iam::<ACCOUNT_ID>:user/lab-alice \
     --action-names s3:DeleteObject \
     --resource-arns arn:aws:s3:::lab-alice-bucket-*/*
   ```
   → Result: `implicitDeny`.
2. Simulate a **hypothetical** policy you haven't attached yet using `simulate-custom-policy` to sanity-check it before `create-policy`.

---

## Lab 12 — Decode a Real `AccessDenied` Error

**Goal:** practice the troubleshooting workflow end to end.

1. As `lab-contractor` from Lab 6, attempt `aws ec2 describe-instances`.
2. Read the full error message returned — note it references the specific boundary or policy ARN that blocked you.
3. Cross-reference against [`troubleshooting.md`](./troubleshooting.md) to identify which of the 5 evaluation layers (SCP / resource policy / boundary / session policy / identity policy) is responsible.
4. Use `aws iam simulate-principal-policy` to confirm your diagnosis before making any policy change.

---

## Wrap-up Checklist

- [ ] Ran Labs 1–12 in a sandbox account
- [ ] Deleted every IAM user/role/policy created for labs
- [ ] Terminated every EC2 instance
- [ ] Reviewed CloudTrail once to see the `AssumeRole` and `AccessDenied` events these labs generated

➡️ When something doesn't behave as expected, go to [`troubleshooting.md`](./troubleshooting.md).
