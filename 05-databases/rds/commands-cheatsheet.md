# AWS RDS — Commands Cheatsheet

Copy-paste reference: AWS CLI + Terraform for every concept in [`README.md`](./README.md). Replace placeholder values (`<region>`, `<db-identifier>`, passwords, ARNs) before running.

## Table of Contents
- [1. Instance Creation](#1-instance-creation)
- [2. Multi-AZ](#2-multi-az)
- [3. Read Replicas](#3-read-replicas)
- [4. Backups & Restore](#4-backups--restore)
- [5. Point-in-Time Recovery](#5-point-in-time-recovery)
- [6. Parameter Groups](#6-parameter-groups)
- [7. Security & Encryption](#7-security--encryption)
- [8. IAM Database Authentication](#8-iam-database-authentication)
- [9. RDS Proxy](#9-rds-proxy)
- [10. Monitoring](#10-monitoring)
- [11. Upgrades](#11-upgrades)
- [12. Storage](#12-storage)

---

## 1. Instance Creation

```bash
aws rds create-db-instance \
  --db-instance-identifier prod-mysql-db \
  --db-instance-class db.t3.medium \
  --engine mysql \
  --engine-version 8.0.35 \
  --master-username admin \
  --master-user-password '<use-secrets-manager-instead>' \
  --allocated-storage 50 \
  --storage-type gp3 \
  --vpc-security-group-ids sg-0123456789abcdef0 \
  --db-subnet-group-name private-db-subnet-group \
  --no-publicly-accessible \
  --backup-retention-period 7 \
  --region ap-south-1
```

```hcl
resource "aws_db_instance" "prod_mysql" {
  identifier             = "prod-mysql-db"
  engine                 = "mysql"
  engine_version         = "8.0.35"
  instance_class         = "db.t3.medium"
  allocated_storage      = 50
  storage_type           = "gp3"
  username               = "admin"
  manage_master_user_password = true   # lets AWS manage the secret in Secrets Manager
  db_subnet_group_name   = aws_db_subnet_group.private.name
  vpc_security_group_ids = [aws_security_group.db_sg.id]
  publicly_accessible    = false
  backup_retention_period = 7
  storage_encrypted      = true         # must be set at creation — cannot be added later
}
```

---

## 2. Multi-AZ

```bash
# Enable Multi-AZ on an existing instance
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-db \
  --multi-az \
  --apply-immediately

# Create a Multi-AZ DB CLUSTER (1 writer + 2 readable standbys)
aws rds create-db-cluster \
  --db-cluster-identifier prod-mysql-cluster \
  --engine mysql \
  --engine-version 8.0.35 \
  --master-username admin \
  --master-user-password '<use-secrets-manager>' \
  --db-cluster-instance-class db.r6g.large \
  --storage-type io1 \
  --iops 3000 \
  --allocated-storage 100
```

```hcl
resource "aws_db_instance" "prod_mysql" {
  # ...
  multi_az = true
}
```

---

## 3. Read Replicas

```bash
# Same-region replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-mysql-replica-1 \
  --source-db-instance-identifier prod-mysql-db \
  --db-instance-class db.t3.small

# Cross-region replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-mysql-replica-us \
  --source-db-instance-identifier arn:aws:rds:ap-south-1:123456789012:db:prod-mysql-db \
  --db-instance-class db.t3.small \
  --region us-east-1

# Promote a replica to standalone read/write primary
aws rds promote-read-replica \
  --db-instance-identifier prod-mysql-replica-1
```

```hcl
resource "aws_db_instance" "read_replica" {
  identifier          = "prod-mysql-replica-1"
  replicate_source_db = aws_db_instance.prod_mysql.identifier
  instance_class      = "db.t3.small"   # can differ from source
}
```

---

## 4. Backups & Restore

```bash
# Set automated backup retention (1-35 days)
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-db \
  --backup-retention-period 14 \
  --preferred-backup-window "02:00-02:30"

# Take a manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier prod-mysql-db \
  --db-snapshot-identifier pre-migration-snapshot-2026-07-10

# Restore from a manual snapshot (always creates a NEW instance)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-mysql-restored \
  --db-snapshot-identifier pre-migration-snapshot-2026-07-10 \
  --db-instance-class db.t3.medium

# List snapshots
aws rds describe-db-snapshots --db-instance-identifier prod-mysql-db
```

```hcl
resource "aws_db_instance" "prod_mysql" {
  # ...
  backup_retention_period = 14
  backup_window            = "02:00-02:30"
}
```

---

## 5. Point-in-Time Recovery

```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-postgres-db \
  --target-db-instance-identifier prod-postgres-recovered \
  --restore-time "2026-07-10T14:05:21.000Z" \
  --db-instance-class db.t3.medium

# Restore to the latest possible point (up to 5 min ago)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-postgres-db \
  --target-db-instance-identifier prod-postgres-recovered \
  --use-latest-restorable-time
```

---

## 6. Parameter Groups

```bash
# Create a custom parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name production-mysql8-pg \
  --db-parameter-group-family mysql8.0 \
  --description "Custom performance configuration"

# Modify a dynamic parameter (no downtime)
aws rds modify-db-parameter-group \
  --db-parameter-group-name production-mysql8-pg \
  --parameters "ParameterName=max_connections,ParameterValue=150,ApplyMethod=immediate"

# Modify a static parameter (pending-reboot)
aws rds modify-db-parameter-group \
  --db-parameter-group-name production-mysql8-pg \
  --parameters "ParameterName=innodb_buffer_pool_size,ParameterValue=4294967296,ApplyMethod=pending-reboot"

# Attach the parameter group to an instance (forces pending-reboot)
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-db \
  --db-parameter-group-name production-mysql8-pg

# Reboot to apply static changes
aws rds reboot-db-instance --db-instance-identifier prod-mysql-db
```

```hcl
resource "aws_db_parameter_group" "custom_mysql_pg" {
  name   = "production-mysql8-pg"
  family = "mysql8.0"

  parameter {
    name  = "max_connections"
    value = "150"
  }

  parameter {
    name         = "innodb_buffer_pool_size"
    value        = "4294967296"
    apply_method = "pending-reboot"
  }
}
```

---

## 7. Security & Encryption

```bash
# Encrypt an existing unencrypted DB (copy-snapshot-then-restore pattern)
aws rds create-db-snapshot \
  --db-instance-identifier legacy-unencrypted-db \
  --db-snapshot-identifier legacy-snapshot

aws rds copy-db-snapshot \
  --source-db-snapshot-identifier legacy-snapshot \
  --target-db-snapshot-identifier legacy-snapshot-encrypted \
  --kms-key-id alias/aws/rds

aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier legacy-db-encrypted \
  --db-snapshot-identifier legacy-snapshot-encrypted \
  --db-instance-class db.t3.medium

# Enforce SSL in a parameter group (PostgreSQL)
aws rds modify-db-parameter-group \
  --db-parameter-group-name production-postgres15-pg \
  --parameters "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate"
```

```hcl
resource "aws_security_group" "db_sg" {
  name   = "db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app_sg.id]   # ONLY the app tier, never 0.0.0.0/0
  }
}
```

---

## 8. IAM Database Authentication

```bash
# Enable on the instance
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-db \
  --enable-iam-database-authentication \
  --apply-immediately

# Generate a connection token (valid 15 min)
aws rds generate-db-auth-token \
  --hostname prod-mysql-db.abcdefg.ap-south-1.rds.amazonaws.com \
  --port 3306 \
  --username app_iam_user \
  --region ap-south-1
```

```sql
-- PostgreSQL: map the DB user to IAM
CREATE USER app_iam_user;
GRANT rds_iam TO app_iam_user;

-- MySQL / MariaDB
CREATE USER 'app_iam_user'@'%' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';
```

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "rds-db:connect",
    "Resource": "arn:aws:rds-db:ap-south-1:123456789012:dbuser:dbi-12345678/app_iam_user"
  }]
}
```

---

## 9. RDS Proxy

```bash
aws rds create-db-proxy \
  --db-proxy-name prod-mysql-proxy \
  --engine-family MYSQL \
  --auth '[{"AuthScheme":"SECRETS","SecretArn":"arn:aws:secretsmanager:ap-south-1:123456789012:secret:rds-creds","IAMAuth":"REQUIRED"}]' \
  --role-arn arn:aws:iam::123456789012:role/rds-proxy-role \
  --vpc-subnet-ids subnet-abc123 subnet-def456 \
  --vpc-security-group-ids sg-0123456789abcdef0

aws rds register-db-proxy-targets \
  --db-proxy-name prod-mysql-proxy \
  --db-instance-identifiers prod-mysql-db
```

```hcl
resource "aws_db_proxy" "prod_proxy" {
  name                   = "prod-mysql-proxy"
  engine_family          = "MYSQL"
  role_arn               = aws_iam_role.proxy_role.arn
  vpc_subnet_ids         = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  vpc_security_group_ids = [aws_security_group.db_sg.id]

  auth {
    auth_scheme = "SECRETS"
    secret_arn  = aws_secretsmanager_secret.db_creds.arn
    iam_auth    = "REQUIRED"
  }
}

resource "aws_db_proxy_default_target_group" "default" {
  db_proxy_name = aws_db_proxy.prod_proxy.name
}

resource "aws_db_proxy_target" "prod_target" {
  db_proxy_name          = aws_db_proxy.prod_proxy.name
  target_group_name      = aws_db_proxy_default_target_group.default.name
  db_instance_identifier = aws_db_instance.prod_mysql.identifier
}
```

---

## 10. Monitoring

```bash
# Enable Enhanced Monitoring (1-second granularity)
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-db \
  --monitoring-interval 1 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role

# Enable Performance Insights
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-db \
  --enable-performance-insights \
  --performance-insights-retention-period 7

# Pull a CloudWatch metric
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=prod-mysql-db \
  --start-time 2026-07-10T00:00:00Z \
  --end-time 2026-07-10T23:59:59Z \
  --period 300 \
  --statistics Average Maximum
```

---

## 11. Upgrades

```bash
# Minor version upgrade (safe, in maintenance window)
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-db \
  --engine-version 8.0.36 \
  --auto-minor-version-upgrade \
  --apply-immediately

# Create a Blue/Green Deployment for a major version upgrade
aws rds create-blue-green-deployment \
  --blue-green-deployment-name pg14-to-pg15-upgrade \
  --source arn:aws:rds:ap-south-1:123456789012:db:prod-postgres-db \
  --target-engine-version 15.5 \
  --target-db-parameter-group-name production-postgres15-pg

# Switch traffic to Green once validated
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier bgd-abc123
```

---

## 12. Storage

```bash
# Enable storage autoscaling (set max threshold)
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-db \
  --max-allocated-storage 500

# Change storage type
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-db \
  --storage-type io2 \
  --iops 5000 \
  --apply-immediately
```
