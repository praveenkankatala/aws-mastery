# AWS Migration — Commands Cheat Sheet

Every command you're likely to need during a migration, grouped so you can find it fast. Placeholders look like `<this>` — replace them.

> **Two habits that prevent most mistakes:**
> ```bash
> export AWS_REGION=ap-south-1          # set it once, in every shell
> export AWS_PAGER=""                   # stop the CLI opening a pager on every call
> aws sts get-caller-identity           # always confirm WHO and WHICH ACCOUNT first
> ```

---

## Contents

- [0. Setup and sanity checks](#0-setup-and-sanity-checks)
- [1. Application Discovery Service](#1-application-discovery-service-ads)
- [2. Migration Hub](#2-migration-hub)
- [3. Application Migration Service (MGN) — Rehost](#3-application-migration-service-mgn--rehost)
- [4. Elastic Disaster Recovery (DRS)](#4-elastic-disaster-recovery-drs)
- [5. VM Import / Export](#5-vm-import--export)
- [6. Database Migration Service (DMS)](#6-database-migration-service-dms)
- [7. Schema Conversion (SCT / DMS SC)](#7-schema-conversion-sct--dms-sc)
- [8. Native database dump and restore](#8-native-database-dump-and-restore)
- [9. RDS and Aurora](#9-rds-and-aurora)
- [10. DataSync](#10-datasync)
- [11. S3 data movement](#11-s3-data-movement)
- [12. Snow Family](#12-snow-family)
- [13. FSx, EFS and Storage Gateway](#13-fsx-efs-and-storage-gateway)
- [14. Transfer Family](#14-transfer-family)
- [15. Networking and connectivity](#15-networking-and-connectivity)
- [16. Route 53 and DNS cutover](#16-route-53-and-dns-cutover)
- [17. EC2, EBS and AMIs](#17-ec2-ebs-and-amis)
- [18. Load balancers and target groups](#18-load-balancers-and-target-groups)
- [19. Systems Manager](#19-systems-manager)
- [20. App2Container and containers](#20-app2container-and-containers)
- [21. Refactor Spaces](#21-refactor-spaces)
- [22. Backup, DR and snapshots](#22-backup-dr-and-snapshots)
- [23. Cost, sizing and optimization](#23-cost-sizing-and-optimization)
- [24. Governance, quotas and tagging](#24-governance-quotas-and-tagging)
- [25. Source-side commands (Linux)](#25-source-side-commands-linux)
- [26. Source-side commands (Windows / PowerShell)](#26-source-side-commands-windows--powershell)
- [27. Validation one-liners](#27-validation-one-liners)
- [28. Cleanup and decommission](#28-cleanup-and-decommission)

---

## 0. Setup and sanity checks

```bash
# Install / verify AWS CLI v2 (Linux x86_64)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip && sudo ./aws/install --update
aws --version

# Configure credentials
aws configure                                    # long-lived keys (sandbox only)
aws configure sso                                # IAM Identity Center (preferred)
aws configure list-profiles
export AWS_PROFILE=migration-sandbox

# Who am I, where am I
aws sts get-caller-identity
aws configure get region
aws ec2 describe-regions --query 'Regions[].RegionName' --output table

# Session Manager plugin (shell into EC2 without SSH/bastion)
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o sm.deb
sudo dpkg -i sm.deb && session-manager-plugin --version

# Assume a cross-account migration role
aws sts assume-role \
  --role-arn arn:aws:iam::<target-account-id>:role/MigrationEngineer \
  --role-session name-wave1 \
  --query 'Credentials' --output json
```

---

## 1. Application Discovery Service (ADS)

```bash
# Turn on continuous export of discovery data to S3/Athena
aws discovery start-continuous-export
aws discovery describe-continuous-exports

# What has been discovered?
aws discovery describe-agents                       # agent + collector health
aws discovery list-configurations --configuration-type SERVER
aws discovery list-configurations --configuration-type SERVER \
  --filters name=server.osName,values=Linux,condition=CONTAINS

# Detail on one server (get configurationId from list-configurations)
aws discovery describe-configurations --configuration-ids <d-server-xxxx>

# Utilization / process / connection summary
aws discovery get-discovery-summary

# Start / stop data collection on specific agents
aws discovery start-data-collection-by-agent-ids --agent-ids <agentId1> <agentId2>
aws discovery stop-data-collection-by-agent-ids  --agent-ids <agentId1>

# Export detailed data (agent-collected) for offline analysis
aws discovery start-export-task \
  --export-data-format CSV \
  --filters name=agentId,values=<agentId>,condition=EQUALS
aws discovery describe-export-tasks

# Tag servers with application / wave metadata
aws discovery create-tags \
  --configuration-ids <d-server-xxxx> \
  --tags key=Wave,value=1 key=Application,value=orders-api

# Group servers into an application
aws discovery create-application --name "orders-api" --description "Wave 1"
aws discovery associate-configuration-items-to-application \
  --application-configuration-id <app-id> \
  --configuration-ids <d-server-1> <d-server-2>
```

**Agent install (source servers):**

```bash
# Linux discovery agent
curl -o ./aws-discovery-agent.tar.gz \
  https://s3.<region>.amazonaws.com/aws-discovery-agent.<region>/linux/latest/aws-discovery-agent.tar.gz
tar -xzf aws-discovery-agent.tar.gz
sudo bash install -r <region> -k <ACCESS_KEY> -s <SECRET_KEY>
sudo systemctl status aws-discovery-daemon
```

```powershell
# Windows discovery agent (silent install)
.\AWSDiscoveryAgentInstaller.exe REGION=<region> KEY_ID=<ACCESS_KEY> KEY_SECRET=<SECRET> /quiet
Get-Service AWSDiscoveryAgent
```

**Athena queries over exported discovery data** (database `application_discovery_service_database`):

```sql
-- Idle candidates for RETIRE
SELECT server_hostname, avg_cpu_usage_pct, max_cpu_usage_pct, total_disk_free_gb
FROM   os_info_agent
WHERE  max_cpu_usage_pct < 5
ORDER  BY max_cpu_usage_pct;

-- Dependency edges (who talks to whom)
SELECT source_server_hostname, destination_server_hostname,
       destination_port, COUNT(*) AS conns
FROM   network_interface_agent
GROUP  BY 1,2,3
ORDER  BY conns DESC
LIMIT  100;

-- Right-sizing input: p95 utilization
SELECT server_hostname,
       approx_percentile(avg_cpu_usage_pct, 0.95) AS p95_cpu,
       approx_percentile(avg_free_ram_gb,   0.05) AS p05_free_ram
FROM   os_info_agent
GROUP  BY server_hostname;
```

---

## 2. Migration Hub

```bash
# Set / read the home region (do this once, before anything else)
aws migrationhub-config create-home-region-control \
  --home-region <region> \
  --target Type=ACCOUNT,Id=<account-id>
aws migrationhub-config get-home-region

# Progress tracking
aws mgh list-progress-update-streams
aws mgh create-progress-update-stream --progress-update-stream-name Wave1
aws mgh list-migration-tasks --progress-update-stream Wave1
aws mgh describe-migration-task \
  --progress-update-stream Wave1 --migration-task-name <task>

aws mgh notify-migration-task-state \
  --progress-update-stream Wave1 \
  --migration-task-name <task> \
  --task '{"Status":"IN_PROGRESS","ProgressPercent":40}' \
  --update-date-time <iso8601> --next-update-seconds 3600

aws mgh notify-application-state \
  --application-id <app-id> --status COMPLETED

# Associate a discovered server with a migration task
aws mgh associate-discovered-resource \
  --progress-update-stream Wave1 --migration-task-name <task> \
  --discovered-resource ConfigurationId=<d-server-xxxx>

# Strategy Recommendations (get an R suggestion per app)
aws migrationhubstrategy get-portfolio-summary
aws migrationhubstrategy start-assessment
aws migrationhubstrategy get-assessment --id <assessment-id>
aws migrationhubstrategy list-application-components
aws migrationhubstrategy get-application-component-strategies \
  --application-component-id <id>
aws migrationhubstrategy list-servers
aws migrationhubstrategy get-server-strategies --server-id <id>

# Migration Hub Orchestrator
aws migrationhuborchestrator list-templates
aws migrationhuborchestrator create-workflow \
  --name rehost-wave1 --template-id <template-id> \
  --input-parameters '{}'
aws migrationhuborchestrator start-workflow --id <workflow-id>
aws migrationhuborchestrator list-workflow-steps --workflow-id <id> --step-group-id <id>
```

---

## 3. Application Migration Service (MGN) — Rehost

### Initialize and configure

```bash
# One-time per account+region: creates roles and default templates
aws mgn initialize-service

# Replication settings template (the staging-area config)
aws mgn describe-replication-configuration-templates
aws mgn update-replication-configuration-template \
  --replication-configuration-template-id <rct-id> \
  --staging-area-subnet-id subnet-<staging> \
  --replication-server-instance-type t3.small \
  --default-large-staging-disk-type GP3 \
  --ebs-encryption DEFAULT \
  --use-dedicated-replication-server false \
  --create-public-ip false \
  --data-plane-routing PRIVATE_IP \
  --bandwidth-throttling 100 \
  --associate-default-security-group true \
  --staging-area-tags Wave=1

# Launch configuration template (what the target EC2 looks like)
aws mgn describe-launch-configuration-templates
aws mgn update-launch-configuration-template \
  --launch-configuration-template-id <lct-id> \
  --launch-disposition STARTED \
  --target-instance-type-right-sizing-method BASIC \
  --copy-tags true --copy-private-ip false \
  --boot-mode LEGACY_BIOS \
  --enable-map-auto-tagging true
```

### Install the replication agent on source servers

```bash
# Linux source
wget -O ./aws-replication-installer-init \
  https://aws-application-migration-service-<region>.s3.<region>.amazonaws.com/latest/linux/aws-replication-installer-init
sudo chmod +x aws-replication-installer-init
sudo ./aws-replication-installer-init \
  --region <region> \
  --aws-access-key-id <AKIA...> \
  --aws-secret-access-key <SECRET> \
  --no-prompt

# Only replicate specific disks
sudo ./aws-replication-installer-init --region <region> --devices /dev/sda,/dev/sdb --no-prompt

# Agent service checks
sudo systemctl status aws-replication-agent
sudo tail -f /var/lib/aws-replication-agent/agent.log.0
```

```powershell
# Windows source
Invoke-WebRequest -Uri "https://aws-application-migration-service-<region>.s3.<region>.amazonaws.com/latest/windows/AwsReplicationWindowsInstaller.exe" `
  -OutFile "C:\Temp\AwsReplicationWindowsInstaller.exe"
C:\Temp\AwsReplicationWindowsInstaller.exe --region <region> `
  --aws-access-key-id <AKIA...> --aws-secret-access-key <SECRET> --no-prompt

Get-Service AWSReplicationAgent
Get-Content "C:\Program Files (x86)\AWS Replication Agent\agent.log.0" -Tail 50
```

### Track, test, cut over

```bash
# List source servers and their state
aws mgn describe-source-servers \
  --query 'items[].{ID:sourceServerID,Host:sourceProperties.identificationHints.hostname,State:dataReplicationInfo.dataReplicationState,Lag:dataReplicationInfo.lagDuration,Life:lifeCycle.state}' \
  --output table

# Watch replication progress for one server
aws mgn describe-source-servers --filters sourceServerIDs=<s-id> \
  --query 'items[0].dataReplicationInfo'

# Per-server overrides
aws mgn update-launch-configuration \
  --source-server-id <s-id> \
  --target-instance-type-right-sizing-method NONE \
  --launch-disposition STARTED --copy-tags true

aws mgn get-launch-configuration --source-server-id <s-id>

# Tag the source server (tags flow to the launched instance if copy-tags on)
aws mgn tag-resource --resource-arn <sourceServerArn> \
  --tags Application=orders-api,Wave=1,Environment=prod

# TEST launch — safe, non-disruptive, repeatable
aws mgn start-test --source-server-id <s-id>
aws mgn describe-job-log-items --job-id <job-id>
aws mgn describe-jobs --filters jobIDs=<job-id>

# Mark test as complete / ready for cutover
aws mgn finalize-cutover --source-server-id <s-id>     # (after cutover, see below)
aws mgn change-server-life-cycle-state \
  --source-server-id <s-id> --life-cycle '{"state":"READY_FOR_CUTOVER"}'

# CUTOVER launch
aws mgn start-cutover --source-server-ids <s-id-1> <s-id-2>

# Confirm and clean up staging resources (stops replication billing)
aws mgn finalize-cutover --source-server-id <s-id>

# Roll back a cutover / test (terminates launched instances, keeps replication)
aws mgn terminate-target-instances --source-server-ids <s-id>
aws mgn start-replication --source-server-id <s-id>       # resume if paused
aws mgn pause-replication --source-server-id <s-id>
aws mgn resume-replication --source-server-id <s-id>
aws mgn retry-data-replication --source-server-id <s-id>

# Disconnect / remove a server from the service
aws mgn disconnect-from-service --source-server-id <s-id>
aws mgn delete-source-server --source-server-id <s-id>

# Post-launch actions (run scripts automatically on launch)
aws mgn list-template-actions --launch-configuration-template-id <lct-id>
aws mgn put-template-action \
  --launch-configuration-template-id <lct-id> \
  --action-id install-cw-agent --action-name "Install CloudWatch Agent" \
  --document-identifier AWS-ConfigureAWSPackage \
  --order 1001 --active true --category MONITORING_AND_OBSERVABILITY \
  --parameters '{"action":[{"type":"STRING","value":"Install"}],"name":[{"type":"STRING","value":"AmazonCloudWatchAgent"}]}'

aws mgn list-source-server-actions --source-server-id <s-id>
```

**Handy watch loop during a wave:**

```bash
watch -n 30 'aws mgn describe-source-servers \
  --query "items[].{H:sourceProperties.identificationHints.hostname,\
S:dataReplicationInfo.dataReplicationState,\
P:dataReplicationInfo.replicatedDisks[0].backloggedStorageBytes,\
L:lifeCycle.state}" --output table'
```

---

## 4. Elastic Disaster Recovery (DRS)

```bash
aws drs initialize-service
aws drs describe-source-servers
aws drs describe-replication-configuration-templates
aws drs update-replication-configuration \
  --source-server-id <s-id> --staging-area-subnet-id subnet-<id> \
  --pit-policy '[{"enabled":true,"interval":10,"retentionDuration":60,"units":"MINUTE","ruleID":1}]'

aws drs describe-recovery-snapshots --source-server-id <s-id>
aws drs start-recovery --source-servers sourceServerID=<s-id>        # drill or real
aws drs start-failback-launch --source-server-ids <s-id>             # go back on-prem
aws drs stop-failback --recovery-instance-id <r-id>
aws drs terminate-recovery-instances --recovery-instance-ids <r-id>
aws drs disconnect-recovery-instance --recovery-instance-id <r-id>
```

---

## 5. VM Import / Export

```bash
# 1) Upload the OVA/VMDK/VHD to S3
aws s3 cp ./legacy-app.ova s3://<import-bucket>/ova/legacy-app.ova

# 2) Trust policy + role (vmimport) — required once
cat > trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"vmie.amazonaws.com"},"Action":"sts:AssumeRole",
 "Condition":{"StringEquals":{"sts:Externalid":"vmimport"}}}]}
EOF
aws iam create-role --role-name vmimport --assume-role-policy-document file://trust.json
aws iam put-role-policy --role-name vmimport --policy-name vmimport \
  --policy-document file://role-policy.json     # s3:Get*, ec2:CopySnapshot, RegisterImage, etc.

# 3) Import
cat > containers.json <<'EOF'
[{"Description":"legacy-app","Format":"ova",
  "UserBucket":{"S3Bucket":"<import-bucket>","S3Key":"ova/legacy-app.ova"}}]
EOF
aws ec2 import-image --description "legacy-app" --disk-containers file://containers.json

aws ec2 describe-import-image-tasks --import-task-ids import-ami-<id>
aws ec2 cancel-import-task --import-task-id import-ami-<id>

# Import just a disk as an EBS snapshot
aws ec2 import-snapshot --description "data-disk" --disk-container file://disk.json
aws ec2 describe-import-snapshot-tasks

# Export an instance back out (for rollback / archival)
aws ec2 create-instance-export-task --instance-id i-<id> \
  --target-environment vmware \
  --export-to-s3-task DiskImageFormat=VMDK,ContainerFormat=ova,S3Bucket=<bucket>,S3Prefix=exports/
```

---

## 6. Database Migration Service (DMS)

### Prerequisite roles

```bash
for R in dms-vpc-role dms-cloudwatch-logs-role dms-access-for-endpoint; do
  aws iam create-role --role-name $R --assume-role-policy-document \
    '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"dms.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
done
aws iam attach-role-policy --role-name dms-vpc-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSVPCManagementRole
aws iam attach-role-policy --role-name dms-cloudwatch-logs-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSCloudWatchLogsRole
```

### Replication instance

```bash
aws dms create-replication-subnet-group \
  --replication-subnet-group-identifier dms-subnets \
  --replication-subnet-group-description "DMS private subnets" \
  --subnet-ids subnet-<a> subnet-<b>

aws dms create-replication-instance \
  --replication-instance-identifier dms-wave1 \
  --replication-instance-class dms.t3.medium \
  --allocated-storage 100 \
  --engine-version 3.5.2 \
  --replication-subnet-group-identifier dms-subnets \
  --vpc-security-group-ids sg-<dms> \
  --no-publicly-accessible \
  --multi-az \
  --kms-key-id <kms-key-id> \
  --tags Key=Wave,Value=1

aws dms describe-replication-instances \
  --query 'ReplicationInstances[].{Id:ReplicationInstanceIdentifier,Status:ReplicationInstanceStatus,IP:ReplicationInstancePrivateIpAddress}' \
  --output table

# Serverless alternative (auto-scaling capacity units)
aws dms create-replication-config \
  --replication-config-identifier dms-sl-wave1 \
  --source-endpoint-arn <src-arn> --target-endpoint-arn <tgt-arn> \
  --replication-type full-load-and-cdc \
  --table-mappings file://table-mappings.json \
  --compute-config '{"MaxCapacityUnits":16,"MinCapacityUnits":2,"MultiAZ":true,"ReplicationSubnetGroupId":"dms-subnets","VpcSecurityGroupIds":["sg-<dms>"]}'
aws dms start-replication --replication-config-arn <arn> --start-replication-type start-replication
```

### Endpoints

```bash
# Source: on-prem MySQL
aws dms create-endpoint \
  --endpoint-identifier src-mysql --endpoint-type source --engine-name mysql \
  --server-name 10.10.5.20 --port 3306 \
  --username dmsuser --password '<password>' \
  --extra-connection-attributes "parallelLoadThreads=8;maxFileSize=1048576"

# Better: credentials from Secrets Manager
aws dms create-endpoint \
  --endpoint-identifier src-pg --endpoint-type source --engine-name postgres \
  --postgre-sql-settings '{"SecretsManagerSecretId":"arn:aws:secretsmanager:...:secret:pgsrc","SecretsManagerAccessRoleArn":"arn:aws:iam::...:role/dms-secrets","DatabaseName":"appdb","CaptureDdls":true,"PluginName":"pglogical"}'

# Target: Aurora PostgreSQL
aws dms create-endpoint \
  --endpoint-identifier tgt-aurora-pg --endpoint-type target --engine-name aurora-postgresql \
  --server-name <cluster-endpoint> --port 5432 \
  --username dmsadmin --password '<password>' --database-name appdb \
  --ssl-mode require --certificate-arn <cert-arn>

# Target: S3 (data lake / parquet)
aws dms create-endpoint \
  --endpoint-identifier tgt-s3 --endpoint-type target --engine-name s3 \
  --s3-settings '{"BucketName":"<bucket>","BucketFolder":"cdc","ServiceAccessRoleArn":"arn:aws:iam::...:role/dms-s3","DataFormat":"parquet","CompressionType":"GZIP","IncludeOpForFullLoad":true,"CdcInsertsAndUpdates":true,"TimestampColumnName":"dms_ts"}'

# ALWAYS test both endpoints before creating a task
aws dms test-connection --replication-instance-arn <ri-arn> --endpoint-arn <ep-arn>
aws dms describe-connections --filters Name=endpoint-arn,Values=<ep-arn>

aws dms describe-endpoints --query 'Endpoints[].{Id:EndpointIdentifier,Type:EndpointType,Engine:EngineName,Status:Status}' --output table
aws dms modify-endpoint --endpoint-arn <arn> --password '<new-password>'
```

### Table mappings and task settings

```bash
cat > table-mappings.json <<'EOF'
{
  "rules": [
    { "rule-type": "selection", "rule-id": "1", "rule-name": "include-app-schema",
      "object-locator": { "schema-name": "appdb", "table-name": "%" },
      "rule-action": "include", "filters": [] },
    { "rule-type": "selection", "rule-id": "2", "rule-name": "exclude-audit-tables",
      "object-locator": { "schema-name": "appdb", "table-name": "audit_%" },
      "rule-action": "exclude" },
    { "rule-type": "transformation", "rule-id": "3", "rule-name": "schema-to-lowercase",
      "rule-target": "schema", "object-locator": { "schema-name": "APPDB" },
      "rule-action": "convert-lowercase" },
    { "rule-type": "transformation", "rule-id": "4", "rule-name": "rename-schema",
      "rule-target": "schema", "object-locator": { "schema-name": "dbo" },
      "rule-action": "rename", "value": "public" },
    { "rule-type": "table-settings", "rule-id": "5", "rule-name": "parallel-load-big-table",
      "object-locator": { "schema-name": "appdb", "table-name": "orders" },
      "parallel-load": { "type": "ranges", "columns": ["id"],
        "boundaries": [["1000000"],["5000000"],["10000000"]] } }
  ]
}
EOF

cat > task-settings.json <<'EOF'
{
  "TargetMetadata": {
    "TargetSchema": "",
    "SupportLobs": true, "FullLobMode": false, "LobChunkSize": 64, "LimitedSizeLobMode": true,
    "LobMaxSize": 32, "BatchApplyEnabled": true, "ParallelLoadThreads": 8, "ParallelLoadBufferSize": 500
  },
  "FullLoadSettings": {
    "TargetTablePrepMode": "DO_NOTHING",
    "CreatePkAfterFullLoad": false,
    "MaxFullLoadSubTasks": 8,
    "TransactionConsistencyTimeout": 600,
    "CommitRate": 10000
  },
  "Logging": {
    "EnableLogging": true,
    "LogComponents": [
      {"Id":"SOURCE_UNLOAD","Severity":"LOGGER_SEVERITY_DEFAULT"},
      {"Id":"TARGET_LOAD","Severity":"LOGGER_SEVERITY_DEFAULT"},
      {"Id":"SOURCE_CAPTURE","Severity":"LOGGER_SEVERITY_DEFAULT"},
      {"Id":"TARGET_APPLY","Severity":"LOGGER_SEVERITY_DEFAULT"}
    ]
  },
  "ValidationSettings": {
    "EnableValidation": true, "ValidationMode": "ROW_LEVEL",
    "ThreadCount": 5, "PartitionSize": 10000,
    "FailureMaxCount": 10000, "TableFailureMaxCount": 1000,
    "HandleCollationDiff": true, "RecordFailureDelayLimitInMinutes": 0
  },
  "ErrorBehavior": {
    "DataErrorPolicy": "LOG_ERROR", "TableErrorPolicy": "SUSPEND_TABLE",
    "DataTruncationErrorPolicy": "LOG_ERROR",
    "ApplyErrorDeletePolicy": "IGNORE_RECORD",
    "ApplyErrorInsertPolicy": "LOG_ERROR", "ApplyErrorUpdatePolicy": "LOG_ERROR",
    "FullLoadIgnoreConflicts": true
  },
  "ChangeProcessingTuning": {
    "BatchApplyPreserveTransaction": true,
    "BatchApplyTimeoutMin": 1, "BatchApplyTimeoutMax": 30,
    "MinTransactionSize": 1000, "CommitTimeout": 1,
    "MemoryLimitTotal": 1024, "MemoryKeepTime": 60, "StatementCacheSize": 50
  },
  "ControlTablesSettings": {
    "ControlSchema": "dms_control",
    "HistoryTimeslotInMinutes": 5,
    "StatusTableEnabled": true,
    "SuspendedTablesTableEnabled": true,
    "HistoryTableEnabled": true
  }
}
EOF
```

### Tasks

```bash
aws dms create-replication-task \
  --replication-task-identifier mysql-to-aurora-wave1 \
  --source-endpoint-arn <src-arn> --target-endpoint-arn <tgt-arn> \
  --replication-instance-arn <ri-arn> \
  --migration-type full-load-and-cdc \
  --table-mappings file://table-mappings.json \
  --replication-task-settings file://task-settings.json \
  --tags Key=Wave,Value=1

# Start / resume / reload
aws dms start-replication-task --replication-task-arn <task-arn> \
  --start-replication-task-type start-replication
aws dms start-replication-task --replication-task-arn <task-arn> \
  --start-replication-task-type resume-processing
aws dms start-replication-task --replication-task-arn <task-arn> \
  --start-replication-task-type reload-target
aws dms stop-replication-task --replication-task-arn <task-arn>

# CDC from a specific point (after a Snowball/dump seed)
aws dms start-replication-task --replication-task-arn <task-arn> \
  --start-replication-task-type start-replication \
  --cdc-start-position "mysql-bin-changelog.000123:4567"     # or LSN / SCN / timestamp

# Monitor
aws dms describe-replication-tasks \
  --query 'ReplicationTasks[].{Task:ReplicationTaskIdentifier,Status:Status,Pct:ReplicationTaskStats.FullLoadProgressPercent,Tables:ReplicationTaskStats.TablesLoaded,Err:ReplicationTaskStats.TablesErrored}' \
  --output table

aws dms describe-table-statistics --replication-task-arn <task-arn> \
  --query 'TableStatistics[].{T:TableName,State:TableState,Ins:Inserts,Upd:Updates,Del:Deletes,Valid:ValidationState,Pending:ValidationPendingRecords,Failed:ValidationFailedRecords}' \
  --output table

# Only the problems
aws dms describe-table-statistics --replication-task-arn <task-arn> \
  --query 'TableStatistics[?ValidationState!=`Validated`].[TableName,TableState,ValidationState]' --output table

# Reload just a couple of tables
aws dms reload-tables --replication-task-arn <task-arn> \
  --tables-to-reload SchemaName=appdb,TableName=orders

# CDC lag (CloudWatch)
aws cloudwatch get-metric-statistics --namespace AWS/DMS \
  --metric-name CDCLatencyTarget --statistics Average --period 300 \
  --start-time $(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --dimensions Name=ReplicationInstanceIdentifier,Value=dms-wave1 \
               Name=ReplicationTaskIdentifier,Value=<task-id>

# Fleet Advisor (discover on-prem DB fleet)
aws dms describe-fleet-advisor-databases
aws dms describe-fleet-advisor-schemas
aws dms run-fleet-advisor-lsa-analysis

# Cleanup
aws dms delete-replication-task --replication-task-arn <task-arn>
aws dms delete-endpoint --endpoint-arn <ep-arn>
aws dms delete-replication-instance --replication-instance-arn <ri-arn>
```

### Source-side CDC prerequisites

```sql
-- MySQL
SET GLOBAL binlog_format = 'ROW';        -- and binlog_row_image=FULL in the parameter group
SHOW VARIABLES LIKE 'binlog%';
SHOW VARIABLES LIKE 'log_bin';
CALL mysql.rds_set_configuration('binlog retention hours', 72);   -- RDS source
CREATE USER 'dmsuser'@'%' IDENTIFIED BY '<pw>';
GRANT SELECT, RELOAD, REPLICATION CLIENT, REPLICATION SLAVE, SHOW VIEW ON *.* TO 'dmsuser'@'%';

-- PostgreSQL
ALTER SYSTEM SET wal_level = logical;    -- rds.logical_replication=1 on RDS, then reboot
SHOW wal_level;
SELECT * FROM pg_replication_slots;
CREATE USER dmsuser WITH PASSWORD '<pw>';
GRANT rds_replication TO dmsuser;        -- RDS
GRANT SELECT ON ALL TABLES IN SCHEMA public TO dmsuser;

-- Oracle
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
SELECT log_mode, supplemental_log_data_min FROM v$database;
GRANT SELECT ANY TRANSACTION, SELECT ANY TABLE, EXECUTE ON DBMS_LOGMNR TO dmsuser;

-- SQL Server
ALTER DATABASE appdb SET RECOVERY FULL;
EXEC msdb.dbo.rds_set_configuration 'binlog retention hours', 24;   -- RDS SQL Server
EXEC sys.sp_cdc_enable_db;
SELECT name, is_cdc_enabled, recovery_model_desc FROM sys.databases;
```

---

## 7. Schema Conversion (SCT / DMS SC)

```bash
# SCT is a desktop app, but it has a CLI for scripted conversions
sct-cli --script /path/to/conversion.scts

# DMS Schema Conversion (serverless, in-console) via CLI
aws dms create-data-provider --data-provider-name src-oracle --engine oracle \
  --settings '{"OracleSettings":{"ServerName":"10.10.5.30","Port":1521,"DatabaseName":"ORCL","SslMode":"none"}}'
aws dms create-data-provider --data-provider-name tgt-aurora-pg --engine aurora-postgresql \
  --settings '{"PostgreSqlSettings":{"ServerName":"<endpoint>","Port":5432,"DatabaseName":"appdb","SslMode":"require"}}'

aws dms create-instance-profile --instance-profile-name sc-profile \
  --subnet-group-identifier dms-subnets --vpc-security-groups sg-<id>

aws dms create-migration-project --migration-project-name ora-to-pg \
  --source-data-provider-descriptors '[{"DataProviderIdentifier":"src-oracle","SecretsManagerSecretId":"<arn>","SecretsManagerAccessRoleArn":"<arn>"}]' \
  --target-data-provider-descriptors '[{"DataProviderIdentifier":"tgt-aurora-pg","SecretsManagerSecretId":"<arn>","SecretsManagerAccessRoleArn":"<arn>"}]' \
  --instance-profile-identifier sc-profile

aws dms start-metadata-model-assessment --migration-project-identifier <id> \
  --selection-rules file://selection-rules.json
aws dms start-metadata-model-conversion --migration-project-identifier <id> \
  --selection-rules file://selection-rules.json
aws dms start-metadata-model-export-to-target --migration-project-identifier <id> \
  --selection-rules file://selection-rules.json --overwrite-extension-pack
aws dms describe-metadata-model-assessments --migration-project-identifier <id>
```

---

## 8. Native database dump and restore

```bash
# ---- MySQL / MariaDB ----
mysqldump -h <src-host> -u root -p \
  --single-transaction --routines --triggers --events \
  --set-gtid-purged=OFF --master-data=2 \
  appdb | gzip > appdb.sql.gz

gunzip -c appdb.sql.gz | mysql -h <rds-endpoint> -u admin -p appdb

# Faster for big DBs: parallel logical dump
mydumper -h <src> -u root -p '<pw>' -B appdb -t 8 -c -o /backup/appdb
myloader  -h <rds-endpoint> -u admin -p '<pw>' -B appdb -t 8 -d /backup/appdb

# ---- PostgreSQL ----
pg_dump -h <src-host> -U postgres -Fd -j 8 -f /backup/appdb appdb
pg_restore -h <rds-endpoint> -U admin -d appdb -j 8 /backup/appdb
pg_dumpall -h <src-host> -U postgres --globals-only > globals.sql   # roles/grants!

# Schema first, data later (lets you drop indexes for a fast load)
pg_dump -h <src> -U postgres -s appdb > schema.sql
pg_dump -h <src> -U postgres -a -Fc appdb > data.dump

# ---- SQL Server: native backup/restore via S3 ----
# On RDS SQL Server (needs the SQLSERVER_BACKUP_RESTORE option group):
EXEC msdb.dbo.rds_restore_database
  @restore_db_name='appdb',
  @s3_arn_to_restore_from='arn:aws:s3:::<bucket>/appdb.bak';
EXEC msdb.dbo.rds_task_status @db_name='appdb';

# ---- Oracle: Data Pump ----
expdp system/<pw>@ORCL schemas=APP directory=DPUMP dumpfile=app_%U.dmp parallel=8 compression=all
impdp admin/<pw>@<rds-endpoint>:1521/ORCL schemas=APP directory=DATA_PUMP_DIR dumpfile=app_%U.dmp parallel=8
# Move the dump file to RDS Oracle via DBMS_FILE_TRANSFER or S3 integration
```

---

## 9. RDS and Aurora

```bash
# Create a target instance
aws rds create-db-instance \
  --db-instance-identifier appdb-prod \
  --db-instance-class db.r6g.xlarge --engine postgres --engine-version 16.3 \
  --master-username admin --manage-master-user-password \
  --allocated-storage 200 --storage-type gp3 --storage-encrypted --kms-key-id <key> \
  --db-subnet-group-name db-private --vpc-security-group-ids sg-<db> \
  --multi-az --backup-retention-period 14 \
  --preferred-backup-window 17:00-18:00 --preferred-maintenance-window sun:19:00-sun:20:00 \
  --enable-performance-insights --enable-cloudwatch-logs-exports '["postgresql","upgrade"]' \
  --deletion-protection --copy-tags-to-snapshot \
  --tags Key=Application,Value=orders Key=Wave,Value=3

# Aurora cluster + writer
aws rds create-db-cluster --db-cluster-identifier appdb-aurora \
  --engine aurora-postgresql --engine-version 16.3 \
  --master-username admin --manage-master-user-password \
  --db-subnet-group-name db-private --vpc-security-group-ids sg-<db> \
  --storage-encrypted --kms-key-id <key> --backup-retention-period 14
aws rds create-db-instance --db-instance-identifier appdb-aurora-1 \
  --db-cluster-identifier appdb-aurora --engine aurora-postgresql \
  --db-instance-class db.r6g.large

# Parameter groups (match source semantics during migration!)
aws rds create-db-parameter-group --db-parameter-group-name pg16-migration \
  --db-parameter-group-family postgres16 --description "Migration tuning"
aws rds modify-db-parameter-group --db-parameter-group-name pg16-migration \
  --parameters "ParameterName=rds.logical_replication,ParameterValue=1,ApplyMethod=pending-reboot" \
               "ParameterName=maintenance_work_mem,ParameterValue=2097152,ApplyMethod=immediate" \
               "ParameterName=max_wal_size,ParameterValue=16384,ApplyMethod=immediate"

# Migration-time tuning (turn back afterwards!)
#   backup-retention-period 0, Multi-AZ off, bigger instance class, autovacuum off
aws rds modify-db-instance --db-instance-identifier appdb-prod \
  --backup-retention-period 0 --no-multi-az --apply-immediately
# ...after the load:
aws rds modify-db-instance --db-instance-identifier appdb-prod \
  --backup-retention-period 14 --multi-az --apply-immediately

aws rds describe-db-instances \
  --query 'DBInstances[].{Id:DBInstanceIdentifier,Status:DBInstanceStatus,EP:Endpoint.Address,Class:DBInstanceClass}' --output table

# Read replicas, snapshots, restores
aws rds create-db-instance-read-replica --db-instance-identifier appdb-ro \
  --source-db-instance-identifier appdb-prod
aws rds create-db-snapshot --db-instance-identifier appdb-prod --db-snapshot-identifier pre-cutover
aws rds restore-db-instance-from-db-snapshot --db-instance-identifier appdb-restore-test \
  --db-snapshot-identifier pre-cutover
aws rds restore-db-instance-to-point-in-time --source-db-instance-identifier appdb-prod \
  --target-db-instance-identifier appdb-pitr --restore-time 2026-07-30T02:00:00Z

# Migrate an on-prem MySQL/PG into RDS from an S3 backup
aws rds restore-db-instance-from-s3 --db-instance-identifier appdb-from-s3 \
  --engine mysql --source-engine mysql --source-engine-version 8.0.36 \
  --s3-bucket-name <bucket> --s3-prefix backups/appdb --s3-ingestion-role-arn <role-arn> \
  --db-instance-class db.r6g.large --master-username admin --manage-master-user-password \
  --allocated-storage 200
```

---

## 10. DataSync

```bash
# Agent (deploy the VM on-prem, then activate)
aws datasync create-agent --activation-key <ACTIVATION-KEY> --agent-name onprem-agent-1 \
  --vpc-endpoint-id vpce-<id> --subnet-arns <subnet-arn> --security-group-arns <sg-arn>
aws datasync list-agents
aws datasync describe-agent --agent-arn <agent-arn>

# Locations — source
aws datasync create-location-nfs --server-hostname 10.10.5.40 \
  --subdirectory /export/data \
  --on-prem-config AgentArns=<agent-arn> \
  --mount-options Version=NFS4_1

aws datasync create-location-smb --server-hostname 10.10.5.41 \
  --subdirectory /shares/finance --user svc_datasync --domain CORP \
  --password '<pw>' --agent-arns <agent-arn> --mount-options Version=SMB3

aws datasync create-location-hdfs --name-nodes Hostname=nn1,Port=8020 \
  --authentication-type SIMPLE --simple-user hdfs --agent-arns <agent-arn>

# Locations — target
aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::<bucket> --subdirectory /migrated \
  --s3-config BucketAccessRoleArn=<role-arn> --s3-storage-class STANDARD

aws datasync create-location-fsx-windows \
  --fsx-filesystem-arn <fsx-arn> --security-group-arns <sg-arn> \
  --user svc_datasync --domain CORP --password '<pw>'

aws datasync create-location-efs --efs-filesystem-arn <efs-arn> \
  --ec2-config SubnetArn=<subnet-arn>,SecurityGroupArns=<sg-arn>

# Task
aws datasync create-task \
  --source-location-arn <src-loc-arn> --destination-location-arn <dst-loc-arn> \
  --name finance-share-to-fsx \
  --cloud-watch-log-group-arn <lg-arn> \
  --options '{"VerifyMode":"POINT_IN_TIME_CONSISTENT","OverwriteMode":"ALWAYS","Atime":"BEST_EFFORT","Mtime":"PRESERVE","Uid":"INT_VALUE","Gid":"INT_VALUE","PreserveDeletedFiles":"PRESERVE","PreserveDevices":"NONE","PosixPermissions":"PRESERVE","BytesPerSecond":-1,"TaskQueueing":"ENABLED","LogLevel":"TRANSFER","TransferMode":"CHANGED","SecurityDescriptorCopyFlags":"OWNER_DACL_SACL"}' \
  --includes FilterType=SIMPLE_PATTERN,Value="/finance/*|/hr/*" \
  --excludes FilterType=SIMPLE_PATTERN,Value="*.tmp|*/~$*|*/Thumbs.db" \
  --schedule ScheduleExpression="cron(0 2 * * ? *)"

# Run and monitor
aws datasync start-task-execution --task-arn <task-arn>
aws datasync describe-task-execution --task-execution-arn <exec-arn> \
  --query '{Status:Status,Files:FilesTransferred,Bytes:BytesTransferred,Skipped:FilesSkipped,Errors:Result}'
aws datasync list-task-executions --task-arn <task-arn>
aws datasync cancel-task-execution --task-execution-arn <exec-arn>

# Bandwidth-limit an in-flight task (e.g. during business hours)
aws datasync update-task --task-arn <task-arn> --options BytesPerSecond=104857600
```

---

## 11. S3 data movement

```bash
# Bulk copy with parallelism tuning
aws configure set default.s3.max_concurrent_requests 40
aws configure set default.s3.multipart_chunksize 64MB
aws configure set default.s3.max_bandwidth 500MB/s

aws s3 sync /data/exports s3://<bucket>/exports/ --storage-class STANDARD_IA
aws s3 sync /data/exports s3://<bucket>/exports/ --delete --exclude "*.tmp"
aws s3 sync s3://<src-bucket>/ s3://<dst-bucket>/ --source-region us-east-1 --region ap-south-1

# Verify
aws s3 ls s3://<bucket>/exports/ --recursive --summarize | tail -3
aws s3api head-object --bucket <bucket> --key exports/big.tar --query '{Size:ContentLength,ETag:ETag,SSE:ServerSideEncryption}'

# Server-side batch copy of huge buckets
aws s3control create-job --account-id <acct> --operation '{"S3PutObjectCopy":{"TargetResource":"arn:aws:s3:::<dst-bucket>"}}' \
  --manifest file://manifest.json --report file://report.json --priority 10 --role-arn <role> --no-confirmation-required

# Lifecycle for archived source data (Retire pattern)
aws s3api put-bucket-lifecycle-configuration --bucket <archive-bucket> \
  --lifecycle-configuration '{"Rules":[{"ID":"to-deep-archive","Status":"Enabled","Filter":{"Prefix":"retired/"},"Transitions":[{"Days":30,"StorageClass":"DEEP_ARCHIVE"}]}]}'

# Transfer acceleration for long-distance uploads
aws s3api put-bucket-accelerate-configuration --bucket <bucket> --accelerate-configuration Status=Enabled
aws s3 cp big.tar s3://<bucket>/ --endpoint-url https://s3-accelerate.amazonaws.com
```

---

## 12. Snow Family

```bash
# Order a job
aws snowball create-job --job-type IMPORT --snowball-type EDGE_S \
  --resources '{"S3Resources":[{"BucketArn":"arn:aws:s3:::<bucket>","KeyRange":{}}]}' \
  --address-id <ADID...> --kms-key-arn <key-arn> --role-arn <snowball-role-arn> \
  --shipping-option SECOND_DAY --description "Wave 5 bulk file data"

aws snowball create-address --address file://address.json
aws snowball describe-job --job-id <JID...>
aws snowball list-jobs
aws snowball get-job-unlock-code --job-id <JID...>
aws snowball get-job-manifest --job-id <JID...>
aws snowball cancel-job --job-id <JID...>

# On the device (Snowball Edge client)
snowballEdge configure                                   # endpoint, manifest, unlock code
snowballEdge unlock-device --endpoint https://<device-ip> \
  --manifest-file ./manifest.bin --unlock-code <code>
snowballEdge describe-device --endpoint https://<device-ip>
snowballEdge list-services --endpoint https://<device-ip>
snowballEdge start-service --service-id s3 --virtual-network-interface-arns <arn>
snowballEdge get-secret-access-key --endpoint https://<device-ip>

# Copy data via the S3 adapter on the device
aws s3 cp /data/archive s3://<bucket> --recursive \
  --endpoint http://<device-ip>:8080 --profile snowball
aws s3 ls s3://<bucket> --endpoint http://<device-ip>:8080 --profile snowball

# NFS interface (often the easiest for file servers)
snowballEdge start-service --service-id nfs \
  --virtual-network-interface-arns <arn> --service-configuration AllowedHosts=10.10.0.0/16
mount -t nfs <device-ip>:/buckets/<bucket> /mnt/snow
rsync -avh --progress /data/ /mnt/snow/

snowballEdge get-service-status --service-id s3
snowballEdge describe-service --service-id s3
```

---

## 13. FSx, EFS and Storage Gateway

```bash
# FSx for Windows File Server (AD-joined)
aws fsx create-file-system --file-system-type WINDOWS \
  --storage-capacity 2048 --storage-type SSD --subnet-ids subnet-<a> subnet-<b> \
  --security-group-ids sg-<fsx> --kms-key-id <key> \
  --windows-configuration '{"ActiveDirectoryId":"d-<id>","DeploymentType":"MULTI_AZ_1","PreferredSubnetId":"subnet-<a>","ThroughputCapacity":512,"AutomaticBackupRetentionDays":14,"DailyAutomaticBackupStartTime":"01:00","CopyTagsToBackups":true,"Aliases":["fileserver01.corp.local"]}' \
  --tags Key=Application,Value=fileshares

aws fsx describe-file-systems --query 'FileSystems[].{Id:FileSystemId,Type:FileSystemType,DNS:DNSName,State:Lifecycle}' --output table
aws fsx create-data-repository-association --file-system-id fs-<id> \
  --file-system-path /ns1 --data-repository-path s3://<bucket>   # FSx for Lustre + S3

# FSx for NetApp ONTAP / OpenZFS / Lustre
aws fsx create-file-system --file-system-type ONTAP --storage-capacity 1024 \
  --subnet-ids subnet-<a> subnet-<b> \
  --ontap-configuration '{"DeploymentType":"MULTI_AZ_1","ThroughputCapacity":512,"PreferredSubnetId":"subnet-<a>"}'

# EFS
aws efs create-file-system --performance-mode generalPurpose --throughput-mode elastic \
  --encrypted --kms-key-id <key> --tags Key=Name,Value=app-shared
aws efs create-mount-target --file-system-id fs-<id> --subnet-id subnet-<a> --security-groups sg-<efs>
aws efs create-access-point --file-system-id fs-<id> \
  --posix-user Uid=1000,Gid=1000 --root-directory 'Path=/app,CreationInfo={OwnerUid=1000,OwnerGid=1000,Permissions=0755}'

# Mount on Linux (TLS)
sudo yum install -y amazon-efs-utils
sudo mount -t efs -o tls,iam fs-<id>:/ /mnt/efs
echo "fs-<id>:/ /mnt/efs efs _netdev,tls,iam 0 0" | sudo tee -a /etc/fstab

# Storage Gateway
aws storagegateway activate-gateway --activation-key <key> --gateway-name onprem-fgw \
  --gateway-timezone GMT+5:30 --gateway-region <region> --gateway-type FILE_S3
aws storagegateway create-nfs-file-share --client-token $(uuidgen) \
  --gateway-arn <gw-arn> --role <role-arn> --location-arn arn:aws:s3:::<bucket>
aws storagegateway list-gateways
aws storagegateway describe-gateway-information --gateway-arn <gw-arn>
```

---

## 14. Transfer Family

```bash
aws transfer create-server --protocols SFTP --identity-provider-type SERVICE_MANAGED \
  --endpoint-type VPC --endpoint-details 'VpcId=vpc-<id>,SubnetIds=subnet-<a>,SecurityGroupIds=sg-<id>' \
  --domain S3 --logging-role <role-arn> --tags Key=Purpose,Value=legacy-sftp

aws transfer create-user --server-id s-<id> --user-name partnerA \
  --role <access-role-arn> --home-directory /<bucket>/partnerA \
  --ssh-public-key-body "ssh-rsa AAAA..."

aws transfer list-servers
aws transfer describe-server --server-id s-<id>
aws transfer test-identity-provider --server-id s-<id> --user-name partnerA
```

---

## 15. Networking and connectivity

```bash
# VPC scaffolding
aws ec2 create-vpc --cidr-block 10.20.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc}]'
aws ec2 create-subnet --vpc-id vpc-<id> --cidr-block 10.20.1.0/24 \
  --availability-zone <az-a> --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-a}]'
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --internet-gateway-id igw-<id> --vpc-id vpc-<id>
aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway --subnet-id subnet-<public> --allocation-id eipalloc-<id>
aws ec2 create-route-table --vpc-id vpc-<id>
aws ec2 create-route --route-table-id rtb-<id> --destination-cidr-block 0.0.0.0/0 --gateway-id igw-<id>
aws ec2 associate-route-table --route-table-id rtb-<id> --subnet-id subnet-<id>

# Site-to-Site VPN
aws ec2 create-customer-gateway --type ipsec.1 --public-ip <onprem-public-ip> --bgp-asn 65000
aws ec2 create-vpn-gateway --type ipsec.1 --amazon-side-asn 64512
aws ec2 attach-vpn-gateway --vpn-gateway-id vgw-<id> --vpc-id vpc-<id>
aws ec2 create-vpn-connection --type ipsec.1 --customer-gateway-id cgw-<id> \
  --vpn-gateway-id vgw-<id> --options TunnelOptions='[{},{}]'
aws ec2 describe-vpn-connections \
  --query 'VpnConnections[].{Id:VpnConnectionId,State:State,Tunnels:VgwTelemetry[].Status}'

# Transit Gateway
aws ec2 create-transit-gateway --description "core-tgw" \
  --options AmazonSideAsn=64512,AutoAcceptSharedAttachments=disable,DefaultRouteTableAssociation=disable,DefaultRouteTablePropagation=disable,DnsSupport=enable
aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id tgw-<id> \
  --vpc-id vpc-<id> --subnet-ids subnet-<a> subnet-<b>
aws ec2 create-transit-gateway-route-table --transit-gateway-id tgw-<id>
aws ec2 create-transit-gateway-route --transit-gateway-route-table-id tgw-rtb-<id> \
  --destination-cidr-block 10.0.0.0/8 --transit-gateway-attachment-id tgw-attach-<id>
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id tgw-rtb-<id> \
  --filters Name=state,Values=active

# Direct Connect
aws directconnect describe-connections
aws directconnect describe-virtual-interfaces
aws directconnect create-private-virtual-interface --connection-id dxcon-<id> \
  --new-private-virtual-interface 'virtualInterfaceName=prod-vif,vlan=100,asn=65000,directConnectGatewayId=<dxgw-id>'
aws directconnect describe-virtual-interfaces \
  --query 'virtualInterfaces[].{Name:virtualInterfaceName,State:virtualInterfaceState,BGP:bgpPeers[0].bgpPeerState}'

# VPC endpoints (keep migration traffic private)
aws ec2 create-vpc-endpoint --vpc-id vpc-<id> --service-name com.amazonaws.<region>.s3 \
  --route-table-ids rtb-<id>                                   # Gateway endpoint
for SVC in ssm ssmmessages ec2messages mgn dms secretsmanager kms monitoring logs; do
  aws ec2 create-vpc-endpoint --vpc-id vpc-<id> --vpc-endpoint-type Interface \
    --service-name com.amazonaws.<region>.$SVC \
    --subnet-ids subnet-<a> subnet-<b> --security-group-ids sg-<endpoints> \
    --private-dns-enabled
done

# Security groups
aws ec2 create-security-group --group-name app-sg --description "App tier" --vpc-id vpc-<id>
aws ec2 authorize-security-group-ingress --group-id sg-<app> \
  --protocol tcp --port 443 --source-group sg-<alb>
aws ec2 authorize-security-group-egress --group-id sg-<mgn-staging> \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# Flow logs (essential during migration troubleshooting)
aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-<id> \
  --traffic-type ALL --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flowlogs --deliver-logs-permission-arn <role-arn>

# Reachability Analyzer — "why can't A talk to B?"
aws ec2 create-network-insights-path --source i-<src> --destination i-<dst> \
  --destination-port 3306 --protocol tcp
aws ec2 start-network-insights-analysis --network-insights-path-id nip-<id>
aws ec2 describe-network-insights-analyses --network-insights-analysis-ids nia-<id> \
  --query 'NetworkInsightsAnalyses[0].{Path:NetworkPathFound,Explanations:Explanations}'
```

---

## 16. Route 53 and DNS cutover

```bash
# Lower TTL a day BEFORE cutover
cat > lower-ttl.json <<'EOF'
{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{
  "Name":"app.example.com","Type":"A","TTL":60,
  "ResourceRecords":[{"Value":"<old-onprem-ip>"}]}}]}
EOF
aws route53 change-resource-record-sets --hosted-zone-id <ZID> --change-batch file://lower-ttl.json

# Cutover: point at the ALB (alias record)
cat > cutover.json <<'EOF'
{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{
  "Name":"app.example.com","Type":"A",
  "AliasTarget":{"HostedZoneId":"<alb-zone-id>","DNSName":"<alb-dns>","EvaluateTargetHealth":true}}}]}
EOF
aws route53 change-resource-record-sets --hosted-zone-id <ZID> --change-batch file://cutover.json
aws route53 get-change --id <change-id>

# Weighted canary: 10% to AWS, 90% on-prem
aws route53 change-resource-record-sets --hosted-zone-id <ZID> --change-batch '{
 "Changes":[
  {"Action":"UPSERT","ResourceRecordSet":{"Name":"app.example.com","Type":"A",
   "SetIdentifier":"aws","Weight":10,
   "AliasTarget":{"HostedZoneId":"<alb-zone>","DNSName":"<alb-dns>","EvaluateTargetHealth":true}}},
  {"Action":"UPSERT","ResourceRecordSet":{"Name":"app.example.com","Type":"A",
   "SetIdentifier":"onprem","Weight":90,"TTL":60,
   "ResourceRecords":[{"Value":"<old-ip>"}]}}]}'

# Hybrid DNS: resolver endpoints and forwarding rules
aws route53resolver create-resolver-endpoint --name outbound-to-onprem --direction OUTBOUND \
  --security-group-ids sg-<resolver> \
  --ip-addresses SubnetId=subnet-<a> SubnetId=subnet-<b>
aws route53resolver create-resolver-rule --name forward-corp-local --rule-type FORWARD \
  --domain-name corp.local --resolver-endpoint-id rslvr-out-<id> \
  --target-ips Ip=10.10.1.10,Port=53 Ip=10.10.1.11,Port=53
aws route53resolver associate-resolver-rule --resolver-rule-id rslvr-rr-<id> --vpc-id vpc-<id>
aws route53resolver create-resolver-endpoint --name inbound-from-onprem --direction INBOUND \
  --security-group-ids sg-<resolver> --ip-addresses SubnetId=subnet-<a> SubnetId=subnet-<b>

# Private hosted zone for internal names
aws route53 create-hosted-zone --name aws.corp.local --vpc VPCRegion=<region>,VPCId=vpc-<id> \
  --caller-reference $(date +%s) --hosted-zone-config PrivateZone=true

# Verify from a client
dig +short app.example.com
dig @10.10.1.10 app.example.com
nslookup app.example.com
```

---

## 17. EC2, EBS and AMIs

```bash
# Right-sizing research
aws ec2 describe-instance-types --filters Name=vcpu-info.default-vcpu,Values=4 \
  Name=memory-info.size-in-mib,Values=16384 Name=current-generation,Values=true \
  --query 'InstanceTypes[].InstanceType' --output text | tr '\t' '\n' | sort

aws ec2 describe-instance-types --instance-types m6i.xlarge \
  --query 'InstanceTypes[0].{vCPU:VCpuInfo.DefaultVCpus,MemGiB:MemoryInfo.SizeInMiB,Net:NetworkInfo.NetworkPerformance,ENA:NetworkInfo.EnaSupport,NVMe:EbsInfo.NvmeSupport}'

# Launch a target instance
aws ec2 run-instances --image-id ami-<id> --instance-type m6i.xlarge \
  --subnet-id subnet-<app> --security-group-ids sg-<app> \
  --iam-instance-profile Name=AppInstanceProfile \
  --key-name <key> --metadata-options HttpTokens=required \
  --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":100,"VolumeType":"gp3","Iops":3000,"Throughput":125,"Encrypted":true,"DeleteOnTermination":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=app01},{Key=Application,Value=orders},{Key=Wave,Value=1}]'

# Instance state, modify type
aws ec2 describe-instances --filters Name=tag:Wave,Values=1 \
  --query 'Reservations[].Instances[].{Id:InstanceId,Name:Tags[?Key==`Name`]|[0].Value,Type:InstanceType,State:State.Name,IP:PrivateIpAddress}' --output table
aws ec2 stop-instances --instance-ids i-<id>
aws ec2 modify-instance-attribute --instance-id i-<id> --instance-type m6i.2xlarge
aws ec2 start-instances --instance-ids i-<id>

# Boot troubleshooting for migrated instances
aws ec2 get-console-output --instance-id i-<id> --output text
aws ec2 get-console-screenshot --instance-id i-<id> --query ImageData --output text | base64 -d > screen.jpg

# EBS: gp2 → gp3, resize
aws ec2 modify-volume --volume-id vol-<id> --volume-type gp3 --iops 3000 --throughput 125
aws ec2 modify-volume --volume-id vol-<id> --size 500
aws ec2 describe-volumes-modifications --volume-ids vol-<id>
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[].{Id:VolumeId,Size:Size,AZ:AvailabilityZone}' --output table   # orphans

# Snapshots and AMIs (golden images / rollback points)
aws ec2 create-snapshot --volume-id vol-<id> --description "pre-cutover app01"
aws ec2 create-image --instance-id i-<id> --name "app01-pre-change-$(date +%F)" --no-reboot
aws ec2 describe-images --owners self --query 'Images[].{Id:ImageId,Name:Name,Date:CreationDate}' --output table
aws ec2 copy-image --source-region <src> --source-image-id ami-<id> --name "app01-dr-copy"
aws ec2 modify-image-attribute --image-id ami-<id> --launch-permission "Add=[{UserId=<acct>}]"

# Elastic IPs
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]' --output table
```

---

## 18. Load balancers and target groups

```bash
aws elbv2 create-load-balancer --name app-alb --type application --scheme internal \
  --subnets subnet-<a> subnet-<b> --security-groups sg-<alb> \
  --tags Key=Application,Value=orders

aws elbv2 create-target-group --name app-tg --protocol HTTP --port 8080 --vpc-id vpc-<id> \
  --target-type instance --health-check-path /health \
  --health-check-interval-seconds 15 --healthy-threshold-count 2 --unhealthy-threshold-count 3 \
  --matcher HttpCode=200

aws elbv2 register-targets --target-group-arn <tg-arn> --targets Id=i-<id1> Id=i-<id2>
aws elbv2 describe-target-health --target-group-arn <tg-arn> \
  --query 'TargetHealthDescriptions[].{Id:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason}' --output table

aws elbv2 create-listener --load-balancer-arn <alb-arn> --protocol HTTPS --port 443 \
  --certificates CertificateArn=<acm-arn> --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --default-actions Type=forward,TargetGroupArn=<tg-arn>

# Blue/green weighted shift at the ALB
aws elbv2 modify-listener --listener-arn <listener-arn> --default-actions '[{
  "Type":"forward","ForwardConfig":{"TargetGroups":[
    {"TargetGroupArn":"<blue-tg>","Weight":90},
    {"TargetGroupArn":"<green-tg>","Weight":10}]}}]'

# Certificates
aws acm request-certificate --domain-name app.example.com --validation-method DNS \
  --subject-alternative-names "*.app.example.com"
aws acm describe-certificate --certificate-arn <arn> --query 'Certificate.{Status:Status,Validation:DomainValidationOptions}'
aws acm import-certificate --certificate fileb://cert.pem --private-key fileb://key.pem \
  --certificate-chain fileb://chain.pem     # for certs migrated from on-prem
```

---

## 19. Systems Manager

```bash
# Inventory & connectivity of migrated instances
aws ssm describe-instance-information \
  --query 'InstanceInformationList[].{Id:InstanceId,Name:ComputerName,Ping:PingStatus,Platform:PlatformName,Agent:AgentVersion}' --output table

# Shell in without SSH keys or bastions
aws ssm start-session --target i-<id>
aws ssm start-session --target i-<id> \
  --document-name AWS-StartPortForwardingSession --parameters '{"portNumber":["3389"],"localPortNumber":["13389"]}'

# Run a validation command across a whole wave
aws ssm send-command --document-name "AWS-RunShellScript" \
  --targets Key=tag:Wave,Values=1 \
  --parameters 'commands=["systemctl is-active nginx","df -h /","curl -sf localhost:8080/health && echo OK"]' \
  --comment "wave1 post-cutover validation" \
  --output-s3-bucket-name <log-bucket> --output-s3-key-prefix ssm/wave1
aws ssm list-command-invocations --command-id <cmd-id> --details \
  --query 'CommandInvocations[].{Instance:InstanceId,Status:Status,Out:CommandPlugins[0].Output}'

aws ssm send-command --document-name "AWS-RunPowerShellScript" \
  --targets Key=tag:Wave,Values=1 \
  --parameters 'commands=["Get-Service | Where-Object {$_.StartType -eq \"Automatic\" -and $_.Status -ne \"Running\"} | Select Name"]'

# Patch baseline & maintenance window
aws ssm create-patch-baseline --name migrated-linux-baseline --operating-system AMAZON_LINUX_2 \
  --approval-rules 'PatchRules=[{PatchFilterGroup={PatchFilters=[{Key=CLASSIFICATION,Values=[Security]}]},ApproveAfterDays=7,ComplianceLevel=CRITICAL}]'
aws ssm create-maintenance-window --name migrated-patching --schedule "cron(0 2 ? * SUN *)" \
  --duration 3 --cutoff 1 --allow-unassociated-targets
aws ssm send-command --document-name AWS-RunPatchBaseline \
  --targets Key=tag:Wave,Values=1 --parameters 'Operation=Install'

# Parameter Store for migrated app config (replace config files with parameters)
aws ssm put-parameter --name /orders/prod/db-endpoint --value "<rds-endpoint>" --type String --overwrite
aws ssm put-parameter --name /orders/prod/db-password --value '<pw>' --type SecureString --key-id <kms>
aws ssm get-parameters-by-path --path /orders/prod --with-decryption

# Automation runbooks (wrap your cutover steps)
aws ssm start-automation-execution --document-name AWS-StopEC2Instance \
  --parameters InstanceId=i-<id>
aws ssm describe-automation-executions --filters Key=DocumentNamePrefix,Values=AWS-Stop
```

---

## 20. App2Container and containers

```bash
# On the app server
sudo app2container init                      # workspace, S3 bucket, profile
sudo app2container inventory                 # lists discovered Java/.NET apps
sudo app2container analyze --application-id java-tomcat-9e8e4799
# edit ./java-tomcat-9e8e4799/analysis.json (ports, env, base image)
sudo app2container containerize --application-id java-tomcat-9e8e4799
sudo app2container generate app-deployment --application-id java-tomcat-9e8e4799 \
  --deploy-target ecs                        # or eks
sudo app2container generate pipeline --application-id java-tomcat-9e8e4799

# ECR + ECS
aws ecr create-repository --repository-name orders-api --image-scanning-configuration scanOnPush=true
aws ecr get-login-password | docker login --username AWS --password-stdin <acct>.dkr.ecr.<region>.amazonaws.com
docker tag orders-api:latest <acct>.dkr.ecr.<region>.amazonaws.com/orders-api:v1
docker push <acct>.dkr.ecr.<region>.amazonaws.com/orders-api:v1

aws ecs create-cluster --cluster-name migration-cluster --capacity-providers FARGATE
aws ecs register-task-definition --cli-input-json file://taskdef.json
aws ecs create-service --cluster migration-cluster --service-name orders-api \
  --task-definition orders-api:1 --desired-count 2 --launch-type FARGATE \
  --network-configuration 'awsvpcConfiguration={subnets=[subnet-a,subnet-b],securityGroups=[sg-app],assignPublicIp=DISABLED}' \
  --load-balancers targetGroupArn=<tg-arn>,containerName=orders-api,containerPort=8080
aws ecs update-service --cluster migration-cluster --service orders-api --force-new-deployment
aws ecs describe-services --cluster migration-cluster --services orders-api \
  --query 'services[0].{Running:runningCount,Desired:desiredCount,Events:events[0:3].message}'
```

---

## 21. Refactor Spaces

```bash
aws migration-hub-refactor-spaces create-environment --name orders-refactor \
  --network-fabric-type TRANSIT_GATEWAY

aws migration-hub-refactor-spaces create-application --environment-identifier env-<id> \
  --name orders-app --proxy-type API_GATEWAY --vpc-id vpc-<id> \
  --api-gateway-proxy '{"EndpointType":"REGIONAL","StageName":"prod"}'

# The monolith becomes the default route
aws migration-hub-refactor-spaces create-service --environment-identifier env-<id> \
  --application-identifier app-<id> --name monolith --endpoint-type URL \
  --url-endpoint '{"Url":"http://<monolith-nlb-dns>"}' --vpc-id vpc-<id>
aws migration-hub-refactor-spaces create-route --environment-identifier env-<id> \
  --application-identifier app-<id> --service-identifier svc-<monolith> \
  --route-type DEFAULT --default-route '{"ActivationState":"ACTIVE"}'

# Peel off one path to a new Lambda service
aws migration-hub-refactor-spaces create-service --environment-identifier env-<id> \
  --application-identifier app-<id> --name pricing --endpoint-type LAMBDA \
  --lambda-endpoint '{"Arn":"arn:aws:lambda:...:function:pricing"}'
aws migration-hub-refactor-spaces create-route --environment-identifier env-<id> \
  --application-identifier app-<id> --service-identifier svc-<pricing> \
  --route-type URI_PATH \
  --uri-path-route '{"SourcePath":"/pricing","ActivationState":"ACTIVE","Methods":["GET","POST"],"IncludeChildPaths":true}'

aws migration-hub-refactor-spaces list-routes --environment-identifier env-<id> --application-identifier app-<id>
```

---

## 22. Backup, DR and snapshots

```bash
# Tag-driven backup for the whole migrated estate
aws backup create-backup-vault --backup-vault-name migrated-workloads --encryption-key-arn <kms>
aws backup create-backup-plan --backup-plan '{
 "BackupPlanName":"migrated-daily-35d",
 "Rules":[{"RuleName":"daily","TargetBackupVaultName":"migrated-workloads",
   "ScheduleExpression":"cron(0 18 ? * * *)","StartWindowMinutes":60,
   "CompletionWindowMinutes":180,
   "Lifecycle":{"DeleteAfterDays":35},"CopyActions":[]}]}'
aws backup create-backup-selection --backup-plan-id <plan-id> --backup-selection '{
 "SelectionName":"by-tag","IamRoleArn":"arn:aws:iam::<acct>:role/service-role/AWSBackupDefaultServiceRole",
 "ListOfTags":[{"ConditionType":"STRINGEQUALS","ConditionKey":"Backup","ConditionValue":"daily"}]}'

aws backup start-backup-job --backup-vault-name migrated-workloads \
  --resource-arn arn:aws:ec2:<region>:<acct>:instance/i-<id> --iam-role-arn <role>
aws backup list-backup-jobs --by-state COMPLETED --max-results 10
aws backup start-restore-job --recovery-point-arn <rp-arn> --iam-role-arn <role> \
  --metadata file://restore-metadata.json      # PROVE the restore works

# Data Lifecycle Manager for snapshot policies
aws dlm create-lifecycle-policy --description "wave1 daily snaps" \
  --state ENABLED --execution-role-arn <role> --policy-details file://dlm.json
```

---

## 23. Cost, sizing and optimization

```bash
# Migration Evaluator / TCO inputs come from discovery; validate against real spend:
aws ce get-cost-and-usage --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# Cost by migration wave (needs the tag activated as a cost allocation tag)
aws ce get-cost-and-usage --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=TAG,Key=MigrationWave

aws ce get-rightsizing-recommendation --service AmazonEC2 \
  --configuration 'RecommendationTarget=SAME_INSTANCE_FAMILY,BenefitsConsidered=true'
aws ce get-savings-plans-purchase-recommendation --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR --payment-option NO_UPFRONT --lookback-period-in-days SIXTY_DAYS

# Compute Optimizer — post-migration right-sizing from real data
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[].{Id:instanceArn,Current:currentInstanceType,Finding:finding,Rec:recommendationOptions[0].instanceType,Save:recommendationOptions[0].estimatedMonthlySavings}' --output table
aws compute-optimizer get-ebs-volume-recommendations
aws compute-optimizer get-enrollment-status
aws compute-optimizer update-enrollment-status --status Active

# Budgets and anomaly detection
aws budgets create-budget --account-id <acct> --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
aws ce create-anomaly-monitor --anomaly-monitor '{"MonitorName":"migration-monitor","MonitorType":"DIMENSIONAL","MonitorDimension":"SERVICE"}'

# Trusted Advisor (needs Business+ support)
aws support describe-trusted-advisor-checks --language en
aws support describe-trusted-advisor-check-result --check-id <id>
```

---

## 24. Governance, quotas and tagging

```bash
# Quotas — raise these EARLY
aws service-quotas list-service-quotas --service-code ec2 \
  --query 'Quotas[?contains(QuotaName,`Running On-Demand Standard`)].[QuotaName,Value]' --output table
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A
aws service-quotas request-service-quota-increase --service-code ec2 --quota-code L-1216C47A --desired-value 500
aws service-quotas list-requested-service-quota-change-history --service-code ec2

# Tagging: find untagged resources
aws resourcegroupstaggingapi get-resources --tags-per-page 100 \
  --query 'ResourceTagMappingList[?length(Tags)==`0`].ResourceARN'
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list <arn1> <arn2> \
  --tags Application=orders,Wave=1,Owner=platform-team,CostCenter=CC1234
aws resourcegroupstaggingapi get-tag-keys

# Config rules to enforce migration standards
aws configservice put-config-rule --config-rule '{
 "ConfigRuleName":"required-tags",
 "Source":{"Owner":"AWS","SourceIdentifier":"REQUIRED_TAGS"},
 "InputParameters":"{\"tag1Key\":\"Application\",\"tag2Key\":\"Owner\",\"tag3Key\":\"Environment\"}"}'
aws configservice put-config-rule --config-rule '{
 "ConfigRuleName":"ebs-encrypted","Source":{"Owner":"AWS","SourceIdentifier":"ENCRYPTED_VOLUMES"}}'
aws configservice describe-compliance-by-config-rule --config-rule-names required-tags

# EBS encryption by default (do this before any migration)
aws ec2 enable-ebs-encryption-by-default
aws ec2 modify-ebs-default-kms-key-id --kms-key-id <key-arn>
aws ec2 get-ebs-default-kms-key-id

# Organizations + SCP guardrails
aws organizations list-accounts --query 'Accounts[].{Id:Id,Name:Name,Status:Status}' --output table
aws organizations create-policy --type SERVICE_CONTROL_POLICY --name deny-other-regions \
  --description "Restrict to approved regions" --content file://scp-regions.json
aws organizations attach-policy --policy-id p-<id> --target-id ou-<id>
```

---

## 25. Source-side commands (Linux)

```bash
# ---- Pre-migration fact-finding ----
hostnamectl; cat /etc/os-release; uname -r
lscpu | egrep 'Model name|^CPU\(s\)|Socket|Thread'
free -h; swapon --show
lsblk -f; df -hT; cat /etc/fstab
ip -br addr; ip route; cat /etc/resolv.conf
ss -tulpn                                     # listening ports & owning processes
ss -tnp state established                     # who is connected right now
systemctl list-unit-files --state=enabled
crontab -l; ls -l /etc/cron.*; systemctl list-timers
rpm -qa | sort   ||  dpkg -l                  # installed packages
getenforce; sestatus                          # SELinux
firewall-cmd --list-all || iptables -S
mount | grep -E 'nfs|cifs'                    # external dependencies!
grep -rInE '([0-9]{1,3}\.){3}[0-9]{1,3}' /etc/ /opt/app/conf/ 2>/dev/null | head -50   # hardcoded IPs

# ---- Baseline before migration ----
sar -u -r -d 1 60 > baseline-$(hostname)-$(date +%F).txt   # or vmstat/iostat
top -bn1 | head -20

# ---- MGN readiness ----
sudo yum install -y dhclient rsync         # dhclient often required
lsblk | awk '{print $1,$4}'                # note every disk MGN must replicate
df -h /                                     # need ~2 GB free for the agent
curl -sI https://mgn.<region>.amazonaws.com | head -1        # endpoint reachability
nc -zv <staging-replication-server-ip> 1500                  # replication port

# ---- Post-migration verification on the target ----
curl -s http://169.254.169.254/latest/meta-data/instance-id     # IMDSv1
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-type
ethtool -i eth0 | grep driver              # expect 'ena'
lsblk                                       # confirm all disks present and mounted
systemctl --failed
journalctl -p err -b --no-pager | tail -40
```

---

## 26. Source-side commands (Windows / PowerShell)

```powershell
# ---- Pre-migration fact-finding ----
Get-ComputerInfo | Select CsName, OsName, OsVersion, CsNumberOfLogicalProcessors, CsTotalPhysicalMemory
Get-Disk | Select Number, FriendlyName, Size, PartitionStyle          # note BIOS vs GPT/UEFI!
Get-Volume | Select DriveLetter, FileSystemLabel, Size, SizeRemaining
Get-NetIPConfiguration
Get-NetTCPConnection -State Listen | Select LocalPort, OwningProcess |
  ForEach-Object { $_ | Add-Member -NotePropertyName Proc -NotePropertyValue (Get-Process -Id $_.OwningProcess).Name -PassThru }
Get-Service | Where-Object StartType -eq 'Automatic' | Select Name, Status
Get-ScheduledTask | Where-Object State -ne 'Disabled' | Select TaskName, TaskPath
Get-SmbShare; Get-SmbMapping
Get-WindowsFeature | Where-Object Installed
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
  Select DisplayName, DisplayVersion | Sort DisplayName
(Get-WmiObject Win32_ComputerSystem).Domain
Get-ChildItem C:\inetpub\wwwroot -Recurse -Include *.config |
  Select-String -Pattern '\b\d{1,3}(\.\d{1,3}){3}\b'          # hardcoded IPs

# ---- Baseline ----
Get-Counter '\Processor(_Total)\% Processor Time','\Memory\Available MBytes',
  '\LogicalDisk(_Total)\Disk Transfers/sec' -SampleInterval 5 -MaxSamples 720 |
  Export-Csv "C:\baseline-$env:COMPUTERNAME.csv"

# ---- MGN readiness ----
Get-WmiObject Win32_DiskDrive | Select DeviceID, Size, Model
Test-NetConnection mgn.<region>.amazonaws.com -Port 443
Test-NetConnection <staging-replication-server-ip> -Port 1500
[System.Environment]::OSVersion
Get-BitLockerVolume        # decrypt or plan for BitLocker before replicating
Confirm-SecureBootUEFI     # boot mode must match the launch template

# ---- Post-migration verification on the target ----
Invoke-RestMethod http://169.254.169.254/latest/meta-data/instance-id
Get-Service AmazonSSMAgent, AWSLiteAgent | Select Name, Status
Get-NetAdapter | Select Name, InterfaceDescription, LinkSpeed     # expect ENA/Elastic
Test-ComputerSecureChannel -Verbose                              # domain trust intact
Get-EventLog -LogName System -EntryType Error -Newest 25 | Select TimeGenerated, Source, Message
Get-Service | Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -ne 'Running' }
Get-ChildItem 'C:\ProgramData\Amazon\EC2Launch\log' | Select Name, LastWriteTime

# ---- Rejoin domain after a rehost, if needed ----
Add-Computer -DomainName corp.local -Credential (Get-Credential) -OUPath "OU=AWS,DC=corp,DC=local" -Restart
```

---

## 27. Validation one-liners

```bash
# Every instance in a wave: is it up, right-sized, tagged, and reporting to SSM?
aws ec2 describe-instances --filters Name=tag:Wave,Values=1 Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,Type:InstanceType,AZ:Placement.AvailabilityZone,IP:PrivateIpAddress,Profile:IamInstanceProfile.Arn}' --output table

# Instances missing required tags
aws ec2 describe-instances --query \
 'Reservations[].Instances[?!not_null(Tags[?Key==`Application`].Value)].InstanceId' --output text

# Unencrypted volumes (should be none)
aws ec2 describe-volumes --filters Name=encrypted,Values=false \
  --query 'Volumes[].{Id:VolumeId,Attached:Attachments[0].InstanceId}' --output table

# Security groups open to the world on admin ports
aws ec2 describe-security-groups --query \
 'SecurityGroups[?IpPermissions[?(FromPort==`22`||FromPort==`3389`) && IpRanges[?CidrIp==`0.0.0.0/0`]]].{Id:GroupId,Name:GroupName}' --output table

# DMS: any table not validated?
aws dms describe-table-statistics --replication-task-arn <arn> \
  --query 'TableStatistics[?ValidationState!=`Validated`].[TableName,ValidationState,ValidationFailedRecords]' --output table

# MGN: anything lagging or unhealthy?
aws mgn describe-source-servers --query \
 'items[?dataReplicationInfo.dataReplicationState!=`CONTINUOUS`].{H:sourceProperties.identificationHints.hostname,S:dataReplicationInfo.dataReplicationState,E:dataReplicationInfo.dataReplicationError.error}' --output table

# Are backups actually running?
aws backup list-backup-jobs --by-state FAILED --max-results 20 \
  --query 'BackupJobs[].{Res:ResourceArn,Msg:StatusMessage}' --output table

# Compare row counts across two databases quickly
mysql -h <src> -N -e "SELECT table_name, table_rows FROM information_schema.tables WHERE table_schema='appdb'" | sort > /tmp/src.txt
mysql -h <tgt> -N -e "SELECT table_name, table_rows FROM information_schema.tables WHERE table_schema='appdb'" | sort > /tmp/tgt.txt
diff /tmp/src.txt /tmp/tgt.txt

psql -h <src> -Atc "SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY 1" > /tmp/src.txt
psql -h <tgt> -Atc "SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY 1" > /tmp/tgt.txt
diff /tmp/src.txt /tmp/tgt.txt

# End-to-end app check
for i in $(seq 1 20); do curl -s -o /dev/null -w "%{http_code} %{time_total}\n" https://app.example.com/health; done
```

---

## 28. Cleanup and decommission

```bash
# ---- MGN teardown after successful cutover ----
aws mgn finalize-cutover --source-server-id <s-id>          # removes staging resources
aws mgn disconnect-from-service --source-server-id <s-id>
aws mgn delete-source-server --source-server-id <s-id>

# Uninstall the agent on the source
sudo /var/lib/aws-replication-agent/uninstall.sh            # Linux
# Windows: "C:\Program Files (x86)\AWS Replication Agent\uninstall.exe"

# ---- DMS teardown ----
aws dms stop-replication-task --replication-task-arn <arn>
aws dms delete-replication-task --replication-task-arn <arn>
aws dms delete-endpoint --endpoint-arn <src-arn>
aws dms delete-endpoint --endpoint-arn <tgt-arn>
aws dms delete-replication-instance --replication-instance-arn <ri-arn>

# ---- DataSync teardown ----
aws datasync delete-task --task-arn <task-arn>
aws datasync delete-location --location-arn <loc-arn>
aws datasync delete-agent --agent-arn <agent-arn>

# ---- Archive the retired source data ----
aws s3 sync /final-backup s3://<archive-bucket>/retired/<hostname>/ --storage-class DEEP_ARCHIVE
aws s3api put-object-tagging --bucket <archive-bucket> --key retired/<hostname>/manifest.json \
  --tagging 'TagSet=[{Key=RetainUntil,Value=2033-07-30},{Key=RetiredApp,Value=crystal-reports}]'

# ---- Sweep for waste after each wave ----
aws ec2 describe-volumes --filters Name=status,Values=available --query 'Volumes[].VolumeId' --output text
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].AllocationId' --output text
aws ec2 describe-snapshots --owner-id self \
  --query 'Snapshots[?StartTime<=`2026-01-01`].[SnapshotId,StartTime,Description]' --output table
aws ec2 describe-images --owners self --query 'Images[?CreationDate<=`2026-01-01`].[ImageId,Name]' --output table

# Delete carefully — always list first, then act
aws ec2 delete-volume --volume-id vol-<id>
aws ec2 release-address --allocation-id eipalloc-<id>
aws ec2 delete-snapshot --snapshot-id snap-<id>
aws ec2 deregister-image --image-id ami-<id>

# ---- Lab cleanup (nuke a sandbox VPC in order) ----
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances \
  --filters Name=vpc-id,Values=vpc-<id> Name=instance-state-name,Values=running,stopped \
  --query 'Reservations[].Instances[].InstanceId' --output text)
aws ec2 delete-nat-gateway --nat-gateway-id nat-<id>
aws ec2 detach-internet-gateway --internet-gateway-id igw-<id> --vpc-id vpc-<id>
aws ec2 delete-internet-gateway --internet-gateway-id igw-<id>
aws ec2 delete-subnet --subnet-id subnet-<id>
aws ec2 delete-vpc --vpc-id vpc-<id>
```

---

*Back to → [README.md](README.md) · Next → [hands-on-labs.md](hands-on-labs.md) · [troubleshooting.md](troubleshooting.md)*
