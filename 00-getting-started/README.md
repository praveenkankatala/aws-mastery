# Getting Started

> **Status:** ✅ Complete
> **Read this before touching any other module.** Everything else in this repo assumes
> the account, CLI, and safety nets described here are already in place.

---

## Table of Contents

1. [AWS Account Setup](#1-aws-account-setup)
2. [Root Account Security](#2-root-account-security)
3. [Creating Your First IAM Administrator User](#3-creating-your-first-iam-administrator-user)
4. [AWS CLI Installation & Configuration](#4-aws-cli-installation--configuration)
5. [Multi-Profile CLI Setup (for multiple accounts)](#5-multi-profile-cli-setup-for-multiple-accounts)
6. [Billing Alerts & Cost Controls](#6-billing-alerts--cost-controls)
7. [Verifying Everything Works](#7-verifying-everything-works)
8. [Getting Started Checklist](#8-getting-started-checklist)
9. [Frequently Missed Details](#9-frequently-missed-details)

---

## 1. AWS Account Setup

1. Go to [aws.amazon.com](https://aws.amazon.com) and choose **Create an AWS Account**.
2. Use a **dedicated email address** for the root user — not your personal daily email.
   Recommended pattern: `aws-root+<companyname>@yourdomain.com` (using a `+` alias if your
   provider supports it, so the account is uniquely addressable but still lands in a
   monitored inbox).
3. Choose an account name that's clear if you'll have multiple accounts later
   (e.g., `yourname-learning`, `yourcompany-prod`).
4. Enter a valid payment method. AWS requires this even for Free Tier usage — this is
   why Section 6 (Billing Alerts) is not optional.
5. Select the **Basic Support Plan** (free) to start — you can upgrade later if you need
   faster support response times.
6. Complete phone verification.

**Free Tier note:** AWS's Free Tier covers a limited quota of usage per service, per month,
typically for 12 months from account creation (some services have an "Always Free" tier
with no time limit — e.g., a small amount of Lambda invocations). Free Tier does **not**
mean zero risk of charges — exceeding the quota bills normally. This is exactly why
Section 6 exists.

---

## 2. Root Account Security

The root user has **unrestricted access to everything**, including the ability to close
the account and change billing. It must never be used for daily work.

1. **Enable MFA on the root user immediately** — before doing anything else:
   - Console → top-right account name → **Security credentials** → **Multi-factor authentication (MFA)** → Assign MFA device
   - Prefer a **hardware security key** (e.g., YubiKey) for production/company accounts;
     a virtual MFA app (Google Authenticator, Authy) is acceptable for personal learning
     accounts.
2. **Set a strong, unique password** for root, stored in a password manager — not memorized,
   not reused from any other account.
3. **Do not create access keys for the root user.** If root access keys already exist
   from account creation, delete them: Console → Security credentials → Access keys → Delete.
4. **Never log in as root for routine tasks.** After completing Section 3 below, root
   login should be reserved for a short list of tasks that genuinely require it:
   changing the account's support plan, closing the account, changing the root email/password,
   or restoring access if all IAM users are somehow locked out.

---

## 3. Creating Your First IAM Administrator User

Do this immediately after securing root — this becomes your actual daily-use identity.

### Console steps
1. Navigate to **IAM** → **Users** → **Create user**
2. Username: e.g., `admin-yourname`
3. Do **not** enable console access with an auto-generated password unless you specifically
   need console login — prefer setting this up with IAM Identity Center (SSO) for anything
   beyond solo learning use (see note below).
4. Attach the managed policy `AdministratorAccess` (fine for a personal learning account;
   for team/production accounts, use least-privilege custom policies instead — see
   `06-security-identity/iam/` once that module exists).
5. **Enable MFA on this user too** — same process as root, different device/method if
   possible.
6. Create an **access key** for this user only if you need CLI access (Section 4)
   — choose "Command Line Interface (CLI)" as the use case when prompted.

### CLI equivalent (run using root credentials one time, before you have any IAM user yet)
```bash
aws iam create-user --user-name admin-yourname

aws iam attach-user-policy \
  --user-name admin-yourname \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

aws iam create-access-key --user-name admin-yourname
```
> Save the `AccessKeyId` and `SecretAccessKey` from the output immediately — the secret
> is shown only once and cannot be retrieved again later (you'd need to create a new key).

### Note on IAM Identity Center (formerly AWS SSO)
For anything beyond a solo learning account, prefer **IAM Identity Center** over long-lived
IAM user access keys. It provides temporary, auto-rotating credentials and centralized user
management — especially important once you have AWS Organizations with multiple accounts
(see `01-foundations/01-aws-global-infrastructure.md`, Step 2). This getting-started guide
uses a plain IAM user for simplicity since it's the fastest path for a single learning account.

---

## 4. AWS CLI Installation & Configuration

### Install AWS CLI v2

**macOS:**
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

**Linux:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Windows:**
Download and run the installer from:
`https://awscli.amazonaws.com/AWSCLIV2.msi`

### Verify installation
```bash
aws --version
# Expected output resembles: aws-cli/2.x.x Python/3.x.x <os>/<arch>
```

### Configure the CLI
```bash
aws configure
```
You'll be prompted for:
```
AWS Access Key ID [None]: <paste your access key from Section 3>
AWS Secret Access Key [None]: <paste your secret key from Section 3>
Default region name [None]: us-east-1        # or your preferred default Region
Default output format [None]: json
```

This writes two files:
- `~/.aws/credentials` — stores the access key/secret (keep this file private, never commit it)
- `~/.aws/config` — stores the default Region and output format

### Confirm the CLI is authenticated correctly
```bash
aws sts get-caller-identity
```
Expected output:
```json
{
    "UserId": "AIDAEXAMPLE123456",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/admin-yourname"
}
```
If this returns your IAM user's ARN (not an error), the CLI is correctly configured.

---

## 5. Multi-Profile CLI Setup (for multiple accounts)

If you'll work across more than one AWS account (e.g., a personal learning account and a
work account), avoid overwriting your default profile — use named profiles instead.

```bash
aws configure --profile personal
aws configure --profile work
```

This produces a `~/.aws/credentials` file structured like:
```ini
[default]
aws_access_key_id = AKIA...
aws_secret_access_key = ...

[personal]
aws_access_key_id = AKIA...
aws_secret_access_key = ...

[work]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
```

Use a specific profile per command with `--profile`:
```bash
aws s3 ls --profile work
```

Or set it for an entire terminal session:
```bash
export AWS_PROFILE=personal
```

**Always verify which account/profile you're targeting before running destructive
commands:**
```bash
aws sts get-caller-identity --profile work
```

---

## 6. Billing Alerts & Cost Controls

This is the single most important section for a learning account — it's extremely easy
to accidentally leave a resource running and get an unexpected bill.

### Step 1 — Enable Billing Alerts (one-time root-level setting)
1. Log in as **root** (this specific setting requires root, or an IAM user with
   explicit billing permissions).
2. Go to **Billing and Cost Management** → **Billing Preferences**.
3. Check **Receive Billing Alerts**. Save.

### Step 2 — Create an AWS Budget with an alert threshold
```bash
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "monthly-safety-net",
    "BudgetLimit": {"Amount": "10", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[{
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "you@example.com"}]
  }]'
```
This sends an email once actual spend crosses 80% of a $10/month budget — adjust the
amount to whatever you're comfortable risking during learning.

**Console equivalent:** Billing and Cost Management → **Budgets** → **Create budget** →
Customize → set amount and email alert threshold.

### Step 3 — Set up a CloudWatch Billing Alarm (belt-and-suspenders backup)
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "billing-alarm-10-usd" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 21600 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=Currency,Value=USD \
  --evaluation-periods 1 \
  --alarm-actions <SNS-topic-ARN> \
  --region us-east-1
```
> Billing metrics are only available in `us-east-1`, regardless of which Region your
> resources actually run in.

### Step 4 — Daily habit: check Cost Explorer
Billing and Cost Management → **Cost Explorer** — check this at least weekly while
actively doing labs, so you catch anything unexpected quickly rather than at month-end.

### Step 5 — The single best habit: tag and clean up
- Tag every resource you create for learning with `Project: learning` or similar, so
  you can filter Cost Explorer by tag and quickly find/delete stragglers.
- **Every hands-on lab in this repo ends with a cleanup step** — always run it, even
  if you plan to "come back to this later." Leftover RDS instances, NAT Gateways, and
  unattached Elastic IPs are the most common sources of surprise charges.

---

## 7. Verifying Everything Works

Run this short sequence to confirm the full setup end-to-end:

```bash
# 1. Confirm CLI authentication
aws sts get-caller-identity

# 2. Confirm default region is set correctly
aws configure get region

# 3. Confirm you can list S3 buckets (read-only, safe, free)
aws s3 ls

# 4. Confirm MFA is enabled on your IAM user
aws iam list-mfa-devices --user-name admin-yourname

# 5. Confirm a budget exists
aws budgets describe-budgets --account-id 123456789012
```

If all five commands return sensible output (not `AccessDenied` or empty where content
is expected), your account is ready for `01-foundations/`.

---

## 8. Getting Started Checklist

- [ ] AWS account created with dedicated root email
- [ ] Root user MFA enabled
- [ ] Root access keys deleted (none should exist)
- [ ] IAM administrator user created, with MFA enabled
- [ ] IAM user access key generated (only if CLI access needed)
- [ ] AWS CLI v2 installed and `aws --version` confirms it
- [ ] `aws configure` completed, `aws sts get-caller-identity` returns the IAM user (not root)
- [ ] Named profiles set up if using multiple accounts
- [ ] Billing alerts preference enabled in account settings
- [ ] At least one AWS Budget created with an email notification threshold
- [ ] CloudWatch billing alarm created as a backup (in `us-east-1`)
- [ ] Bookmarked Cost Explorer for weekly checks

---

## 9. Frequently Missed Details

- **Billing alerts must be enabled at the account level before Budgets/CloudWatch billing
  alarms will work at all** — this one-time toggle in Billing Preferences is easy to skip
  and silently breaks the rest of Section 6 if missed.
- **Billing metrics only exist in `us-east-1`** — a CloudWatch billing alarm created in
  any other Region simply won't receive data, even though the resources it should be
  monitoring are running elsewhere.
- **Free Tier limits are per-account, not per-Region** — running the same "free" resource
  in two Regions simultaneously can still push you over the monthly Free Tier quota.
- **Deleting an IAM user's access key does not delete the user** — and vice versa; these
  are managed as separate actions and both need cleanup if you're decommissioning an
  identity.
- **`aws configure` overwrites the `[default]` profile silently** — if you're setting up
  a second account, always use `--profile <name>`, or you'll lose access to the first
  account's stored credentials in the default profile.
- **A Budget's email alert is not real-time** — AWS Budgets typically refresh cost data
  a few times per day, not instantly; don't rely on it to catch a runaway resource within
  minutes. The CloudWatch billing alarm has similar granularity limitations. For genuinely
  fast-reacting cost control, combine budgets with resource-level safeguards (e.g.,
  auto-scaling group max-size limits, S3 lifecycle policies) rather than alerts alone.

---
Zones, the Shared Responsibility Model, and cloud delivery models, all of which assume
the account/CLI setup from this guide is already complete.
