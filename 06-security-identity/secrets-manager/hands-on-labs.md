# Hands-On Labs

Twelve guided labs, in order, taking you from a bare AWS account to a fully working config + secrets pipeline with Terraform, IAM, and application code. Each lab has an **Objective**, **Steps**, and **Validation** so you can confirm it worked before moving on.

> Prerequisite for all labs: an AWS account with CLI access configured (`aws configure`), and permissions to create SSM parameters, Secrets Manager secrets, KMS keys, and IAM policies in a sandbox account. Do **not** run these against a production account.

---

## Lab 1 — Design and seed a path hierarchy

**Objective:** create a realistic multi-environment parameter tree.

**Steps:**
```bash
aws ssm put-parameter --name "/dev/myapp/db_url" \
  --value "jdbc:mysql://dev-db.cluster.internal:3306/mydb" --type "String" --overwrite

aws ssm put-parameter --name "/dev/myapp/db_password" \
  --value "SuperSecretPassword123!" --type "SecureString" --overwrite

aws ssm put-parameter --name "/global/dns/primary_domain" \
  --value "example.com" --type "String" --overwrite
```

**Validation:**
```bash
aws ssm get-parameters-by-path --path "/dev/myapp/" --recursive
```
You should see both `db_url` and `db_password` (as ciphertext, since you didn't pass `--with-decryption`).

---

## Lab 2 — Read parameters three ways

**Objective:** understand the difference between single reads, decrypted reads, and bulk reads.

**Steps:**
```bash
# 1. Single, no decryption
aws ssm get-parameter --name "/dev/myapp/db_password"

# 2. Single, with decryption
aws ssm get-parameter --name "/dev/myapp/db_password" --with-decryption

# 3. Bulk path read
aws ssm get-parameters-by-path --path "/dev/myapp/" --recursive --with-decryption
```

**Validation:** Compare the `Value` field across the three calls — (1) should be ciphertext-looking, (2) and (3) should show `SuperSecretPassword123!` in cleartext.

---

## Lab 3 — Consume parameters from Python

**Objective:** load config into an app at runtime with a single API call.

**Steps:** Save as `load_config.py`:
```python
import boto3

def load_env_config():
    client = boto3.client('ssm', region_name='us-east-1')
    response = client.get_parameters_by_path(
        Path='/dev/myapp/',
        Recursive=True,
        WithDecryption=True
    )
    return {p['Name'].split('/')[-1]: p['Value'] for p in response['Parameters']}

if __name__ == "__main__":
    config = load_env_config()
    print(f"Database Target: {config.get('db_url')}")
```
Run it: `python3 load_config.py`

**Validation:** Output should print the decrypted `db_url` value without you ever hardcoding it in the script.

---

## Lab 4 — Version history and rollback

**Objective:** prove that overwriting a parameter preserves history.

**Steps:**
```bash
aws ssm put-parameter --name "/dev/myapp/db_password" --value "NewPassword456!" --type "SecureString" --overwrite
aws ssm get-parameter --name "/dev/myapp/db_password:1" --with-decryption
aws ssm get-parameter-history --name "/dev/myapp/db_password" --with-decryption
```

**Validation:** Version `:1` should still return the original password, even though the "current" value is now `NewPassword456!`.

---

## Lab 5 — Provision with Terraform

**Objective:** move from manual CLI seeding to reproducible IaC.

**Steps:** Save as `main.tf`, then run `terraform init && terraform apply`:
```hcl
resource "aws_kms_key" "ssm_key" {
  description             = "KMS key for Parameter Store SecureStrings"
  deletion_window_in_days = 7
}

resource "aws_ssm_parameter" "db_url" {
  name  = "/dev/payment-service/db/url"
  type  = "String"
  value = "jdbc:mysql://dev-db.local:3306/payments"
}

resource "aws_ssm_parameter" "db_password" {
  name   = "/dev/payment-service/db/password"
  type   = "SecureString"
  key_id = aws_kms_key.ssm_key.arn
  value  = "SuperSecretPassword123!"
}
```

**Validation:** `terraform state list` should show all three resources; `aws ssm get-parameter --name "/dev/payment-service/db/url"` should confirm it exists in AWS.

---

## Lab 6 — Lock down access with least-privilege IAM

**Objective:** confirm a scoped role can read `dev` but not `prod`.

**Steps:**
1. Create the policy (save as `dev-readonly-policy.json`):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ssm:GetParameter", "ssm:GetParameters", "ssm:GetParametersByPath"],
      "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/dev/payment-service/*"
    },
    { "Effect": "Allow", "Action": ["kms:Decrypt"], "Resource": "arn:aws:kms:us-east-1:123456789012:key/your-custom-kms-key-id" }
  ]
}
```
2. Attach it to a test role, assume that role, and try reading both `/dev/payment-service/db/url` and a `/prod/...` parameter.

**Validation:** The `/dev/...` read succeeds; the `/prod/...` read returns `AccessDeniedException`.

---

## Lab 7 — Container entrypoint injection

**Objective:** feed SSM parameters into a legacy app that only reads plain env vars.

**Steps:** Save as `entrypoint.sh`:
```bash
#!/bin/bash
PARAM_JSON=$(aws ssm get-parameters-by-path --path "/dev/payment-service/db/" --recursive --with-decryption --region us-east-1)
export DB_USER=$(echo $PARAM_JSON | jq -r '.Parameters[] | select(.Name | endswith("/username")) | .Value')
export DB_PASS=$(echo $PARAM_JSON | jq -r '.Parameters[] | select(.Name | endswith("/password")) | .Value')
exec "$@"
```
Make it executable and use it as your Docker `ENTRYPOINT`, then run `env | grep DB_` inside the container.

**Validation:** `DB_USER` and `DB_PASS` show up as real environment variables inside the running container.

---

## Lab 8 — Create your first Secrets Manager secret

**Objective:** store a full credential bundle as one JSON secret.

**Steps:**
```bash
aws secretsmanager create-secret \
    --name "prod/myapp/database" \
    --description "Production MySQL Database Credentials" \
    --secret-string '{"username":"admin_user","password":"SuperSecurePassword2026!","host":"prod-db.internal","port":"3306"}'
```

**Validation:**
```bash
aws secretsmanager get-secret-value --secret-id "prod/myapp/database"
```
The response should include a `SecretString` field containing your escaped JSON.

---

## Lab 9 — Parse a secret in Python

**Objective:** load structured credentials into a native dict at runtime.

**Steps:** Save as `get_secret.py`:
```python
import boto3, json
from botocore.exceptions import ClientError

def get_secret():
    client = boto3.session.Session().client('secretsmanager', region_name='us-east-1')
    try:
        response = client.get_secret_value(SecretId="prod/myapp/database")
    except ClientError as e:
        raise e
    return json.loads(response['SecretString'])

if __name__ == "__main__":
    creds = get_secret()
    print(f"Connecting to {creds['host']} as {creds['username']}")
```

**Validation:** Script prints the host and username without a `KeyError` or `json.JSONDecodeError`.

---

## Lab 10 — Update a secret and inspect version history

**Objective:** see how `AWSCURRENT` / `AWSPREVIOUS` shift after an update.

**Steps:**
```bash
aws secretsmanager put-secret-value \
    --secret-id "prod/myapp/database" \
    --secret-string '{"username":"admin_user","password":"NEW_Stronger_Password_2026#","host":"prod-db.internal","port":"3306"}'

aws secretsmanager list-secret-version-ids --secret-id "prod/myapp/database"
```

**Validation:** The output shows two version IDs — one tagged `AWSCURRENT` (the new password) and one tagged `AWSPREVIOUS` (the old one).

---

## Lab 11 — Provision Secrets Manager with Terraform

**Objective:** codify the secret container and its initial payload separately.

**Steps:**
```hcl
resource "aws_kms_key" "secrets_key" {
  description             = "KMS key for encrypting production secrets"
  deletion_window_in_days = 7
}

resource "aws_secretsmanager_secret" "db_secret" {
  name       = "prod/payment-service/rds-mysql"
  kms_key_id = aws_kms_key.secrets_key.arn
}

resource "aws_secretsmanager_secret_version" "db_secret_val" {
  secret_id = aws_secretsmanager_secret.db_secret.id
  secret_string = jsonencode({
    username = "db_admin"
    password = "InitialSecurePasswordPassword123!"
    engine   = "mysql"
    host     = "prod-db.cluster-c123.us-east-1.rds.amazonaws.com"
    port     = "3306"
  })
}
```

**Validation:** `terraform apply` succeeds and `aws secretsmanager describe-secret --secret-id prod/payment-service/rds-mysql` returns the resource with your KMS key ARN attached.

---

## Lab 12 — Wire up automatic rotation (conceptual walkthrough)

**Objective:** understand the rotation lifecycle end to end, even without deploying a full rotation Lambda.

**Steps:**
1. Review the rotation sequence in [`README.md` §2.5](./README.md#25-automated-rotation) — Trigger → Create → Test → Commit.
2. Enable rotation on the Lab 8 secret, pointing at a rotation Lambda ARN (use one of AWS's [RDS rotation templates](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotate-secrets_lambda-functions.html) if you have an RDS instance handy):
```bash
aws secretsmanager rotate-secret \
    --secret-id "prod/myapp/database" \
    --rotation-lambda-arn "arn:aws:lambda:us-east-1:123456789012:function:rotate-mysql-secret" \
    --rotation-rules AutomaticallyAfterDays=30
```
3. Force one rotation cycle to test the wiring: `aws secretsmanager rotate-secret --secret-id "prod/myapp/database"`

**Validation:** `list-secret-version-ids` shows a new `AWSPENDING` version appear, then flip to `AWSCURRENT` once the Lambda completes successfully. If it fails, see [`troubleshooting.md`](./troubleshooting.md#rotation-failures).

---

## What's next

- Break these labs into your own CI pipeline (GitHub Actions / CodePipeline) so parameters and secrets are provisioned automatically per environment.
- Add EventBridge + a Lambda notifier for change auditing (see README §1.7).
- If something breaks along the way, check [`troubleshooting.md`](./troubleshooting.md) before re-running a lab.
