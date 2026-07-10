# AWS S3 — Commands & Policy Cheat Sheet

Quick-reference for every CLI command, JSON policy, and Terraform snippet covered in [`README.md`](./README.md). Organized in the same topic order — search by heading or `Ctrl+F` a keyword.

---

## Core Bucket Operations

```bash
# Create a bucket
aws s3 mb s3://my-new-bucket --region ap-south-1

# List buckets
aws s3 ls

# List objects under a prefix
aws s3 ls s3://my-bucket/logs/2026/ --recursive

# Copy a local file up
aws s3 cp ./file.txt s3://my-bucket/path/file.txt

# Sync a local folder (only uploads changed/new files)
aws s3 sync ./local-folder s3://my-bucket/remote-folder

# Delete an object
aws s3 rm s3://my-bucket/path/file.txt

# Delete a bucket (must be empty)
aws s3 rb s3://my-bucket

# Delete a bucket and everything in it
aws s3 rb s3://my-bucket --force

# Get object metadata without downloading
aws s3api head-object --bucket my-bucket --key path/file.txt
```

---

## Bucket Policy (PARC Model)

```bash
# Apply a bucket policy from a local JSON file
aws s3api put-bucket-policy --bucket my-bucket --policy file://policy.json

# View the current bucket policy
aws s3api get-bucket-policy --bucket my-bucket --query Policy --output text

# Delete a bucket policy entirely
aws s3api delete-bucket-policy --bucket my-bucket
```

**Enforce HTTPS-only:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "EnforceHTTPSOnly",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::production-app-assets", "arn:aws:s3:::production-app-assets/*"],
    "Condition": { "Bool": { "aws:SecureTransport": "false" } }
  }]
}
```

**Trust an entire AWS Organization:**
```json
"Condition": { "StringEquals": { "aws:PrincipalOrgId": "o-xxxxxxxxxx" } }
```

---

## Cross-Account Access

**Method 1 — direct resource-based access.** Bucket policy in Account A:
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "AllowCrossAccountAccess",
        "Effect": "Allow",
        "Principal": { "AWS": "arn:aws:iam::ACCOUNT_B_ID:role/AccountBWorkloadRole" },
        "Action": ["s3:GetObject", "s3:ListBucket", "s3:PutObject"],
        "Resource": ["arn:aws:s3:::account-a-data-bucket", "arn:aws:s3:::account-a-data-bucket/*"]
    }]
}
```

IAM policy in Account B:
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "AllowAccessToAccountABucket",
        "Effect": "Allow",
        "Action": ["s3:GetObject", "s3:ListBucket", "s3:PutObject"],
        "Resource": ["arn:aws:s3:::account-a-data-bucket", "arn:aws:s3:::account-a-data-bucket/*"]
    }]
}
```

**Method 2 — STS AssumeRole.**
```bash
# Account B calls this to get temporary credentials for the role in Account A
aws sts assume-role \
    --role-arn arn:aws:iam::ACCOUNT_A_ID:role/CrossAccountS3AccessRole \
    --role-session-name my-session

# Then export the returned AccessKeyId / SecretAccessKey / SessionToken and call S3 normally
```

**Fix Object Ownership so Account A can manage objects uploaded by Account B:**
```bash
aws s3api put-bucket-ownership-controls \
    --bucket account-a-data-bucket \
    --ownership-controls '{"Rules":[{"ObjectOwnership":"BucketOwnerEnforced"}]}'
```


---

## Presigned URLs

```bash
# Generate a GET presigned URL, valid for 15 minutes
aws s3 presign s3://my-secure-bucket/architecture-diagram.pdf --expires-in 900
```

```python
import boto3

def create_presigned_url(bucket_name, object_name, expiration=3600):
    s3_client = boto3.client('s3')
    return s3_client.generate_presigned_url(
        'get_object',
        Params={'Bucket': bucket_name, 'Key': object_name},
        ExpiresIn=expiration
    )

# For uploads, swap the operation:
def create_presigned_put_url(bucket_name, object_name, expiration=3600):
    s3_client = boto3.client('s3')
    return s3_client.generate_presigned_url(
        'put_object',
        Params={'Bucket': bucket_name, 'Key': object_name},
        ExpiresIn=expiration
    )
```

```bash
# Consume a presigned PUT URL from the frontend (raw HTTP, no AWS creds needed)
curl -X PUT -T ./myfile.zip "https://my-bucket.s3.amazonaws.com/uploads/myfile.zip?X-Amz-..."
```

---

## Requester Pays

```bash
# Owner: enable Requester Pays
aws s3api put-bucket-request-payment \
    --bucket your-shared-data-bucket \
    --request-payment-configuration '{"Payer": "Requester"}'

# Owner: check current setting
aws s3api get-bucket-request-payment --bucket your-shared-data-bucket

# Requester: must include the flag on every call
aws s3 cp s3://your-shared-data-bucket/dataset.zip . --request-payer requester
aws s3 ls s3://your-shared-data-bucket/ --request-payer requester
```

```python
# Boto3 equivalent
s3_client.get_object(Bucket='your-shared-data-bucket', Key='dataset.zip', RequestPayer='requester')
```

---

## Versioning & Delete Markers

```bash
# Enable versioning
aws s3api put-bucket-versioning --bucket my-bucket --versioning-configuration Status=Enabled

# Suspend versioning (cannot fully disable once enabled)
aws s3api put-bucket-versioning --bucket my-bucket --versioning-configuration Status=Suspended

# List all versions (including delete markers)
aws s3api list-object-versions --bucket my-bucket --prefix path/file.txt

# Restore a "deleted" object: delete the delete-marker itself
aws s3api delete-object --bucket my-bucket --key path/file.txt --version-id <DELETE_MARKER_VERSION_ID>

# Permanently purge a specific version (irreversible)
aws s3api delete-object --bucket my-bucket --key path/file.txt --version-id <VERSION_ID>

# Enable MFA Delete (must use ROOT account credentials)
aws s3api put-bucket-versioning \
    --bucket my-bucket \
    --versioning-configuration Status=Enabled,MFADelete=Enabled \
    --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device 123456"
```

**Noncurrent version cleanup lifecycle rule:**
```json
{
  "Rules": [{
    "ID": "ManageNoncurrentVersions",
    "Status": "Enabled",
    "Filter": {"Prefix": "logs/"},
    "NoncurrentVersionExpiration": { "NoncurrentDays": 90, "NewerNoncurrentVersions": 2 }
  }]
}
```


---

## Multipart Upload

```bash
# High-level CLI auto-multiparts anything above ~8MB — nothing extra needed
aws s3 cp ./huge-backup.tar.gz s3://my-bucket/backups/huge-backup.tar.gz

# --- Manual / low-level walkthrough ---

# 1. Initiate
aws s3api create-multipart-upload --bucket my-bucket --key backups/huge-backup.tar.gz
# → returns an UploadId

# 2. Upload each part (repeat per chunk, PartNumber 1..N)
aws s3api upload-part \
    --bucket my-bucket --key backups/huge-backup.tar.gz \
    --part-number 1 --body part1.bin \
    --upload-id <UPLOAD_ID>
# → each call returns an ETag; record PartNumber -> ETag

# 3. Complete (after uploading & recording every part)
aws s3api complete-multipart-upload \
    --bucket my-bucket --key backups/huge-backup.tar.gz \
    --upload-id <UPLOAD_ID> \
    --multipart-upload file://parts-manifest.json

# parts-manifest.json looks like:
# { "Parts": [ {"ETag": "\"abc123\"", "PartNumber": 1}, {"ETag": "\"def456\"", "PartNumber": 2} ] }

# List in-progress multipart uploads (find orphans)
aws s3api list-multipart-uploads --bucket my-bucket

# Manually abort an orphaned upload
aws s3api abort-multipart-upload --bucket my-bucket --key backups/huge-backup.tar.gz --upload-id <UPLOAD_ID>
```

**Lifecycle rule to auto-clean orphaned multipart chunks:**
```json
{
  "Rules": [{
    "ID": "AbortIncompleteMultipartUploadsAfter7Days",
    "Status": "Enabled",
    "Filter": {},
    "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
  }]
}
```

```python
# Boto3 Transfer Manager handles chunking, threading, and ETag tracking for you
import boto3
from boto3.s3.transfer import TransferConfig

config = TransferConfig(multipart_threshold=8*1024*1024, max_concurrency=10)
s3 = boto3.client('s3')
s3.upload_file('huge-backup.tar.gz', 'my-bucket', 'backups/huge-backup.tar.gz', Config=config)
```

---

## S3 Transfer Acceleration

```bash
# Enable on the bucket
aws s3api put-bucket-accelerate-configuration \
    --bucket my-bucket \
    --accelerate-configuration Status=Enabled

# Check current status
aws s3api get-bucket-accelerate-configuration --bucket my-bucket

# Use the accelerated endpoint explicitly
aws s3 cp ./bigfile.mp4 s3://my-bucket/bigfile.mp4 --endpoint-url https://my-bucket.s3-accelerate.amazonaws.com

# Or globally for the CLI session
aws configure set default.s3.use_accelerate_endpoint true
```


---

## Encryption — SSE-S3 / SSE-KMS / SSE-C

```bash
# Set default bucket encryption to SSE-S3 (AES-256)
aws s3api put-bucket-encryption \
    --bucket my-bucket \
    --server-side-encryption-configuration '{
        "Rules": [{ "ApplyServerSideEncryptionByDefault": { "SSEAlgorithm": "AES256" } }]
    }'

# Set default bucket encryption to SSE-KMS with a specific CMK, and enable S3 Bucket Keys
aws s3api put-bucket-encryption \
    --bucket my-bucket \
    --server-side-encryption-configuration '{
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "aws:kms",
                "KMSMasterKeyID": "arn:aws:kms:ap-south-1:111122223333:key/abcd-1234"
            },
            "BucketKeyEnabled": true
        }]
    }'

# Upload a single object with SSE-KMS explicitly
aws s3api put-object \
    --bucket my-bucket --key secrets/data.csv --body ./data.csv \
    --server-side-encryption aws:kms \
    --ssekms-key-id arn:aws:kms:ap-south-1:111122223333:key/abcd-1234

# Upload with SSE-C (customer-supplied key) — you must re-supply the key on every read too
aws s3api put-object \
    --bucket my-bucket --key secrets/data.csv --body ./data.csv \
    --sse-customer-algorithm AES256 \
    --sse-customer-key <base64-encoded-256-bit-key>
```

**KMS key policy: grant a cross-account role access to decrypt/generate data keys**
```json
{
  "Sid": "AllowAccountBDecrypt",
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::ACCOUNT_B_ID:role/AccountBWorkloadRole" },
  "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
  "Resource": "*"
}
```

```bash
# Enable S3 Bucket Keys on an existing SSE-KMS bucket (cuts KMS API calls ~99%)
aws s3api put-bucket-encryption \
    --bucket my-bucket \
    --server-side-encryption-configuration '{
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": { "SSEAlgorithm": "aws:kms", "KMSMasterKeyID": "alias/my-cmk" },
            "BucketKeyEnabled": true
        }]
    }'
```

```python
# Client-Side Encryption using the AWS Encryption SDK with a KMS-backed keyring
import boto3
from aws_encryption_sdk import EncryptionSDKClient, StrictAwsKmsMasterKeyProvider

client = EncryptionSDKClient()
kms_key_provider = StrictAwsKmsMasterKeyProvider(key_ids=['arn:aws:kms:ap-south-1:111122223333:key/abcd-1234'])

ciphertext, header = client.encrypt(source=b"sensitive payload", key_provider=kms_key_provider)

s3 = boto3.client('s3')
s3.put_object(Bucket='my-bucket', Key='cse-data/file.enc', Body=ciphertext)
```


---

## Storage Classes

```bash
# Upload directly into a specific storage class
aws s3 cp ./archive.zip s3://my-bucket/archive.zip --storage-class GLACIER

# Valid --storage-class values:
# STANDARD | STANDARD_IA | ONEZONE_IA | INTELLIGENT_TIERING
# GLACIER_IR | GLACIER | DEEP_ARCHIVE | REDUCED_REDUNDANCY | EXPRESS_ONEZONE

# Change the storage class of an existing object (this is a COPY under the hood)
aws s3 cp s3://my-bucket/data.csv s3://my-bucket/data.csv \
    --storage-class GLACIER --metadata-directive COPY

# Check an object's current storage class
aws s3api head-object --bucket my-bucket --key data.csv --query StorageClass

# Restore a Glacier object back to a readable state temporarily
aws s3api restore-object \
    --bucket my-bucket --key archive.zip \
    --restore-request '{"Days":7,"GlacierJobParameters":{"Tier":"Expedited"}}'

# Enable Intelligent-Tiering as the default for a whole prefix via a bucket-level config
aws s3api put-bucket-intelligent-tiering-configuration \
    --bucket my-bucket --id "auto-tier-all" \
    --intelligent-tiering-configuration '{
        "Id": "auto-tier-all",
        "Status": "Enabled",
        "Tierings": [
            {"Days": 90, "AccessTier": "ARCHIVE_ACCESS"},
            {"Days": 180, "AccessTier": "DEEP_ARCHIVE_ACCESS"}
        ]
    }'
```

---

## Lifecycle Configuration

```bash
# Apply a lifecycle configuration from file
aws s3api put-bucket-lifecycle-configuration \
    --bucket my-bucket \
    --lifecycle-configuration file://lifecycle.json

# View current lifecycle rules
aws s3api get-bucket-lifecycle-configuration --bucket my-bucket

# Remove all lifecycle rules
aws s3api delete-bucket-lifecycle --bucket my-bucket
```

**Full production `lifecycle.json` (transitions + expirations + versioning + multipart cleanup):**
```json
{
  "Rules": [
    {
      "ID": "archive-and-cleanup-application-logs",
      "Status": "Enabled",
      "Filter": { "Prefix": "app-logs/" },
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 },
      "NoncurrentVersionTransitions": [
        { "NoncurrentDays": 14, "StorageClass": "GLACIER" }
      ],
      "NoncurrentVersionExpiration": { "NoncurrentDays": 90 },
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    },
    {
      "ID": "cleanup-orphaned-delete-markers",
      "Status": "Enabled",
      "Filter": {},
      "Expiration": { "ExpiredObjectDeleteMarker": true }
    }
  ]
}
```

**Terraform equivalent:**
```hcl
resource "aws_s3_bucket_lifecycle_configuration" "log_bucket_lifecycle" {
  bucket = "my-company-production-logs"

  rule {
    id     = "archive-and-cleanup-application-logs"
    status = "Enabled"
    filter { prefix = "app-logs/" }

    transition { days = 30  storage_class = "STANDARD_IA" }
    transition { days = 90  storage_class = "GLACIER" }
    expiration { days = 365 }

    noncurrent_version_transition { noncurrent_days = 14  storage_class = "GLACIER" }
    noncurrent_version_expiration { noncurrent_days = 90 }
  }
}
```


---

## Replication (CRR / SRR)

```bash
# Both source and destination buckets need versioning enabled first
aws s3api put-bucket-versioning --bucket source-bucket --versioning-configuration Status=Enabled
aws s3api put-bucket-versioning --bucket destination-bucket --versioning-configuration Status=Enabled

# Apply a replication configuration
aws s3api put-bucket-replication \
    --bucket source-bucket \
    --replication-configuration file://replication.json

# Check current replication config
aws s3api get-bucket-replication --bucket source-bucket

# Kick off a one-time batch job to replicate pre-existing objects
aws s3control create-job \
    --account-id 111122223333 \
    --operation '{"S3ReplicateObject": {}}' \
    --manifest file://manifest-spec.json \
    --report file://report-spec.json \
    --priority 10 \
    --role-arn arn:aws:iam::111122223333:role/BatchReplicationRole
```

**`replication.json` (basic CRR with storage-class override on the copy):**
```json
{
  "Role": "arn:aws:iam::111122223333:role/s3-replication-role",
  "Rules": [{
    "ID": "disaster-recovery-sync",
    "Status": "Enabled",
    "Filter": {},
    "Destination": {
      "Bucket": "arn:aws:s3:::destination-bucket",
      "StorageClass": "STANDARD_IA"
    }
  }]
}
```

**Terraform:**
```hcl
resource "aws_s3_bucket_replication_configuration" "replication" {
  depends_on = [aws_s3_bucket_versioning.source_versioning]
  role   = aws_iam_role.replication_exec_role.arn
  bucket = aws_s3_bucket.source.id

  rule {
    id     = "disaster-recovery-sync"
    status = "Enabled"
    destination {
      bucket        = aws_s3_bucket.destination.arn
      storage_class = "STANDARD_IA"
    }
  }
}
```

---

## S3 Select / Glacier Select

```bash
# S3 Select against a CSV object (synchronous)
aws s3api select-object-content \
    --bucket my-bucket --key data/transactions.csv \
    --expression "SELECT * FROM s3object s WHERE s.error_id = 'ERR-999'" \
    --expression-type SQL \
    --input-serialization '{"CSV": {"FileHeaderInfo": "USE"}, "CompressionType": "NONE"}' \
    --output-serialization '{"CSV": {}}' \
    output.csv

# Glacier Select — async restore + query in one call
aws s3api restore-object \
    --bucket my-bucket --key archive/2023-transactions.csv \
    --restore-request '{
        "Days": 1,
        "GlacierJobParameters": { "Tier": "Standard" },
        "OutputLocation": {
            "S3": {
                "BucketName": "my-results-bucket",
                "Prefix": "select-results/",
                "Encryption": { "EncryptionType": "AES256" }
            }
        },
        "SelectParameters": {
            "InputSerialization": { "CSV": {} },
            "ExpressionType": "SQL",
            "Expression": "SELECT * FROM s3object s WHERE s.status = '"'"'FAILED'"'"'",
            "OutputSerialization": { "CSV": {} }
        }
    }'
```


---

## Object Lock (WORM)

```bash
# Object Lock can only be set at bucket CREATION
aws s3api create-bucket \
    --bucket company-immutable-audit-vault \
    --region ap-south-1 \
    --create-bucket-configuration LocationConstraint=ap-south-1 \
    --object-lock-enabled-for-bucket

# Set a bucket-level default retention rule
aws s3api put-object-lock-configuration \
    --bucket company-immutable-audit-vault \
    --object-lock-configuration '{
        "ObjectLockEnabled": "Enabled",
        "Rule": { "DefaultRetention": { "Mode": "COMPLIANCE", "Days": 365 } }
    }'

# Apply retention to a specific object version
aws s3api put-object-retention \
    --bucket company-immutable-audit-vault --key audit/report.pdf \
    --retention '{"Mode":"GOVERNANCE","RetainUntilDate":"2027-01-01T00:00:00Z"}'

# Apply / remove a Legal Hold
aws s3api put-object-legal-hold --bucket company-immutable-audit-vault --key audit/report.pdf \
    --legal-hold '{"Status":"ON"}'
aws s3api put-object-legal-hold --bucket company-immutable-audit-vault --key audit/report.pdf \
    --legal-hold '{"Status":"OFF"}'

# Bypass a Governance-mode lock (requires s3:BypassGovernanceRetention permission)
aws s3api delete-object \
    --bucket company-immutable-audit-vault --key audit/report.pdf \
    --version-id <VERSION_ID> --bypass-governance-retention
```

**Terraform:**
```hcl
resource "aws_s3_bucket" "compliance_log_vault" {
  bucket              = "company-immutable-audit-vault"
  object_lock_enabled = true
}

resource "aws_s3_bucket_versioning" "vault_versioning" {
  bucket = aws_s3_bucket.compliance_log_vault.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_object_lock_configuration" "vault_lock_config" {
  depends_on = [aws_s3_bucket_versioning.vault_versioning]
  bucket     = aws_s3_bucket.compliance_log_vault.id
  rule {
    default_retention { mode = "COMPLIANCE"  days = 365 }
  }
}
```

---

## Advanced / 2025–2026 Features

```bash
# --- S3 Tables ---
aws s3tables create-table-bucket --name my-analytics-tables
aws s3tables create-namespace --table-bucket-arn <ARN> --namespace analytics
aws s3tables create-table --table-bucket-arn <ARN> --namespace analytics --name events --format ICEBERG

# --- S3 Vectors ---
aws s3vectors create-vector-bucket --vector-bucket-name my-rag-vectors
aws s3vectors create-index \
    --vector-bucket-name my-rag-vectors --index-name doc-embeddings \
    --dimension 1536 --distance-metric cosine \
    --data-type float32

# --- S3 Files (mount an existing general-purpose bucket as NFS) ---
aws s3files create-file-system --bucket my-bucket
aws s3files create-mount-target --file-system-id <FS_ID> --subnet-id subnet-abc123 --security-groups sg-abc123
sudo mkdir /mnt/s3files && sudo mount -t s3files <FS_ID>:/ /mnt/s3files

# --- S3 Batch Operations (bulk re-encrypt a whole bucket, for example) ---
aws s3control create-job \
    --account-id 111122223333 \
    --operation '{"S3PutObjectCopy": {"TargetKeyPrefix": "", "StorageClass": "STANDARD"}}' \
    --manifest file://manifest-spec.json \
    --report file://report-spec.json \
    --priority 5 \
    --role-arn arn:aws:iam::111122223333:role/BatchOperationsRole

# --- Account Regional Namespaces (guaranteed, squat-proof bucket names) ---
aws s3api create-bucket \
    --bucket mybucket-123456789012-us-east-1-an \
    --bucket-namespace account-regional \
    --region us-east-1

# --- Object Annotations ---
aws s3api put-object-annotations \
    --bucket my-bucket --key datasets/customers.parquet \
    --annotations file://annotations.json
```

