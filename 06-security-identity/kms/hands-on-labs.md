# AWS KMS — Hands-On Labs

> Seven guided labs, from your very first key to a realistic encrypted application pattern. Each lab tells you *what* to do, *why* you're doing it, and what to check to confirm it worked. Clean-up steps are included at the end of each lab — KMS keys cost ~$1/month each, so tidy up after yourself.

**Related docs:** [`README.md`](./README.md) · [`commands-cheatsheet.md`](./commands-cheatsheet.md) · [`troubleshooting.md`](./troubleshooting.md)

---

## Before You Start

```bash
aws --version                 # confirm AWS CLI v2
aws sts get-caller-identity   # confirm your credentials & account
```

Set a couple of variables you'll reuse throughout (adjust the region to your own):
```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account: $ACCOUNT_ID | Region: $AWS_REGION"
```

---

## Lab 1 — Create Your First Key and Encrypt Something

**Goal:** understand the absolute basics — create a key, encrypt a small value, decrypt it back.

### Step 1: Create a symmetric key
```bash
aws kms create-key \
  --description "Lab 1 - my first KMS key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --tags TagKey=Lab,TagValue=Lab1
```
Copy the `KeyId` from the output — you'll need it below. Notice the response also includes `KeyState: Enabled`, `Origin: AWS_KMS`, and `KeyManager: CUSTOMER` — confirming this is a customer managed key with AWS-generated material.

### Step 2: Give it an alias (so you never have to type that UUID again)
```bash
aws kms create-alias \
  --alias-name alias/lab1-key \
  --target-key-id <paste-key-id-here>
```

### Step 3: Encrypt a small secret
```bash
aws kms encrypt \
  --key-id alias/lab1-key \
  --plaintext "hello from KMS" \
  --output text --query CiphertextBlob | base64 --decode > lab1-secret.encrypted
```

### Step 4: Decrypt it back
```bash
aws kms decrypt \
  --ciphertext-blob fileb://lab1-secret.encrypted \
  --output text --query Plaintext | base64 --decode
```
You should see `hello from KMS` printed back.

### ✅ Checkpoint
- You created a customer managed key
- You confirmed encrypt → decrypt round-trips correctly
- You understand this only works for payloads under 4 KB

### Clean up
```bash
aws kms delete-alias --alias-name alias/lab1-key
aws kms schedule-key-deletion --key-id <key-id> --pending-window-in-days 7
rm lab1-secret.encrypted
```

---

## Lab 2 — Envelope Encryption From Scratch

**Goal:** understand *why* real applications don't call `encrypt` directly on bulk data — build the envelope encryption pattern by hand.

### Step 1: Create a fresh key for this lab
```bash
aws kms create-key --description "Lab 2 - envelope encryption" \
  --key-usage ENCRYPT_DECRYPT --key-spec SYMMETRIC_DEFAULT \
  --tags TagKey=Lab,TagValue=Lab2 --query KeyMetadata.KeyId --output text
aws kms create-alias --alias-name alias/lab2-key --target-key-id <key-id>
```

### Step 2: Create a "large" file to encrypt (bigger than the 4 KB direct limit)
```bash
head -c 10000 /dev/urandom > big-file.bin   # ~10 KB — too big for direct encrypt
```
Try encrypting it directly to see the failure for yourself:
```bash
aws kms encrypt --key-id alias/lab2-key --plaintext fileb://big-file.bin
```
You should get a `ValidationException` about the plaintext exceeding the maximum size — this is expected and is exactly the problem envelope encryption solves.

### Step 3: Generate a data key instead
```bash
aws kms generate-data-key --key-id alias/lab2-key --key-spec AES_256 --output json > datakey.json
jq -r .Plaintext datakey.json | base64 --decode > plaintext-key.bin
jq -r .CiphertextBlob datakey.json | base64 --decode > encrypted-key.bin
```

### Step 4: Encrypt the big file locally with the plaintext data key
```bash
openssl enc -aes-256-cbc -salt -in big-file.bin -out big-file.enc -pass file:plaintext-key.bin
```

### Step 5: Discard the plaintext key — this is the step people skip and shouldn't
```bash
shred -u plaintext-key.bin
```

### Step 6: Simulate "later" — decrypt using only what you stored
```bash
aws kms decrypt --ciphertext-blob fileb://encrypted-key.bin \
  --output text --query Plaintext | base64 --decode > plaintext-key.bin

openssl enc -d -aes-256-cbc -in big-file.enc -out big-file-decrypted.bin -pass file:plaintext-key.bin

diff big-file.bin big-file-decrypted.bin && echo "✅ Files match — round trip successful"
```

### ✅ Checkpoint
- You proved the 4 KB direct-encrypt limit exists
- You built the full generate → encrypt-locally → discard → store → decrypt-later cycle by hand
- You understand why `encrypted-key.bin` is safe to store next to the ciphertext (it's useless without KMS authorization)

### Clean up
```bash
aws kms delete-alias --alias-name alias/lab2-key
aws kms schedule-key-deletion --key-id <key-id> --pending-window-in-days 7
rm -f big-file.bin big-file.enc big-file-decrypted.bin plaintext-key.bin encrypted-key.bin datakey.json
```

---

## Lab 3 — Key Policies & IAM: Understanding Who Can Do What

**Goal:** deliberately break and fix access, so the key-policy-vs-IAM relationship actually clicks.

### Step 1: Create a key and inspect its default policy
```bash
aws kms create-key --description "Lab 3 - policies" --query KeyMetadata.KeyId --output text
aws kms get-key-policy --key-id <key-id> --policy-name default --output text | jq .
```
Notice the `"Enable IAM User Permissions"` statement granting the account root `kms:*`. This is what lets IAM take over from here.

### Step 2: Create a test IAM role with NO kms permissions yet
```bash
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::ACCOUNT_ID:root" },
    "Action": "sts:AssumeRole"
  }]
}
EOF
sed -i "s/ACCOUNT_ID/$ACCOUNT_ID/" trust-policy.json

aws iam create-role --role-name Lab3TestRole --assume-role-policy-document file://trust-policy.json
```

### Step 3: Try to use the key as that role — confirm it's denied
Attach only an IAM policy allowing `kms:Decrypt` scoped to this key, but keep it unattached for now — first prove that *without* it, access fails when simulated:
```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::$ACCOUNT_ID:role/Lab3TestRole \
  --action-names kms:Decrypt \
  --resource-arns arn:aws:kms:$AWS_REGION:$ACCOUNT_ID:key/<key-id>
```
Result: `implicitDeny` — the role has no IAM policy granting this yet.

### Step 4: Attach an IAM policy and re-check
```bash
cat > kms-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
    "Resource": "arn:aws:kms:REGION:ACCOUNT_ID:key/KEY_ID"
  }]
}
EOF
sed -i "s/REGION/$AWS_REGION/;s/ACCOUNT_ID/$ACCOUNT_ID/;s/KEY_ID/<key-id>/" kms-policy.json

aws iam put-role-policy --role-name Lab3TestRole --policy-name KmsAccess --policy-document file://kms-policy.json

aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::$ACCOUNT_ID:role/Lab3TestRole \
  --action-names kms:Decrypt \
  --resource-arns arn:aws:kms:$AWS_REGION:$ACCOUNT_ID:key/<key-id>
```
Result should now be `allowed` — because **both** the key policy (via its IAM delegation statement) and the role's IAM policy agree.

### Step 5: Prove the key policy still wins — remove the IAM delegation statement
As an experiment (on a throwaway lab key only!), replace the key policy with one that does **not** delegate to IAM, and re-run the simulation — you'll see it now fails again regardless of the IAM policy, because the key policy no longer trusts IAM to decide.

### ✅ Checkpoint
- You saw an implicit deny with no IAM policy
- You saw an explicit allow once both layers agreed
- You proved the key policy is the ultimate gatekeeper, not IAM alone

### Clean up
```bash
aws iam delete-role-policy --role-name Lab3TestRole --policy-name KmsAccess
aws iam delete-role --role-name Lab3TestRole
aws kms schedule-key-deletion --key-id <key-id> --pending-window-in-days 7
rm -f trust-policy.json kms-policy.json
```

---

## Lab 4 — Grants: Temporary, Programmatic Access

**Goal:** use grants instead of editing key policies — the pattern AWS services use internally.

### Step 1: Create a key and a role to grant access to
```bash
aws kms create-key --description "Lab 4 - grants" --query KeyMetadata.KeyId --output text
# (reuse a role, or create one similar to Lab 3)
```

### Step 2: Create a grant
```bash
aws kms create-grant \
  --key-id <key-id> \
  --grantee-principal arn:aws:iam::$ACCOUNT_ID:role/Lab3TestRole \
  --operations Decrypt GenerateDataKey \
  --output json
```
Note the `GrantId` and `GrantToken` in the response.

### Step 3: List grants on the key
```bash
aws kms list-grants --key-id <key-id>
```

### Step 4: Revoke the grant
```bash
aws kms revoke-grant --key-id <key-id> --grant-id <grant-id>
aws kms list-grants --key-id <key-id>   # confirm it's gone
```

### ✅ Checkpoint
- You granted access without touching the key policy JSON at all
- You confirmed you can audit and revoke grants independently
- You understand this is exactly what happens behind the scenes when you enable SSE-KMS on an S3 bucket

### Clean up
```bash
aws kms schedule-key-deletion --key-id <key-id> --pending-window-in-days 7
```

---

## Lab 5 — Automatic Key Rotation

**Goal:** enable rotation and confirm old ciphertext still decrypts correctly afterward (conceptually — rotation itself takes real time to occur, so this lab focuses on configuration and verification, not waiting for an actual rotation event).

### Step 1: Create a key and enable rotation
```bash
aws kms create-key --description "Lab 5 - rotation" --query KeyMetadata.KeyId --output text
aws kms create-alias --alias-name alias/lab5-key --target-key-id <key-id>
aws kms enable-key-rotation --key-id alias/lab5-key --rotation-period-in-days 90
```

### Step 2: Confirm rotation status
```bash
aws kms get-key-rotation-status --key-id alias/lab5-key
```
You should see `KeyRotationEnabled: true` and `RotationPeriodInDays: 90`.

### Step 3: Encrypt something now, before any rotation happens
```bash
aws kms encrypt --key-id alias/lab5-key --plaintext "pre-rotation data" \
  --output text --query CiphertextBlob | base64 --decode > pre-rotation.encrypted
```

### Step 4: Reason through what happens at rotation time
When AWS rotates the backing material, the **key ID and ARN stay identical** — your alias and any application code referencing `alias/lab5-key` needs zero changes. AWS retains the old key material internally, so `pre-rotation.encrypted` will still decrypt correctly even after rotation, because KMS tracks which key material version encrypted which ciphertext.

### Step 5: Try rotation on a key type that doesn't support it (to see the guardrail)
```bash
aws kms create-key --key-usage SIGN_VERIFY --key-spec ECC_NIST_P256 \
  --description "Lab 5 - should reject rotation" --query KeyMetadata.KeyId --output text
aws kms enable-key-rotation --key-id <asymmetric-key-id>
```
This should fail — asymmetric keys don't support automatic rotation, confirming what Section 4.8 of the README explains.

### ✅ Checkpoint
- You enabled and verified rotation configuration
- You understand rotation doesn't break existing ciphertext
- You confirmed asymmetric keys reject rotation attempts

### Clean up
```bash
aws kms delete-alias --alias-name alias/lab5-key
aws kms schedule-key-deletion --key-id <key-id> --pending-window-in-days 7
aws kms schedule-key-deletion --key-id <asymmetric-key-id> --pending-window-in-days 7
rm -f pre-rotation.encrypted
```

---

## Lab 6 — Encrypting an S3 Bucket With Your Own Key (SSE-KMS)

**Goal:** see envelope encryption happen automatically via a real AWS service integration, and understand the grant it creates behind the scenes.

### Step 1: Create a key dedicated to this bucket
```bash
aws kms create-key --description "Lab 6 - S3 bucket encryption" --query KeyMetadata.KeyId --output text
aws kms create-alias --alias-name alias/lab6-s3-key --target-key-id <key-id>
```

### Step 2: Create a bucket and set default encryption to use your key
```bash
BUCKET_NAME="kms-lab6-$(date +%s)"
aws s3api create-bucket --bucket $BUCKET_NAME --region $AWS_REGION

aws s3api put-bucket-encryption \
  --bucket $BUCKET_NAME \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/lab6-s3-key"
      },
      "BucketKeyEnabled": true
    }]
  }'
```
Note `BucketKeyEnabled: true` — this is an S3 optimization that reduces KMS API calls (and cost) by reusing a bucket-level data key for multiple objects for a short time, instead of calling KMS per object.

### Step 3: Upload a file and confirm it's encrypted with your key
```bash
echo "sensitive report data" > report.txt
aws s3 cp report.txt s3://$BUCKET_NAME/report.txt

aws s3api head-object --bucket $BUCKET_NAME --key report.txt \
  --query '[ServerSideEncryption, SSEKMSKeyId]'
```
You should see `"aws:kms"` and your key's ARN.

### Step 4: Check the grant S3 created on your behalf
```bash
aws kms list-grants --key-id alias/lab6-s3-key
```
You'll see a grant with S3's service principal as the grantee — this is the exact mechanism from Section 4.5 of the README, created automatically without you touching the key policy.

### Step 5: Download the file and confirm it decrypts transparently
```bash
aws s3 cp s3://$BUCKET_NAME/report.txt report-downloaded.txt
diff report.txt report-downloaded.txt && echo "✅ Transparent decrypt worked"
```

### ✅ Checkpoint
- You saw a real AWS service use envelope encryption + a grant automatically
- You understand `BucketKeyEnabled` as a cost/performance optimization
- You confirmed the object metadata records which KMS key protected it

### Clean up
```bash
aws s3 rm s3://$BUCKET_NAME/report.txt
aws s3api delete-bucket --bucket $BUCKET_NAME
aws kms delete-alias --alias-name alias/lab6-s3-key
aws kms schedule-key-deletion --key-id <key-id> --pending-window-in-days 7
rm -f report.txt report-downloaded.txt
```

---

## Lab 7 — Multi-Region Keys for Cross-Region Encrypt/Decrypt

**Goal:** encrypt in one region, decrypt in another, without re-encrypting — and see what happens when you try this with a *regular* (non-multi-region) key for contrast.

### Step 1 (contrast case): try cross-region use of a normal key
```bash
aws kms create-key --region us-east-1 --description "Lab 7 - regular key" \
  --query KeyMetadata.KeyId --output text
aws kms encrypt --region us-east-1 --key-id <key-id> --plaintext "test" \
  --output text --query CiphertextBlob | base64 --decode > cross-region-test.encrypted

# Now try to decrypt it from a DIFFERENT region
aws kms decrypt --region us-west-2 --ciphertext-blob fileb://cross-region-test.encrypted
```
This fails — a standard key simply doesn't exist outside its home region.

### Step 2: Create a multi-region primary key
```bash
aws kms create-key --region us-east-1 --multi-region \
  --description "Lab 7 - multi-region primary" \
  --query KeyMetadata.KeyId --output text
```

### Step 3: Replicate it to another region
```bash
aws kms replicate-key \
  --key-id <primary-key-id> \
  --replica-region us-west-2 \
  --description "Lab 7 - multi-region replica"
```

### Step 4: Encrypt in the primary region
```bash
aws kms encrypt --region us-east-1 --key-id <primary-key-id> --plaintext "cross-region data" \
  --output text --query CiphertextBlob | base64 --decode > mr-test.encrypted
```

### Step 5: Decrypt using the REPLICA key ID, in the replica region
```bash
aws kms decrypt --region us-west-2 --key-id <replica-key-id> \
  --ciphertext-blob fileb://mr-test.encrypted \
  --output text --query Plaintext | base64 --decode
```
This works — because the replica shares the same underlying key material as the primary.

### ✅ Checkpoint
- You confirmed regular keys are strictly regional
- You created and used a multi-region primary + replica pair
- You proved ciphertext from one region decrypts cleanly via the replica in another

### Clean up
```bash
aws kms schedule-key-deletion --region us-east-1 --key-id <key-id> --pending-window-in-days 7
aws kms schedule-key-deletion --region us-east-1 --key-id <primary-key-id> --pending-window-in-days 7
aws kms schedule-key-deletion --region us-west-2 --key-id <replica-key-id> --pending-window-in-days 7
rm -f cross-region-test.encrypted mr-test.encrypted
```

---

## What You've Learned

By completing all seven labs, you've gone from creating your first key to:
- Building envelope encryption by hand and understanding *why* it exists
- Debugging real access-control interactions between key policies and IAM
- Using grants the way AWS services use them internally
- Configuring and reasoning about automatic key rotation
- Watching a real service (S3) use KMS transparently, including cost optimizations
- Solving cross-region encryption with multi-region keys

**Next step:** if something didn't work as expected along the way, check [`troubleshooting.md`](./troubleshooting.md) — most of the errors you might have hit are documented there with fixes.

---

*Related docs: [`README.md`](./README.md) · [`commands-cheatsheet.md`](./commands-cheatsheet.md) · [`troubleshooting.md`](./troubleshooting.md)*
