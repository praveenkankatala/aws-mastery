# AWS KMS — Troubleshooting Guide

> Real error messages you'll actually run into, what's really going on under the hood, and how to fix them. Organized by category so you can jump straight to the relevant section.

**Related docs:** [`README.md`](./README.md) · [`commands-cheatsheet.md`](./commands-cheatsheet.md) · [`hands-on-labs.md`](./hands-on-labs.md)

---

## Table of Contents
1. [Access Denied Errors](#1-access-denied-errors)
2. [Payload & Size Errors](#2-payload--size-errors)
3. [Key State Errors](#3-key-state-errors)
4. [Region & Cross-Region Errors](#4-region--cross-region-errors)
5. [Encryption Context Errors](#5-encryption-context-errors)
6. [Rotation Errors](#6-rotation-errors)
7. [Grant Errors](#7-grant-errors)
8. [Import / BYOK Errors](#8-import--byok-errors)
9. [Throttling & Quota Errors](#9-throttling--quota-errors)
10. [Alias Errors](#10-alias-errors)
11. [General Debugging Workflow](#11-general-debugging-workflow)

---

## 1. Access Denied Errors

### `AccessDeniedException: User is not authorized to perform: kms:Decrypt`

**What's really happening:** this is almost always one of two things — a missing IAM policy statement, or a key policy that doesn't delegate to IAM.

**Fix, step by step:**
1. Confirm the caller's identity: `aws sts get-caller-identity`
2. Check the caller's IAM policy actually allows `kms:Decrypt` on this specific key ARN (not just `"Resource": "*"` on an unrelated policy).
3. Check the **key policy** includes the `"Enable IAM User Permissions"` statement delegating to account root/IAM. Without it, IAM policies are irrelevant no matter how permissive they are.
4. Use the simulator to get a definitive answer instead of guessing:
   ```bash
   aws iam simulate-principal-policy \
     --policy-source-arn <caller-arn> \
     --action-names kms:Decrypt \
     --resource-arns <key-arn>
   ```
5. If using a role assumed via `AssumeRole`, double-check the role's *trust policy* too — a correctly scoped KMS permission is useless if the role can't be assumed in the first place.

**Remember:** both the key policy AND the relevant identity-based (IAM) policy must say yes. It's an AND, not an OR.

---

### `AccessDeniedException` even though your IAM policy looks correct and uses `Resource: "*"`

**What's really happening:** a wildcard resource in an IAM policy still has to be permitted by the key's own policy. Key policies are evaluated independently and are the authoritative gate for resource-based access.

**Fix:** inspect the key policy directly:
```bash
aws kms get-key-policy --key-id <key-id> --policy-name default --output text
```
Look for whether the calling principal (or the IAM-delegation statement) is actually referenced.

---

### `AccessDeniedException` when calling from a Lambda function / EC2 instance

**What's really happening:** the execution role attached to the compute resource doesn't have KMS permissions — a very common gap, since the *console user* creating the Lambda often has broad permissions and doesn't notice the function's own role doesn't.

**Fix:** check the specific execution role (not your own user):
```bash
aws lambda get-function-configuration --function-name <fn> --query Role
aws iam list-attached-role-policies --role-name <role-name-from-above>
```

---

## 2. Payload & Size Errors

### `ValidationException: 1 validation error detected: Value at 'plaintext' failed to satisfy constraint: Member must have length less than or equal to 4096`

**What's really happening:** you tried to `Encrypt`/`Decrypt` something larger than KMS's 4 KB direct payload limit. This is by design — KMS keys are meant to wrap small data keys, not encrypt bulk data directly.

**Fix:** switch to envelope encryption — generate a data key with `GenerateDataKey`, encrypt your actual payload locally (e.g., with AES-256 via OpenSSL or a crypto library), and only send the small data key itself to KMS. See [Lab 2](./hands-on-labs.md#lab-2--envelope-encryption-from-scratch) for the full walkthrough.

---

### `InvalidCiphertextException: The specified ciphertext is corrupted or otherwise invalid`

**What's really happening:** usually one of:
- The ciphertext blob got base64-encoded/decoded incorrectly somewhere along the way (very common when copy-pasting between CLI/JSON/files).
- You're passing a *plaintext* file where an *encrypted* blob was expected, or vice versa.
- The ciphertext was truncated or modified (e.g., accidentally opened/saved as text instead of binary).

**Fix:** always read/write ciphertext as raw binary (`fileb://` in CLI, binary mode in code — never treat it as a UTF-8 string). Re-run the encrypt step and diff the resulting file sizes if unsure.

---

## 3. Key State Errors

### `DisabledException: <key-arn> is disabled`

**What's really happening:** the key exists but has been explicitly disabled (via `DisableKey` or the console).

**Fix:**
```bash
aws kms describe-key --key-id <key-id> --query KeyMetadata.KeyState
aws kms enable-key --key-id <key-id>
```

---

### `KMSInvalidStateException: <key-arn> is pending deletion`

**What's really happening:** someone scheduled this key for deletion, and it's currently in its waiting window (`PendingDeletion` state) — all crypto operations are blocked during this time.

**Fix — if you still need the key:**
```bash
aws kms cancel-key-deletion --key-id <key-id>
aws kms enable-key --key-id <key-id>   # cancellation leaves it Disabled — re-enable it
```
**If you don't need it:** nothing to do — it will finish deleting automatically once the window elapses.

---

### `KMSInvalidStateException` when trying to import key material

**What's really happening:** you're trying to `ImportKeyMaterial` on a key that isn't in the `PendingImport` state — usually because the import token/public key you're using is stale or expired (import tokens expire, typically within 24 hours).

**Fix:** re-run `GetParametersForImport` to get a fresh token and public key, re-wrap your material, and import again promptly.

---

## 4. Region & Cross-Region Errors

### `NotFoundException: Key '<key-id>' does not exist` (when you're SURE it exists)

**What's really happening:** almost always a region mismatch. KMS keys (other than multi-region keys) are strictly regional resources — a key ID that's valid in `us-east-1` means nothing in `eu-west-1`.

**Fix:**
```bash
# Explicitly set the region rather than relying on a default
aws kms describe-key --key-id <key-id> --region us-east-1
```
Also check `$AWS_DEFAULT_REGION`/`~/.aws/config` for a region that doesn't match where you actually created the key.

**If you genuinely need cross-region use:** you need a Multi-Region key, not a regular one — see [Lab 7](./hands-on-labs.md#lab-7--multi-region-keys-for-cross-region-encryptdecrypt).

---

### `UnsupportedOperationException` when calling `ReplicateKey`

**What's really happening:** you're trying to replicate a key that wasn't created with `--multi-region` in the first place. Multi-region-ness is set at creation time and can't be added to an existing standard key.

**Fix:** you'll need to create a new multi-region primary key; there's no in-place conversion. Plan for multi-region needs at creation time when possible.

---

## 5. Encryption Context Errors

### `InvalidCiphertextException` after previously successful decrypts of the same-looking data

**What's really happening:** your application is passing a different (or missing) `EncryptionContext` on decrypt than what was used during encrypt. Since encryption context is cryptographically bound to the ciphertext (as AAD), even a single differing key/value causes decryption to fail — and the error message doesn't explicitly say "encryption context mismatch," which makes this one sneaky to debug.

**Fix:**
1. Log (non-secret) the encryption context used at encrypt time.
2. Confirm the exact same key-value pairs are supplied at decrypt time — including matching key names and casing.
3. If context is dynamic (e.g., includes a timestamp or request ID), that's the bug — encryption context should be *stable* metadata like `customer-id`, not something that changes between encrypt and decrypt.

---

## 6. Rotation Errors

### `UnsupportedOperationException: <key-arn> key spec does not support automatic key rotation`

**What's really happening:** you tried to enable rotation on an asymmetric key, an HMAC key, or a key with imported (`EXTERNAL`) material — none of which support AWS's automatic rotation.

**Fix:** there's no way to enable automatic rotation for these key types. Instead, implement manual rotation: create a new key, update your alias to point at it, and phase out the old key once nothing depends on it anymore (keep it around, disabled rather than deleted, until you're sure old ciphertext doesn't still need it).

---

### "I enabled rotation but my old ciphertext stopped decrypting" (this should never actually happen — debugging the real cause)

**What's really happening:** rotation itself does *not* break old ciphertext — AWS retains all historical key material internally. If decryption broke after "rotation," the real cause is almost always something else that coincided with it: someone deleted imported key material, scheduled key deletion, or changed the key policy at the same time.

**Fix:** check `DescribeKey` for the actual `KeyState`, and check CloudTrail around the time things broke for any `PutKeyPolicy`, `DisableKey`, or `ScheduleKeyDeletion` events — rotation is very unlikely to be the actual culprit.

---

## 7. Grant Errors

### `AccessDeniedException` even after creating a grant

**What's really happening:** grants often require presenting a **grant token** in the same request if the grant was *just* created — there's a small eventual-consistency window before a new grant is fully recognized everywhere.

**Fix:** pass the grant token returned by `CreateGrant` explicitly in the subsequent `Decrypt`/`GenerateDataKey` call:
```bash
aws kms decrypt --ciphertext-blob fileb://file.enc --grant-tokens <token-from-create-grant>
```

---

### `NotFoundException: Grant does not exist` when calling `RevokeGrant`

**What's really happening:** either the grant ID is wrong/stale, or the grant was already retired/revoked (grants aren't idempotent-listable after removal).

**Fix:** re-list current grants first rather than reusing an old ID:
```bash
aws kms list-grants --key-id <key-id>
```

---

## 8. Import / BYOK Errors

### `IncorrectKeyMaterialException` during `ImportKeyMaterial`

**What's really happening:** the key material you wrapped doesn't match what KMS expects — usually a wrapping algorithm mismatch, wrong key length (KMS symmetric keys expect exactly 256-bit/32-byte material), or you wrapped it with the wrong public key/token pairing.

**Fix:** make sure the wrapping algorithm used with `openssl`/your tool exactly matches what you specified in `GetParametersForImport` (e.g., `RSAES_OAEP_SHA_256`), and that you're using the public key and import token from the *same* `GetParametersForImport` call — mixing tokens from different calls will fail.

---

### `ExpiredImportTokenException`

**What's really happening:** import tokens are short-lived (commonly ~24 hours) and yours expired before you finished wrapping and importing the material.

**Fix:** call `GetParametersForImport` again to get a fresh token/public key pair, and complete the wrap-and-import steps promptly this time.

---

## 9. Throttling & Quota Errors

### `ThrottlingException: Rate exceeded`

**What's really happening:** you've exceeded the per-account, per-region, per-API-action request rate quota — common under high-volume workloads calling `GenerateDataKey`/`Decrypt` very frequently (e.g., per-row database encryption without caching).

**Fix:**
1. Implement exponential backoff with jitter for retries (the AWS SDKs do this automatically in most cases — make sure you're not disabling it).
2. Reduce KMS call volume where possible: cache data keys client-side for a short TTL instead of calling `GenerateDataKey` per item; for S3, enable `BucketKeyEnabled` (see [Lab 6](./hands-on-labs.md#lab-6--encrypting-an-s3-bucket-with-your-own-key-sse-kms)).
3. If sustained high volume is a genuine, ongoing requirement, request a quota increase via Service Quotas for the specific API action you're hitting.

---

## 10. Alias Errors

### `AlreadyExistsException: An alias with the name arn:aws:kms:...:alias/my-key already exists`

**What's really happening:** aliases are unique per account+region — you're trying to create one that's already taken (possibly by a different key than you think).

**Fix:** check what it currently points to before reassigning:
```bash
aws kms list-aliases --query "Aliases[?AliasName=='alias/my-key']"
```
If you genuinely want to repoint it, use `update-alias` instead of `create-alias`.

---

### `NotFoundException: Alias arn:aws:kms:...:alias/my-key is not found` (right after you thought you created it)

**What's really happening:** almost always a region mismatch again, or a typo — aliases must include the `alias/` prefix exactly, and are case-sensitive.

**Fix:** double check the exact alias name and region with `list-aliases`, and make sure you're not omitting the `alias/` prefix in your code (a surprisingly common bug — passing just `my-key` instead of `alias/my-key`).

---

## 11. General Debugging Workflow

When something with KMS isn't working and you're not sure why, work through this order — it resolves the vast majority of issues fastest:

1. **Confirm identity:** `aws sts get-caller-identity` — are you (or your app's role) who you think you are?
2. **Confirm region:** is the key ID/alias you're using valid in the region you're actually calling?
3. **Confirm key state:** `aws kms describe-key --key-id <id>` — is it `Enabled`, or `Disabled`/`PendingDeletion`/`PendingImport`?
4. **Confirm key policy:** does it delegate to IAM, and/or explicitly allow the calling principal?
5. **Confirm IAM policy:** does the caller's identity-based policy allow the specific action on the specific key ARN?
6. **Simulate it directly** rather than guessing:
   ```bash
   aws iam simulate-principal-policy \
     --policy-source-arn <caller-arn> \
     --action-names <kms-action> \
     --resource-arns <key-arn>
   ```
7. **Check CloudTrail** for the actual denied call — it will show you exactly which check failed, often more precisely than the client-side error message:
   ```bash
   aws cloudtrail lookup-events \
     --lookup-attributes AttributeKey=EventName,AttributeValue=<ActionName> \
     --max-results 10
   ```
8. **Check for encryption context mismatches** if the error is decrypt-specific rather than a flat access denial.

If you make it through all eight steps and you're still stuck, the answer is almost always in the CloudTrail event detail — it records the precise reason (`errorCode`/`errorMessage`) AWS's own evaluation engine used to deny the request.

---

*Related docs: [`README.md`](./README.md) · [`commands-cheatsheet.md`](./commands-cheatsheet.md) · [`hands-on-labs.md`](./hands-on-labs.md)*
