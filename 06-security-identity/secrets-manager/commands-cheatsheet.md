# Commands Cheatsheet

Copy-paste reference for every CLI command used across the SSM Parameter Store and Secrets Manager workflows. See [`README.md`](./README.md) for the concepts behind each one, and [`hands-on-labs.md`](./hands-on-labs.md) to run them as guided exercises.

---

## SSM Parameter Store

### Create / Write

```bash
# Standard string parameter
aws ssm put-parameter \
    --name "/dev/myapp/db_url" \
    --value "jdbc:mysql://dev-db.cluster.internal:3306/mydb" \
    --type "String" \
    --overwrite

# Encrypted SecureString parameter
aws ssm put-parameter \
    --name "/dev/myapp/db_password" \
    --value "SuperSecretPassword123!" \
    --type "SecureString" \
    --overwrite

# SecureString with a customer-managed KMS key
aws ssm put-parameter \
    --name "/dev/myapp/db_password" \
    --value "SuperSecretPassword123!" \
    --type "SecureString" \
    --key-id "alias/ssm-key" \
    --overwrite
```

### Read — single parameter

```bash
# Standard string
aws ssm get-parameter --name "/dev/myapp/db_url"

# SecureString WITHOUT decryption -> returns ciphertext
aws ssm get-parameter --name "/dev/myapp/db_password"

# SecureString WITH decryption -> returns cleartext
aws ssm get-parameter --name "/dev/myapp/db_password" --with-decryption
```

### Read — bulk path (recommended for apps)

```bash
aws ssm get-parameters-by-path \
    --path "/dev/myapp/" \
    --recursive \
    --with-decryption
```

### Versioning

```bash
# Every --overwrite auto-increments the version
aws ssm put-parameter --name "/dev/myapp/db_password" --value "NewPassword456!" --type "SecureString" --overwrite

# Fetch a specific historical version
aws ssm get-parameter --name "/dev/myapp/db_password:1" --with-decryption

# List version history
aws ssm get-parameter-history --name "/dev/myapp/db_password" --with-decryption
```

### Delete / cleanup

```bash
aws ssm delete-parameter --name "/dev/myapp/db_url"

# Delete an entire path's worth of parameters (loop)
for p in $(aws ssm get-parameters-by-path --path "/dev/myapp/" --recursive --query "Parameters[].Name" --output text); do
  aws ssm delete-parameter --name "$p"
done
```

### Throughput

```bash
# Check / raise the throughput tier (console-driven, but can be scripted via)
aws ssm update-service-setting \
    --setting-id "arn:aws:ssm:us-east-1:123456789012:servicesetting/ssm/parameter-store/high-throughput-enabled" \
    --setting-value "true"
```

---

## AWS Secrets Manager

### Create

```bash
aws secretsmanager create-secret \
    --name "prod/myapp/database" \
    --description "Production MySQL Database Credentials" \
    --secret-string '{"username":"admin_user","password":"SuperSecurePassword2026!","host":"prod-db.internal","port":"3306"}'
```

### Read

```bash
aws secretsmanager get-secret-value --secret-id "prod/myapp/database"

# Pull just the decoded value with jq
aws secretsmanager get-secret-value --secret-id "prod/myapp/database" \
  | jq -r '.SecretString | fromjson'
```

### Update

```bash
aws secretsmanager put-secret-value \
    --secret-id "prod/myapp/database" \
    --secret-string '{"username":"admin_user","password":"NEW_Stronger_Password_2026#","host":"prod-db.internal","port":"3306"}'
```

### Versioning & staging labels

```bash
# List version history and staging labels (AWSCURRENT / AWSPREVIOUS / AWSPENDING)
aws secretsmanager list-secret-version-ids --secret-id "prod/myapp/database"

# Fetch a specific staged version
aws secretsmanager get-secret-value --secret-id "prod/myapp/database" --version-stage AWSPREVIOUS
```

### Rotation

```bash
# Enable rotation with a Lambda rotation function, every 30 days
aws secretsmanager rotate-secret \
    --secret-id "prod/myapp/database" \
    --rotation-lambda-arn "arn:aws:lambda:us-east-1:123456789012:function:rotate-mysql-secret" \
    --rotation-rules AutomaticallyAfterDays=30

# Force an immediate rotation (testing)
aws secretsmanager rotate-secret --secret-id "prod/myapp/database"
```

### Delete / cleanup

```bash
# Soft delete with a 7-day recovery window (minimum)
aws secretsmanager delete-secret --secret-id "prod/myapp/database" --recovery-window-in-days 7

# Force-delete immediately, no recovery window
aws secretsmanager delete-secret --secret-id "prod/myapp/database" --force-delete-without-recovery
```

---

## Terraform quick reference

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan

# Target just the secrets-related resources
terraform apply -target=aws_secretsmanager_secret.db_secret

# Import an existing parameter/secret into state
terraform import aws_ssm_parameter.db_password /dev/payment-service/db/password
terraform import aws_secretsmanager_secret.db_secret prod/payment-service/rds-mysql
```

---

## jq snippets (parsing responses)

```bash
# Pull one value out of a get-parameters-by-path response
echo "$PARAM_JSON" | jq -r '.Parameters[] | select(.Name | endswith("/username")) | .Value'

# Turn a get-secret-value response into flat env-var export lines
aws secretsmanager get-secret-value --secret-id "prod/myapp/database" \
  | jq -r '.SecretString | fromjson | to_entries[] | "export \(.key | ascii_upcase)=\(.value)"'
```

---

## Quick lookup table

| I want to... | Parameter Store | Secrets Manager |
|---|---|---|
| Create | `put-parameter --overwrite` | `create-secret` |
| Read one | `get-parameter --with-decryption` | `get-secret-value` |
| Read many | `get-parameters-by-path --recursive --with-decryption` | (loop `get-secret-value`, or use `batch-get-secret-value`) |
| Update | `put-parameter --overwrite` | `put-secret-value` |
| History | `get-parameter-history` | `list-secret-version-ids` |
| Delete | `delete-parameter` | `delete-secret` |
