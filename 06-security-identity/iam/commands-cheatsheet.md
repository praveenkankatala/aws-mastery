# AWS IAM — Commands Cheat Sheet

All commands assume the AWS CLI v2 is installed and configured (`aws configure`). Replace `123456789012`, ARNs, and names with your own values. See [`README.md`](./README.md) for the concepts behind each command and [`hands-on-labs.md`](./hands-on-labs.md) to use these in context.

---

## Table of Contents

- [1. Users](#1-users)
- [2. Groups](#2-groups)
- [3. Policies](#3-policies)
- [4. Roles & Trust Policies](#4-roles--trust-policies)
- [5. Assuming Roles (STS)](#5-assuming-roles-sts)
- [6. Cross-Account Access](#6-cross-account-access)
- [7. Resource-Based Policies (S3 example)](#7-resource-based-policies-s3-example)
- [8. Permissions Boundaries](#8-permissions-boundaries)
- [9. Tags & ABAC](#9-tags--abac)
- [10. iam:PassRole in Practice](#10-iampassrole-in-practice)
- [11. Organizations: SCPs & RCPs](#11-organizations-scps--rcps)
- [12. IAM Access Analyzer](#12-iam-access-analyzer)
- [13. IAM Policy Simulator](#13-iam-policy-simulator)
- [14. MFA](#14-mfa)
- [15. IAM Identity Center (SSO)](#15-iam-identity-center-sso)
- [16. IAM Roles Anywhere](#16-iam-roles-anywhere)
- [17. Auditing / Debugging Requests](#17-auditing--debugging-requests)

---

## 1. Users

```bash
# Create a user
aws iam create-user --user-name alice

# Give console access (creates a login profile)
aws iam create-login-profile --user-name alice \
  --password 'TempPassw0rd!23' --password-reset-required

# Create programmatic access keys
aws iam create-access-key --user-name alice

# List all users
aws iam list-users

# Attach a managed policy directly to a user (avoid in production — prefer groups/roles)
aws iam attach-user-policy --user-name alice \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# Delete a user (must remove keys/policies/login profile first)
aws iam delete-login-profile --user-name alice
aws iam delete-access-key --user-name alice --access-key-id AKIA...
aws iam detach-user-policy --user-name alice --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
aws iam delete-user --user-name alice
```

## 2. Groups

```bash
# Create a group
aws iam create-group --group-name Developers

# Attach a managed policy to the group
aws iam attach-group-policy --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Add a user to the group
aws iam add-user-to-group --user-name alice --group-name Developers

# List users in a group
aws iam get-group --group-name Developers

# Remove a user from a group
aws iam remove-user-from-group --user-name alice --group-name Developers
```

## 3. Policies

```bash
# Create a customer-managed policy from a local JSON file
aws iam create-policy --policy-name S3AppBucketReadOnly \
  --policy-document file://s3-readonly-policy.json

# List AWS-managed policies
aws iam list-policies --scope AWS --max-items 20

# List your customer-managed policies
aws iam list-policies --scope Local

# Get the default version (actual JSON) of a policy
aws iam get-policy --policy-arn arn:aws:iam::123456789012:policy/S3AppBucketReadOnly
aws iam get-policy-version --policy-arn arn:aws:iam::123456789012:policy/S3AppBucketReadOnly \
  --version-id v1

# Create a new version of a policy (policies are versioned, max 5 versions kept)
aws iam create-policy-version \
  --policy-arn arn:aws:iam::123456789012:policy/S3AppBucketReadOnly \
  --policy-document file://s3-readonly-policy-v2.json --set-as-default

# Inline policy — embedded directly on a user/group/role (1:1, use sparingly)
aws iam put-user-policy --user-name alice --policy-name InlineS3Access \
  --policy-document file://inline-policy.json

aws iam list-user-policies --user-name alice
aws iam delete-user-policy --user-name alice --policy-name InlineS3Access
```

## 4. Roles & Trust Policies

Every role needs a **trust policy** (who can assume it) and one or more **permission policies** (what it can do once assumed).

```bash
# trust-policy-ec2.json — lets the EC2 service assume this role
cat > trust-policy-ec2.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role --role-name EC2-S3-ReadOnly-Role \
  --assume-role-policy-document file://trust-policy-ec2.json

# Attach a permission policy to the role
aws iam attach-role-policy --role-name EC2-S3-ReadOnly-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create an Instance Profile and add the role to it (EC2 needs this wrapper)
aws iam create-instance-profile --instance-profile-name EC2-S3-ReadOnly-Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-ReadOnly-Profile \
  --role-name EC2-S3-ReadOnly-Role

# Attach the instance profile to a running/new EC2 instance
aws ec2 associate-iam-instance-profile \
  --instance-id i-0123456789abcdef0 \
  --iam-instance-profile Name=EC2-S3-ReadOnly-Profile

# Lambda execution role example (trust policy for lambda.amazonaws.com)
cat > trust-policy-lambda.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
aws iam create-role --role-name Lambda-Basic-Exec-Role \
  --assume-role-policy-document file://trust-policy-lambda.json
aws iam attach-role-policy --role-name Lambda-Basic-Exec-Role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

## 5. Assuming Roles (STS)

```bash
# Assume a role and get temporary credentials
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/EC2-S3-ReadOnly-Role \
  --role-session-name my-session

# Response gives AccessKeyId, SecretAccessKey, SessionToken (valid up to 1hr–12hr)
# Export them to use in the current shell:
export AWS_ACCESS_KEY_ID=ASIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

# Check who you currently are (great sanity check after assuming a role)
aws sts get-caller-identity
```

## 6. Cross-Account Access

```bash
# In Account B: trust policy that allows a specific user/role in Account A
cat > cross-account-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111111111111:user/alice" },
      "Action": "sts:AssumeRole",
      "Condition": { "Bool": { "aws:MultiFactorAuthPresent": "true" } }
    }
  ]
}
EOF

aws iam create-role --role-name CrossAccount-DevAccess \
  --assume-role-policy-document file://cross-account-trust.json

# From Account A, alice assumes into Account B:
aws sts assume-role \
  --role-arn arn:aws:iam::222222222222:role/CrossAccount-DevAccess \
  --role-session-name alice-cross-account-session
```

## 7. Resource-Based Policies (S3 example)

Unlike identity policies, resource policies **must** name a `Principal`.

```bash
cat > bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::222222222222:root" },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-shared-bucket",
        "arn:aws:s3:::my-shared-bucket/*"
      ]
    }
  ]
}
EOF

aws s3api put-bucket-policy --bucket my-shared-bucket --policy file://bucket-policy.json
aws s3api get-bucket-policy --bucket my-shared-bucket
aws s3api delete-bucket-policy --bucket my-shared-bucket
```

## 8. Permissions Boundaries

```bash
# Create the boundary policy (max permissions ceiling)
aws iam create-policy --policy-name S3OnlyBoundary \
  --policy-document file://s3-only-boundary.json

# Apply it when creating a user
aws iam create-user --user-name contractor-bob \
  --permissions-boundary arn:aws:iam::123456789012:policy/S3OnlyBoundary

# Apply it to an existing role
aws iam put-role-permissions-boundary --role-name SomeRole \
  --permissions-boundary arn:aws:iam::123456789012:policy/S3OnlyBoundary

# Remove a boundary
aws iam delete-role-permissions-boundary --role-name SomeRole
```

## 9. Tags & ABAC

```bash
# Tag an IAM user (used in ABAC conditions)
aws iam tag-user --user-name alice --tags Key=team,Value=Alpha

# Tag an EC2 instance the same way
aws ec2 create-tags --resources i-0123456789abcdef0 \
  --tags Key=team,Value=Alpha

# ABAC policy: allow stopping/starting only if tags match (put this in the policy JSON)
cat > abac-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:StopInstances", "ec2:StartInstances"],
      "Resource": "*",
      "Condition": {
        "StringEquals": { "aws:ResourceTag/team": "${aws:PrincipalTag/team}" }
      }
    }
  ]
}
EOF
aws iam create-policy --policy-name ABAC-TeamMatch --policy-document file://abac-policy.json

# Policy-variable example: each user only reaches their own S3 folder
# "Resource": "arn:aws:s3:::my-bucket/home/${aws:username}/*"
```

## 10. iam:PassRole in Practice

```bash
# The identity launching a resource needs this permission, scoped to the exact role:
cat > passrole-scoped.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::123456789012:role/EC2-S3-ReadOnly-Role",
      "Condition": {
        "StringEquals": { "iam:PassedToService": "ec2.amazonaws.com" }
      }
    }
  ]
}
EOF
aws iam create-policy --policy-name ScopedPassRole --policy-document file://passrole-scoped.json

# Launch an instance passing that role (fails without the PassRole grant above)
aws ec2 run-instances --image-id ami-0123456789abcdef0 --instance-type t3.micro \
  --iam-instance-profile Name=EC2-S3-ReadOnly-Profile
```

## 11. Organizations: SCPs & RCPs

*(Run from the AWS Organizations management account.)*

```bash
# Enable the SCP policy type on the Organization (one-time setup)
aws organizations enable-policy-type \
  --root-id r-abcd --policy-type SERVICE_CONTROL_POLICY

# Create an SCP restricting to one region
cat > region-lock-scp.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "NotAction": ["iam:*", "organizations:*", "sts:*", "support:*"],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": { "aws:RequestedRegion": ["us-east-1", "ap-south-1"] }
      }
    }
  ]
}
EOF
aws organizations create-policy --name RegionLock --type SERVICE_CONTROL_POLICY \
  --content file://region-lock-scp.json --description "Deny all regions except us-east-1/ap-south-1"

# Attach it to an OU or account
aws organizations attach-policy --policy-id p-examplescp --target-id ou-abcd-11111111

# Resource Control Policies (RCP) — same workflow, different policy type
aws organizations enable-policy-type \
  --root-id r-abcd --policy-type RESOURCE_CONTROL_POLICY

aws organizations create-policy --name DataPerimeter --type RESOURCE_CONTROL_POLICY \
  --content file://data-perimeter-rcp.json --description "Only org identities can touch our S3/KMS"

aws organizations attach-policy --policy-id p-examplercp --target-id r-abcd
```

## 12. IAM Access Analyzer

```bash
# Create an analyzer for your account (or ORGANIZATION type at the mgmt account)
aws accessanalyzer create-analyzer --analyzer-name account-analyzer --type ACCOUNT

# List findings (public/external sharing on resource policies)
aws accessanalyzer list-findings --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/account-analyzer

# Validate a policy document before attaching it (catches syntax + best-practice issues)
aws accessanalyzer validate-policy \
  --policy-document file://s3-readonly-policy.json \
  --policy-type IDENTITY_POLICY

# Unused access analyzer (finds permissions granted but never used)
aws accessanalyzer create-analyzer --analyzer-name unused-access-analyzer \
  --type ACCOUNT_UNUSED_ACCESS
```

## 13. IAM Policy Simulator

```bash
# Simulate whether a principal can perform an action on a resource — no real API call made
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/alice \
  --action-names s3:DeleteObject \
  --resource-arns arn:aws:s3:::my-awesome-bucket/file.txt

# Simulate a hypothetical policy that isn't even attached yet
aws iam simulate-custom-policy \
  --policy-input-list file://hypothetical-policy.json \
  --action-names s3:PutObject \
  --resource-arns arn:aws:s3:::my-awesome-bucket/*
```

## 14. MFA

```bash
# Create a virtual MFA device
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name alice-mfa \
  --outfile ./QRCode.png --bootstrap-method QRCodePNG

# Enable it for a user (need two consecutive codes from the authenticator app)
aws iam enable-mfa-device --user-name alice \
  --serial-number arn:aws:iam::123456789012:mfa/alice-mfa \
  --authentication-code1 123456 --authentication-code2 789012

# Require MFA in a policy condition
# "Condition": { "Bool": { "aws:MultiFactorAuthPresent": "true" } }
```

## 15. IAM Identity Center (SSO)

*(Most setup is console/SCIM-driven; the CLI mainly reads state.)*

```bash
# List the Identity Center instance for your Organization
aws sso-admin list-instances

# List permission sets (the SSO equivalent of an IAM policy bundle)
aws sso-admin list-permission-sets --instance-arn arn:aws:sso:::instance/ssoins-example

# Provision (assign) a permission set to an account for a group/user
aws sso-admin create-account-assignment \
  --instance-arn arn:aws:sso:::instance/ssoins-example \
  --target-id 123456789012 --target-type AWS_ACCOUNT \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-example/ps-example \
  --principal-type GROUP --principal-id <group-id>
```

## 16. IAM Roles Anywhere

```bash
# Create a Trust Anchor pointing at your own Certificate Authority
aws rolesanywhere create-trust-anchor \
  --name on-prem-ca \
  --source '{"sourceType":"CERTIFICATE_BUNDLE","sourceData":{"x509CertificateData":"file://ca-cert.pem"}}' \
  --enabled

# Create a Profile that maps to an IAM role
aws rolesanywhere create-profile \
  --name on-prem-servers-profile \
  --role-arns arn:aws:iam::123456789012:role/OnPremAppRole \
  --enabled

# The on-prem server then uses the AWS "credential helper" binary with its cert to
# exchange for temporary STS credentials — see AWS docs for the helper install steps.
```

## 17. Auditing / Debugging Requests

```bash
# Who am I right now (identity + account)?
aws sts get-caller-identity

# Find the most recent AccessDenied events in CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AccessDenied \
  --max-results 10

# Generate a least-privilege policy from actual CloudTrail activity for a role
aws iam generate-service-last-accessed-details --arn arn:aws:iam::123456789012:role/SomeRole
aws iam get-service-last-accessed-details --job-id <job-id-from-above>
```

---

➡️ Next: put these into practice in [`hands-on-labs.md`](./hands-on-labs.md).
