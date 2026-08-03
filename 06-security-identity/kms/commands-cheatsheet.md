# AWS KMS — Commands Cheatsheet

> Every command here assumes AWS CLI v2 and that you've already run `aws configure`. Replace `<placeholders>` with real values. Add `| jq` to any command for pretty-printed JSON if you have `jq` installed.

**Related docs:** [`README.md`](./README.md) · [`hands-on-labs.md`](./hands-on-labs.md) · [`troubleshooting.md`](./troubleshooting.md)

---

## Table of Contents
1. [Key Lifecycle](#1-key-lifecycle)
2. [Aliases](#2-aliases)
3. [Key Policies](#3-key-policies)
4. [Tags](#4-tags)
5. [Encrypting & Decrypting](#5-encrypting--decrypting)
6. [Envelope Encryption (Data Keys)](#6-envelope-encryption-data-keys)
7. [Asymmetric Keys — Encrypt/Decrypt](#7-asymmetric-keys--encryptdecrypt)
8. [Signing & Verifying (Asymmetric / HMAC)](#8-signing--verifying-asymmetric--hmac)
9. [Key Rotation](#9-key-rotation)
10. [Grants](#10-grants)
11. [Multi-Region Keys](#11-multi-region-keys)
12. [Importing Key Material (BYOK)](#12-importing-key-material-byok)
13. [Custom Key Store (CloudHSM-backed)](#13-custom-key-store-cloudhsm-backed)
14. [Auditing & Discovery](#14-auditing--discovery)
15. [File-Based Encryption Workflow (End-to-End)](#15-file-based-encryption-workflow-end-to-end)

---

## 1. Key Lifecycle

```bash
# Create a standard symmetric encryption key (most common)
aws kms create-key \
  --description "My app encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT

# Create an asymmetric key (RSA, for encrypt/decrypt)
aws kms create-key \
  --description "Asymmetric encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec RSA_2048

# Create an asymmetric key (for signing)
aws kms create-key \
  --description "Signing key" \
  --key-usage SIGN_VERIFY \
  --key-spec ECC_NIST_P256

# Create an HMAC key
aws kms create-key \
  --description "HMAC key" \
  --key-usage GENERATE_VERIFY_MAC \
  --key-spec HMAC_256

# List all keys in the account/region
aws kms list-keys

# Describe a specific key (full metadata: state, spec, origin, etc.)
aws kms describe-key --key-id <key-id-or-arn-or-alias>

# Disable a key (stops all crypto operations, but key still exists)
aws kms disable-key --key-id <key-id>

# Re-enable a disabled key
aws kms enable-key --key-id <key-id>

# Schedule deletion (mandatory 7–30 day waiting period, default 30)
aws kms schedule-key-deletion \
  --key-id <key-id> \
  --pending-window-in-days 7

# Cancel a pending deletion (if within the waiting window)
aws kms cancel-key-deletion --key-id <key-id>
```

---

## 2. Aliases

```bash
# Create an alias pointing to a key
aws kms create-alias \
  --alias-name alias/my-app-key \
  --target-key-id <key-id>

# List all aliases
aws kms list-aliases

# Point an existing alias at a different key (e.g., during key rotation strategy)
aws kms update-alias \
  --alias-name alias/my-app-key \
  --target-key-id <new-key-id>

# Delete an alias (does NOT delete the underlying key)
aws kms delete-alias --alias-name alias/my-app-key
```

---

## 3. Key Policies

```bash
# View the current key policy
aws kms get-key-policy \
  --key-id <key-id> \
  --policy-name default \
  --output text

# Replace the key policy entirely (be careful — this can lock you out)
aws kms put-key-policy \
  --key-id <key-id> \
  --policy-name default \
  --policy file://key-policy.json

# List the policy names attached to a key (there's only ever one: "default")
aws kms list-key-policies --key-id <key-id>
```

**Minimal safe key policy template** (`key-policy.json`) — delegates day-to-day control to IAM:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableIAMUserPermissions",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<account-id>:root" },
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
```

---

## 4. Tags

```bash
# Tag a key
aws kms tag-resource \
  --key-id <key-id> \
  --tags TagKey=Environment,TagValue=Production

# List tags on a key
aws kms list-resource-tags --key-id <key-id>

# Remove a tag
aws kms untag-resource \
  --key-id <key-id> \
  --tag-keys Environment
```

---

## 5. Encrypting & Decrypting

> Reminder: direct `encrypt`/`decrypt` is capped at **4 KB** — use this for small secrets/config values, not bulk data.

```bash
# Encrypt a small piece of plaintext
aws kms encrypt \
  --key-id alias/my-app-key \
  --plaintext "my secret value" \
  --output text \
  --query CiphertextBlob | base64 --decode > secret.encrypted

# Decrypt it back
aws kms decrypt \
  --ciphertext-blob fileb://secret.encrypted \
  --output text \
  --query Plaintext | base64 --decode

# Encrypt with encryption context (recommended — binds ciphertext to metadata)
aws kms encrypt \
  --key-id alias/my-app-key \
  --plaintext "my secret value" \
  --encryption-context Department=Finance,Environment=Prod \
  --output text --query CiphertextBlob | base64 --decode > secret.encrypted

# Decrypt — MUST provide the exact same encryption context, or it fails
aws kms decrypt \
  --ciphertext-blob fileb://secret.encrypted \
  --encryption-context Department=Finance,Environment=Prod \
  --output text --query Plaintext | base64 --decode

# Re-encrypt ciphertext under a different key, without exposing plaintext
aws kms re-encrypt \
  --ciphertext-blob fileb://secret.encrypted \
  --destination-key-id alias/other-key \
  --output text --query CiphertextBlob | base64 --decode > secret-rewrapped.encrypted
```

---

## 6. Envelope Encryption (Data Keys)

> This is the pattern for anything larger than 4 KB — i.e., basically all real application data.

```bash
# Generate a data key: returns BOTH plaintext and encrypted versions
aws kms generate-data-key \
  --key-id alias/my-app-key \
  --key-spec AES_256

# Generate a data key WITHOUT the plaintext copy (useful when you only need
# to store the encrypted key now and decrypt later, e.g., pre-provisioning)
aws kms generate-data-key-without-plaintext \
  --key-id alias/my-app-key \
  --key-spec AES_256

# Decrypt a stored encrypted data key back into a usable plaintext key
aws kms decrypt \
  --ciphertext-blob fileb://encrypted-data-key.bin \
  --output text --query Plaintext | base64 --decode > data-key-plaintext.bin

# Generate a random data key pair (for asymmetric envelope scenarios)
aws kms generate-data-key-pair \
  --key-id alias/my-app-key \
  --key-pair-spec RSA_2048

# Generate cryptographically secure random bytes (via KMS's HSMs)
aws kms generate-random --number-of-bytes 32
```

**Full local file encryption workflow** — see [Section 15](#15-file-based-encryption-workflow-end-to-end).

---

## 7. Asymmetric Keys — Encrypt/Decrypt

```bash
# Get the public key (safe to share/export — this is the whole point of asymmetric keys)
aws kms get-public-key \
  --key-id alias/my-asymmetric-key \
  --output text --query PublicKey | base64 --decode > public_key.der

# Encrypt using the public key (can be done by ANYONE with the public key,
# even outside AWS, using standard crypto libraries)
aws kms encrypt \
  --key-id alias/my-asymmetric-key \
  --plaintext fileb://plaintext.txt \
  --encryption-algorithm RSAES_OAEP_SHA_256 \
  --output text --query CiphertextBlob | base64 --decode > ciphertext.bin

# Decrypt — only works if the caller is authorized on the PRIVATE key in KMS
aws kms decrypt \
  --key-id alias/my-asymmetric-key \
  --ciphertext-blob fileb://ciphertext.bin \
  --encryption-algorithm RSAES_OAEP_SHA_256 \
  --output text --query Plaintext | base64 --decode
```

---

## 8. Signing & Verifying (Asymmetric / HMAC)

```bash
# Sign a message digest with an asymmetric signing key
aws kms sign \
  --key-id alias/my-signing-key \
  --message fileb://message.txt \
  --message-type RAW \
  --signing-algorithm ECDSA_SHA_256 \
  --output text --query Signature | base64 --decode > signature.bin

# Verify the signature
aws kms verify \
  --key-id alias/my-signing-key \
  --message fileb://message.txt \
  --message-type RAW \
  --signature fileb://signature.bin \
  --signing-algorithm ECDSA_SHA_256

# Generate an HMAC tag
aws kms generate-mac \
  --key-id alias/my-hmac-key \
  --message fileb://message.txt \
  --mac-algorithm HMAC_SHA_256

# Verify an HMAC tag
aws kms verify-mac \
  --key-id alias/my-hmac-key \
  --message fileb://message.txt \
  --mac-algorithm HMAC_SHA_256 \
  --mac <mac-value-from-generate-mac>
```

---

## 9. Key Rotation

```bash
# Enable automatic rotation (symmetric AWS-generated keys only)
aws kms enable-key-rotation --key-id alias/my-app-key

# Set a custom rotation period (in days)
aws kms enable-key-rotation \
  --key-id alias/my-app-key \
  --rotation-period-in-days 180

# Check rotation status
aws kms get-key-rotation-status --key-id alias/my-app-key

# Disable rotation
aws kms disable-key-rotation --key-id alias/my-app-key

# List rotation history (see past rotation dates)
aws kms list-key-rotations --key-id alias/my-app-key
```

---

## 10. Grants

```bash
# Create a grant giving a role permission to use the key for specific operations
aws kms create-grant \
  --key-id alias/my-app-key \
  --grantee-principal arn:aws:iam::<account-id>:role/MyAppRole \
  --operations Decrypt GenerateDataKey

# List grants on a key
aws kms list-grants --key-id alias/my-app-key

# List grants made BY you (as a grantee, useful for delegation chains)
aws kms list-retirable-grants \
  --retiring-principal arn:aws:iam::<account-id>:role/MyAppRole

# Revoke a grant (as the key owner/admin)
aws kms revoke-grant \
  --key-id alias/my-app-key \
  --grant-id <grant-id>

# Retire a grant (as the grantee, giving up the permission voluntarily)
aws kms retire-grant \
  --grant-token <grant-token>
```

---

## 11. Multi-Region Keys

```bash
# Create a primary multi-region key
aws kms create-key \
  --multi-region \
  --description "Primary multi-region key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT

# Replicate it into another region
aws kms replicate-key \
  --key-id <primary-key-id> \
  --replica-region us-west-2 \
  --description "Replica in us-west-2"

# Update the primary region (promote a replica to primary)
aws kms update-primary-region \
  --key-id <key-id> \
  --primary-region us-west-2
```

---

## 12. Importing Key Material (BYOK)

```bash
# Step 1: Create a key shell with EXTERNAL origin
aws kms create-key \
  --origin EXTERNAL \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT

# Step 2: Get a public key + import token to wrap your key material with
aws kms get-parameters-for-import \
  --key-id <key-id> \
  --wrapping-algorithm RSAES_OAEP_SHA_256 \
  --wrapping-key-spec RSA_2048

# Step 3 (outside AWS): encrypt your key material with the returned public key
# using OpenSSL or similar — produces a wrapped key blob

# Step 4: Import the wrapped key material
aws kms import-key-material \
  --key-id <key-id> \
  --encrypted-key-material fileb://WrappedKeyMaterial.bin \
  --import-token fileb://ImportToken.bin \
  --expiration-model KEY_MATERIAL_DOES_NOT_EXPIRE

# Delete imported key material (instantly makes the key unusable — "crypto-shredding")
aws kms delete-imported-key-material --key-id <key-id>
```

---

## 13. Custom Key Store (CloudHSM-backed)

```bash
# Connect a KMS custom key store to an existing CloudHSM cluster
aws kms create-custom-key-store \
  --custom-key-store-name my-hsm-store \
  --cloud-hsm-cluster-id <cluster-id> \
  --trust-anchor-certificate file://customerCA.crt \
  --key-store-password <password>

# Connect it (activate)
aws kms connect-custom-key-store --custom-key-store-id <store-id>

# Check status
aws kms describe-custom-key-stores --custom-key-store-id <store-id>

# Create a key inside the custom key store
aws kms create-key \
  --custom-key-store-id <store-id> \
  --origin AWS_CLOUDHSM \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT
```

---

## 14. Auditing & Discovery

```bash
# Confirm your identity/account context (useful before touching keys)
aws sts get-caller-identity

# Find CloudTrail events for a specific key (last N events matching KMS)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com \
  --max-results 20

# Find who called Decrypt recently
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=Decrypt \
  --max-results 20

# Simulate whether a principal CAN perform an action (great for debugging access denials)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<account-id>:role/MyAppRole \
  --action-names kms:Decrypt \
  --resource-arns arn:aws:kms:<region>:<account-id>:key/<key-id>
```

---

## 15. File-Based Encryption Workflow (End-to-End)

A realistic, copy-pasteable local workflow using `openssl` for the actual bulk encryption (this is what "envelope encryption" looks like outside of a managed SDK):

```bash
# 1. Generate a data key (get both plaintext + encrypted versions)
aws kms generate-data-key \
  --key-id alias/my-app-key \
  --key-spec AES_256 \
  --output json > datakey.json

# 2. Extract and decode the plaintext and encrypted portions
jq -r .Plaintext datakey.json | base64 --decode > plaintext-key.bin
jq -r .CiphertextBlob datakey.json | base64 --decode > encrypted-key.bin

# 3. Encrypt your actual file locally with the plaintext data key (AES-256-CBC example)
openssl enc -aes-256-cbc -salt -in myfile.txt -out myfile.enc \
  -pass file:plaintext-key.bin

# 4. IMPORTANT: securely delete the plaintext key — never store it
shred -u plaintext-key.bin      # Linux
# or: rm -P plaintext-key.bin   # macOS

# 5. Store myfile.enc + encrypted-key.bin together (e.g., upload both to S3)

# --- Later, to decrypt ---

# 6. Send the encrypted data key back to KMS to get the plaintext key again
aws kms decrypt \
  --ciphertext-blob fileb://encrypted-key.bin \
  --output text --query Plaintext | base64 --decode > plaintext-key.bin

# 7. Decrypt the file locally
openssl enc -d -aes-256-cbc -in myfile.enc -out myfile-decrypted.txt \
  -pass file:plaintext-key.bin

# 8. Clean up the plaintext key again
shred -u plaintext-key.bin
```

---

*Related docs: [`README.md`](./README.md) · [`hands-on-labs.md`](./hands-on-labs.md) · [`troubleshooting.md`](./troubleshooting.md)*
