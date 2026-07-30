# AWS Migration — Hands-On Labs

Twelve labs that take you from an empty AWS account to a migrated, validated, cut-over application — and then to a clean-up that stops the bill.

These are designed to be done **in order**. Each one builds on the last. If you only have time for three, do **Lab 1, Lab 4 and Lab 6** — discovery, rehost, and database replatform are the skills you'll actually use.

---

## Before you start

### What you need

| Item | Notes |
|---|---|
| An AWS sandbox account | Not production. Set a budget alarm first. |
| AWS CLI v2, configured | `aws sts get-caller-identity` must work |
| A "source environment" | See the options below |
| ~4–8 hours total | Labs are 20–90 minutes each |
| Budget: roughly $15–40 | If you finish the cleanup in Lab 12 the same day |

### Simulating an on-premises environment

You don't need a data centre. Pick one:

| Option | How | Realism |
|---|---|---|
| **A second AWS region** (recommended) | Build "on-prem" servers in `us-east-1`, migrate to `ap-south-1`. MGN treats any non-target infrastructure as a source. | High, and free of local setup pain |
| **Another cloud** | An Azure/GCP VM as the source | Very high |
| **Local VirtualBox/VMware VMs** | Two VMs on your laptop with internet access | Highest, needs 8 GB+ RAM |
| **A second AWS account** | Cleanest separation | High |

Throughout these labs: **`SOURCE`** = your simulated data centre, **`TARGET`** = the AWS account/region you're migrating into.

```bash
# Keep two shells open, or use these consistently
export SOURCE_REGION=us-east-1
export TARGET_REGION=ap-south-1
export AWS_PAGER=""
```

### Set a cost guardrail right now

```bash
aws budgets create-budget --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget '{"BudgetName":"migration-labs","BudgetLimit":{"Amount":"50","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}'
```

### Lab index

| Lab | Topic | Time | R covered |
|---|---|---|---|
| [0](#lab-0--build-the-source-environment) | Build the "on-prem" source environment | 30 min | — |
| [1](#lab-1--discovery-with-application-discovery-service) | Discovery + Migration Hub | 45 min | Assess |
| [2](#lab-2--portfolio-analysis-and-the-6-r-decision) | Portfolio analysis, TCO, disposition | 40 min | All |
| [3](#lab-3--build-the-target-landing-zone) | Landing zone: VPC, subnets, endpoints | 40 min | — |
| [4](#lab-4--rehost-a-linux-server-with-mgn) | **Rehost a Linux server with MGN** | 75 min | Rehost |
| [5](#lab-5--rehost-a-windows-server-with-mgn) | Rehost Windows + post-launch actions | 60 min | Rehost |
| [6](#lab-6--replatform-a-database-with-dms) | **Replatform MySQL → RDS with DMS + CDC** | 90 min | Replatform |
| [7](#lab-7--heterogeneous-migration-with-schema-conversion) | Heterogeneous: SQL Server → PostgreSQL | 60 min | Replatform |
| [8](#lab-8--migrate-file-data-with-datasync) | File data with DataSync → S3/EFS | 45 min | Replatform |
| [9](#lab-9--offline-bulk-transfer-with-snow-family) | Snowball workflow (walkthrough) | 20 min | Replatform |
| [10](#lab-10--refactor-containerize-with-app2container) | Refactor: containerize with A2C → ECS | 60 min | Refactor |
| [11](#lab-11--execute-a-real-cutover) | **Full cutover: DNS, validation, rollback** | 60 min | All |
| [12](#lab-12--optimize-decommission-and-clean-up) | Optimize, decommission, clean up | 40 min | Post |

---

## Lab 0 — Build the source environment

**Goal:** create a realistic mini data centre: a Linux web/app server, a MySQL database server, a file server, and a Windows server. This is what you'll migrate in later labs.

### 0.1 Source VPC

```bash
export AWS_REGION=$SOURCE_REGION

SRC_VPC=$(aws ec2 create-vpc --cidr-block 172.31.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=onprem-datacenter}]' \
  --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id $SRC_VPC --enable-dns-hostnames

SRC_SUBNET=$(aws ec2 create-subnet --vpc-id $SRC_VPC --cidr-block 172.31.1.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=onprem-lan}]' \
  --query Subnet.SubnetId --output text)

SRC_IGW=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $SRC_IGW --vpc-id $SRC_VPC
SRC_RTB=$(aws ec2 create-route-table --vpc-id $SRC_VPC --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $SRC_RTB --destination-cidr-block 0.0.0.0/0 --gateway-id $SRC_IGW
aws ec2 associate-route-table --route-table-id $SRC_RTB --subnet-id $SRC_SUBNET
aws ec2 modify-subnet-attribute --subnet-id $SRC_SUBNET --map-public-ip-on-launch

SRC_SG=$(aws ec2 create-security-group --group-name onprem-sg \
  --description "Simulated on-prem LAN" --vpc-id $SRC_VPC --query GroupId --output text)
MYIP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress --group-id $SRC_SG --protocol tcp --port 22 --cidr $MYIP/32
aws ec2 authorize-security-group-ingress --group-id $SRC_SG --protocol tcp --port 3389 --cidr $MYIP/32
aws ec2 authorize-security-group-ingress --group-id $SRC_SG --protocol tcp --port 80 --cidr $MYIP/32
aws ec2 authorize-security-group-ingress --group-id $SRC_SG --protocol -1 --source-group $SRC_SG

echo "SRC_VPC=$SRC_VPC SRC_SUBNET=$SRC_SUBNET SRC_SG=$SRC_SG" | tee ~/lab-source.env
```

### 0.2 The "web/app server" (migration candidate: Rehost)

```bash
cat > /tmp/web-userdata.sh <<'EOF'
#!/bin/bash
dnf install -y nginx php php-mysqlnd
systemctl enable --now nginx
cat > /usr/share/nginx/html/index.php <<'PHP'
<?php
echo "<h1>Orders App</h1>";
echo "<p>Served by: " . gethostname() . "</p>";
echo "<p>Environment: ON-PREMISES</p>";
$db = getenv('DB_HOST') ?: 'db01.onprem.local';
echo "<p>Configured DB host: $db</p>";
?>
PHP
cat > /usr/share/nginx/html/health <<'H'
OK
H
# Deliberately leave a hardcoded IP in a config file — you'll find it in Lab 1
echo "db.host=172.31.1.50" > /etc/orders-app.conf
echo "cache.host=172.31.1.60" >> /etc/orders-app.conf
# A cron job that only runs monthly — the classic discovery trap
echo "0 3 1 * * root /usr/local/bin/monthly-close.sh" > /etc/cron.d/monthly-close
echo -e '#!/bin/bash\necho "monthly close ran at $(date)" >> /var/log/monthly-close.log' > /usr/local/bin/monthly-close.sh
chmod +x /usr/local/bin/monthly-close.sh
systemctl restart nginx
EOF

AL2023=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text)

aws ec2 run-instances --image-id $AL2023 --instance-type t3.small \
  --subnet-id $SRC_SUBNET --security-group-ids $SRC_SG \
  --user-data file:///tmp/web-userdata.sh \
  --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web01-onprem},{Key=Role,Value=web},{Key=App,Value=orders}]'
```

### 0.3 The database server (migration candidate: Replatform)

```bash
cat > /tmp/db-userdata.sh <<'EOF'
#!/bin/bash
dnf install -y mariadb105-server
systemctl enable --now mariadb

# Enable binary logging in ROW format — required for DMS CDC later
cat >> /etc/my.cnf.d/mariadb-server.cnf <<'CNF'
[mysqld]
server_id=1
log_bin=/var/log/mysql/mysql-bin.log
binlog_format=ROW
binlog_row_image=FULL
expire_logs_days=3
bind-address=0.0.0.0
CNF
mkdir -p /var/log/mysql && chown mysql:mysql /var/log/mysql
systemctl restart mariadb

mysql <<'SQL'
CREATE DATABASE appdb;
USE appdb;
CREATE TABLE customers (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(120) NOT NULL,
  email VARCHAR(200),
  country CHAR(2),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT NOT NULL,
  amount DECIMAL(12,2) NOT NULL,
  status VARCHAR(20) DEFAULT 'NEW',
  notes TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_cust (customer_id),
  CONSTRAINT fk_cust FOREIGN KEY (customer_id) REFERENCES customers(id)
);
-- a table with no primary key, to see how DMS behaves
CREATE TABLE audit_log (event VARCHAR(200), ts DATETIME DEFAULT CURRENT_TIMESTAMP);

INSERT INTO customers (name,email,country) VALUES
 ('Alice Kumar','alice@example.com','IN'),
 ('Bo Zhang','bo@example.com','CN'),
 ('Émile Rousseau','emile@example.com','FR'),
 ('Ünal Öztürk','unal@example.com','TR'),
 ('O''Brien, Sean','sean@example.com','IE');

-- generate 50k orders so full-load takes a visible amount of time
DELIMITER //
CREATE PROCEDURE seed(IN n INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < n DO
    INSERT INTO orders (customer_id, amount, status, notes)
    VALUES (1+FLOOR(RAND()*5), ROUND(RAND()*5000,2),
            ELT(1+FLOOR(RAND()*3),'NEW','SHIPPED','CANCELLED'),
            REPEAT('detail ',20));
    SET i = i + 1;
  END WHILE;
END//
DELIMITER ;
CALL seed(50000);

CREATE USER 'dmsuser'@'%' IDENTIFIED BY 'DmsPass123!';
GRANT SELECT, RELOAD, REPLICATION CLIENT, REPLICATION SLAVE, SHOW VIEW ON *.* TO 'dmsuser'@'%';
CREATE USER 'app'@'%' IDENTIFIED BY 'AppPass123!';
GRANT ALL ON appdb.* TO 'app'@'%';
FLUSH PRIVILEGES;
SQL
EOF

aws ec2 run-instances --image-id $AL2023 --instance-type t3.small \
  --subnet-id $SRC_SUBNET --security-group-ids $SRC_SG \
  --private-ip-address 172.31.1.50 \
  --user-data file:///tmp/db-userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=db01-onprem},{Key=Role,Value=database},{Key=App,Value=orders}]'
```

### 0.4 The file server (migration candidate: Replatform to EFS/FSx)

```bash
cat > /tmp/fs-userdata.sh <<'EOF'
#!/bin/bash
dnf install -y nfs-utils
mkdir -p /export/finance /export/hr /export/archive
for d in finance hr archive; do
  for i in $(seq 1 200); do
    dd if=/dev/urandom of=/export/$d/file_$i.dat bs=64k count=8 status=none
  done
done
mkdir -p /export/finance/2024 /export/finance/2025
touch /export/finance/report.tmp /export/hr/Thumbs.db     # to test exclude filters
echo "/export *(rw,sync,no_root_squash,no_subtree_check)" > /etc/exports
systemctl enable --now nfs-server
exportfs -ra
EOF

aws ec2 run-instances --image-id $AL2023 --instance-type t3.small \
  --subnet-id $SRC_SUBNET --security-group-ids $SRC_SG \
  --user-data file:///tmp/fs-userdata.sh \
  --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":30,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=fs01-onprem},{Key=Role,Value=fileserver},{Key=App,Value=shared}]'
```

### 0.5 The Windows server (migration candidate: Rehost)

```bash
WIN=$(aws ssm get-parameter --name /aws/service/ami-windows-latest/Windows_Server-2022-English-Full-Base \
  --query Parameter.Value --output text)

aws ec2 run-instances --image-id $WIN --instance-type t3.medium \
  --subnet-id $SRC_SUBNET --security-group-ids $SRC_SG --key-name <your-keypair> \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=win01-onprem},{Key=Role,Value=app},{Key=App,Value=reporting}]'
```

### 0.6 An idle server (migration candidate: Retire)

```bash
aws ec2 run-instances --image-id $AL2023 --instance-type t3.micro \
  --subnet-id $SRC_SUBNET --security-group-ids $SRC_SG \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=old-reports01-onprem},{Key=Role,Value=unknown}]'
```

### ✅ Lab 0 checkpoint

```bash
aws ec2 describe-instances --filters Name=vpc-id,Values=$SRC_VPC Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,IP:PrivateIpAddress,Pub:PublicIpAddress,Type:InstanceType}' --output table
curl -s http://<web01-public-ip>/index.php | head -5
```

You now have five "on-prem" servers. Note their IPs — you'll need them constantly.

---

## Lab 1 — Discovery with Application Discovery Service

**Goal:** discover your source estate, capture dependencies, and group servers into applications in Migration Hub.

**Time:** 45 min (plus ideally leaving agents running overnight)

### 1.1 Set the Migration Hub home region

```bash
export AWS_REGION=$TARGET_REGION
ACCT=$(aws sts get-caller-identity --query Account --output text)

aws migrationhub-config create-home-region-control \
  --home-region $TARGET_REGION --target Type=ACCOUNT,Id=$ACCT
aws migrationhub-config get-home-region
```

> ⚠️ Choose deliberately. Changing the home region later means losing your discovery data.

### 1.2 Create an IAM user for the discovery agents

Agents authenticate with static keys (this is the one place they're still used). Least privilege:

```bash
aws iam create-user --user-name ads-agent-user
aws iam attach-user-policy --user-name ads-agent-user \
  --policy-arn arn:aws:iam::aws:policy/AWSApplicationDiscoveryAgentAccess
aws iam create-access-key --user-name ads-agent-user
# Note the AccessKeyId and SecretAccessKey
```

### 1.3 Install the discovery agent on each source server

SSH into `web01-onprem`, `db01-onprem`, `fs01-onprem` and `old-reports01-onprem`:

```bash
export REG=$TARGET_REGION
curl -o aws-discovery-agent.tar.gz \
  https://s3.$REG.amazonaws.com/aws-discovery-agent.$REG/linux/latest/aws-discovery-agent.tar.gz
tar -xzf aws-discovery-agent.tar.gz
sudo bash install -r $REG -k <ACCESS_KEY_ID> -s <SECRET_ACCESS_KEY>

sudo systemctl status aws-discovery-daemon
sudo tail -20 /var/log/aws/discovery/aws-discovery-daemon.log
```

On `win01-onprem` (PowerShell as Administrator):

```powershell
$r = "<target-region>"
Invoke-WebRequest -Uri "https://s3.$r.amazonaws.com/aws-discovery-agent.$r/windows/latest/AWSDiscoveryAgentInstaller.exe" -OutFile "C:\Temp\ADSAgent.exe"
C:\Temp\ADSAgent.exe REGION=$r KEY_ID=<ACCESS_KEY_ID> KEY_SECRET=<SECRET> /quiet
Get-Service AWSDiscoveryAgent
```

### 1.4 Start collection and let it run

```bash
aws discovery describe-agents \
  --query 'agentsInfo[].{Id:agentId,Host:hostName,Health:health,Collection:collectionStatus,Version:version}' --output table

AGENTS=$(aws discovery describe-agents --query 'agentsInfo[].agentId' --output text)
aws discovery start-data-collection-by-agent-ids --agent-ids $AGENTS

# Turn on continuous export to S3 + Athena for analysis
aws discovery start-continuous-export
aws discovery describe-continuous-exports \
  --query 'descriptions[].{Status:status,Bucket:s3Bucket,Schema:schemaStorageConfig}'
```

**Generate some traffic** so the dependency map has something in it. From `web01`:

```bash
sudo dnf install -y mariadb105
for i in $(seq 1 200); do
  mysql -h 172.31.1.50 -u app -pAppPass123! -e "SELECT COUNT(*) FROM appdb.orders" >/dev/null
  sleep 2
done
```

Leave everything running for at least an hour — overnight if you can. In a real migration this is **2–4 weeks**.

### 1.5 Explore what was discovered

```bash
aws discovery get-discovery-summary

aws discovery list-configurations --configuration-type SERVER \
  --query 'configurations[].{Host:"server.hostname",OS:"server.osName",CPU:"server.cpuType",Type:"server.type"}' --output table

# Pick one and look in detail
CFG=$(aws discovery list-configurations --configuration-type SERVER \
  --query 'configurations[0]."server.configurationId"' --output text)
aws discovery describe-configurations --configuration-ids $CFG

# Processes and connections
aws discovery list-configurations --configuration-type PROCESS \
  --query 'configurations[0:20].{Name:"process.name",Cmd:"process.commandLine"}' --output table
aws discovery list-configurations --configuration-type CONNECTION \
  --query 'configurations[0:20]' --output table
```

### 1.6 Group servers into applications

```bash
APP_ORDERS=$(aws discovery create-application --name "orders-app" \
  --description "Web + DB, Wave 1" --query configurationId --output text)

# Get the configuration IDs of web01 and db01 (match on hostname)
WEB_CFG=$(aws discovery list-configurations --configuration-type SERVER \
  --filters name=server.hostname,values=<web01-hostname>,condition=CONTAINS \
  --query 'configurations[0]."server.configurationId"' --output text)
DB_CFG=$(aws discovery list-configurations --configuration-type SERVER \
  --filters name=server.hostname,values=<db01-hostname>,condition=CONTAINS \
  --query 'configurations[0]."server.configurationId"' --output text)

aws discovery associate-configuration-items-to-application \
  --application-configuration-id $APP_ORDERS --configuration-ids $WEB_CFG $DB_CFG

aws discovery create-tags --configuration-ids $WEB_CFG $DB_CFG \
  --tags key=Wave,value=1 key=Application,value=orders-app key=Disposition,value=TBD
```

### 1.7 Query discovery data in Athena

Console → Athena → database `application_discovery_service_database`:

```sql
-- Which servers are idle? (RETIRE candidates)
SELECT server_hostname,
       ROUND(AVG(avg_cpu_usage_pct),2)  AS avg_cpu,
       ROUND(MAX(max_cpu_usage_pct),2)  AS max_cpu,
       ROUND(AVG(avg_network_bytes_per_second_tx),0) AS avg_tx
FROM   application_discovery_service_database.os_info_agent
GROUP  BY server_hostname
ORDER  BY max_cpu ASC;

-- Dependency edges (who calls whom, on what port)
SELECT source_server_hostname, destination_server_hostname,
       destination_port, COUNT(*) AS connections
FROM   application_discovery_service_database.network_interface_agent
WHERE  destination_server_hostname IS NOT NULL
GROUP  BY 1,2,3
ORDER  BY connections DESC;

-- What is actually running on each host?
SELECT DISTINCT server_hostname, process_name
FROM   application_discovery_service_database.process_agent
ORDER  BY 1,2;
```

### ✅ Lab 1 checkpoint

- [ ] All agents show `HEALTHY` and `STARTED`
- [ ] You can see web01 → db01 on port 3306 in the connection data
- [ ] `old-reports01` shows near-zero CPU and no connections → your Retire candidate
- [ ] You found the hardcoded IP in `/etc/orders-app.conf` and the monthly cron job

**The lesson:** discovery told you *what*, but only reading the config file told you about the hardcoded `172.31.1.50`. Both matter.

---

## Lab 2 — Portfolio analysis and the 6 R decision

**Goal:** turn discovery data into a disposition register, a wave plan, and a cost estimate.

**Time:** 40 min · No AWS charges

### 2.1 Run Strategy Recommendations

Console → **Migration Hub → Strategy Recommendations → Get started**. Answer the questionnaire (business goals, target platform preferences), then:

```bash
aws migrationhubstrategy get-portfolio-summary
aws migrationhubstrategy start-assessment
aws migrationhubstrategy get-assessment --id <assessment-id>
aws migrationhubstrategy list-servers \
  --query 'serverInfos[].{Name:displayName,OS:systemInfo.osInfo.type,Rec:recommendationSet.strategy,Tool:recommendationSet.transformationTool.name}' --output table
```

Compare the machine's recommendation with your own judgement. It'll be right on the obvious ones and naive on the interesting ones — which is the point.

### 2.2 Build the disposition register

Create `portfolio.csv`:

```csv
app_name,server,os,vcpu,ram_gb,disk_gb,p95_cpu,criticality,deps,lifespan_yrs,disposition,reason,wave,target
orders-app,web01-onprem,AL2023,2,2,20,18,High,db01,4,Rehost,Simple stateless web tier; modernize later,1,t3.small
orders-app,db01-onprem,AL2023,2,2,20,35,High,web01,4,Replatform,MySQL to RDS - remove OS toil and get HA,3,db.t3.small
shared,fs01-onprem,AL2023,2,2,30,4,Medium,none,5,Replatform,NFS to EFS - elastic and multi-AZ,2,EFS
reporting,win01-onprem,Win2022,2,4,30,12,Low,db01,2,Rehost,Being replaced next year - no investment,2,t3.medium
legacy-reports,old-reports01-onprem,AL2023,1,1,8,1,None,none,0,Retire,No connections in 30 days; no owner found,0,N/A
```

Then reason through it out loud — this is the artefact that survives the project:

| Server | Disposition | Why |
|---|---|---|
| `web01` | **Rehost** | Stateless, simple, low risk. Get it out first. Containerize in the modernization backlog. |
| `db01` | **Replatform** | Managed backups, patching and Multi-AZ are worth more than OS access. Homogeneous → low risk. |
| `fs01` | **Replatform** | An NFS server is pure toil. EFS is elastic and multi-AZ. |
| `win01` | **Rehost** | Being decommissioned in 2 years — spend nothing on it. |
| `old-reports01` | **Retire** | 1% CPU, no connections, no owner. Archive and kill it. |

### 2.3 Estimate the cost of the target state

```bash
# What does the source spec cost if you lift it as-is?
aws pricing get-products --service-code AmazonEC2 --region us-east-1 \
  --filters Type=TERM_MATCH,Field=instanceType,Value=t3.small \
            Type=TERM_MATCH,Field=location,Value="Asia Pacific (Mumbai)" \
            Type=TERM_MATCH,Field=operatingSystem,Value=Linux \
            Type=TERM_MATCH,Field=tenancy,Value=Shared \
            Type=TERM_MATCH,Field=preInstalledSw,Value=NA \
            Type=TERM_MATCH,Field=capacitystatus,Value=Used \
  --query 'PriceList[0]' --output text | jq -r '.terms.OnDemand[].priceDimensions[].pricePerUnit.USD'
```

Right-size properly: `web01` runs at 18% p95 CPU on 2 vCPU. A `t3.small` is already the floor, but in a real estate you'd find 16 vCPU boxes at 8% and drop them to 4 vCPU. **That's where the 30% saving lives.**

Do the arithmetic for the wave, including:
- Compute (with a 1-year Compute Savings Plan ≈ −28%)
- EBS gp3 storage + snapshots
- RDS Multi-AZ premium (≈ 2× single-AZ, and worth it)
- EFS storage + Infrequent Access lifecycle
- Data transfer out
- Migration tooling: MGN is free per server for 90 days, but you pay for staging EBS, snapshots and replication instances; DMS instance-hours

### 2.4 Wave plan

| Wave | Content | Cutover | Risk | Rollback |
|---|---|---|---|---|
| 0 | Retire `old-reports01` | Any time | None | Restore from archive |
| 1 | Rehost `web01` | Sat 22:00 | Low | Repoint DNS to source |
| 2 | `fs01` → EFS, Rehost `win01` | Sat 22:00 | Low | Remount old NFS |
| 3 | `db01` → RDS (with `web01` reconfig) | Sat 22:00 | Medium | Reverse DMS + repoint app |

Note wave 1 before wave 3: the migrated `web01` in AWS talks back to the on-prem `db01` for two weeks. **That's a deliberate, temporary WAN hop** — acceptable here because the query volume is low. If it were chatty, you'd have to move them together. This decision is the single most common wave-planning judgement call.

### ✅ Lab 2 checkpoint

- [ ] Every server has a disposition and a written reason
- [ ] Every disposition has an owner and a wave
- [ ] You know your expected monthly AWS cost within ±20%
- [ ] You can explain why wave 1 and wave 3 are separate

---

## Lab 3 — Build the target landing zone

**Goal:** a target VPC that a migrated server can actually live in — three tiers, two AZs, endpoints, and a dedicated MGN staging subnet.

**Time:** 40 min

```bash
export AWS_REGION=$TARGET_REGION
AZ_A=${TARGET_REGION}a; AZ_B=${TARGET_REGION}b

# VPC — note the CIDR does NOT overlap the source 172.31.0.0/16
VPC=$(aws ec2 create-vpc --cidr-block 10.20.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=migration-target-vpc}]' \
  --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-support

mksubnet () { # name cidr az
  aws ec2 create-subnet --vpc-id $VPC --cidr-block $2 --availability-zone $3 \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$1}]" \
    --query Subnet.SubnetId --output text
}
PUB_A=$(mksubnet pub-a 10.20.1.0/24 $AZ_A)
PUB_B=$(mksubnet pub-b 10.20.2.0/24 $AZ_B)
APP_A=$(mksubnet app-a 10.20.11.0/24 $AZ_A)
APP_B=$(mksubnet app-b 10.20.12.0/24 $AZ_B)
DAT_A=$(mksubnet data-a 10.20.21.0/24 $AZ_A)
DAT_B=$(mksubnet data-b 10.20.22.0/24 $AZ_B)
STG_A=$(mksubnet mgn-staging-a 10.20.31.0/24 $AZ_A)     # dedicated MGN staging

# Internet + NAT
IGW=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC
EIP=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
NAT=$(aws ec2 create-nat-gateway --subnet-id $PUB_A --allocation-id $EIP --query NatGateway.NatGatewayId --output text)
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT

PUB_RTB=$(aws ec2 create-route-table --vpc-id $VPC --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $PUB_RTB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
for S in $PUB_A $PUB_B; do aws ec2 associate-route-table --route-table-id $PUB_RTB --subnet-id $S; done
aws ec2 modify-subnet-attribute --subnet-id $PUB_A --map-public-ip-on-launch

PRIV_RTB=$(aws ec2 create-route-table --vpc-id $VPC --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $PRIV_RTB --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT
for S in $APP_A $APP_B $DAT_A $DAT_B $STG_A; do
  aws ec2 associate-route-table --route-table-id $PRIV_RTB --subnet-id $S; done

# Security groups
SG_ALB=$(aws ec2 create-security-group --group-name alb-sg --description "ALB" --vpc-id $VPC --query GroupId --output text)
SG_APP=$(aws ec2 create-security-group --group-name app-sg --description "App tier" --vpc-id $VPC --query GroupId --output text)
SG_DB=$(aws ec2 create-security-group  --group-name db-sg  --description "Data tier" --vpc-id $VPC --query GroupId --output text)
SG_STG=$(aws ec2 create-security-group --group-name mgn-staging-sg --description "MGN staging" --vpc-id $VPC --query GroupId --output text)
SG_EP=$(aws ec2 create-security-group  --group-name endpoints-sg --description "Interface endpoints" --vpc-id $VPC --query GroupId --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ALB --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ALB --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_APP --protocol tcp --port 80 --source-group $SG_ALB
aws ec2 authorize-security-group-ingress --group-id $SG_DB --protocol tcp --port 3306 --source-group $SG_APP
aws ec2 authorize-security-group-ingress --group-id $SG_STG --protocol tcp --port 1500 --cidr 172.31.0.0/16
aws ec2 authorize-security-group-ingress --group-id $SG_EP --protocol tcp --port 443 --cidr 10.20.0.0/16

# EBS encryption by default — set this BEFORE any migration
aws ec2 enable-ebs-encryption-by-default

# VPC endpoints — keep migration traffic private and cut NAT costs
aws ec2 create-vpc-endpoint --vpc-id $VPC --service-name com.amazonaws.$TARGET_REGION.s3 \
  --route-table-ids $PRIV_RTB
for SVC in ssm ssmmessages ec2messages secretsmanager kms monitoring logs; do
  aws ec2 create-vpc-endpoint --vpc-id $VPC --vpc-endpoint-type Interface \
    --service-name com.amazonaws.$TARGET_REGION.$SVC \
    --subnet-ids $APP_A $APP_B --security-group-ids $SG_EP --private-dns-enabled >/dev/null
  echo "endpoint: $SVC"
done

# Flow logs — you'll want these when something can't connect
aws logs create-log-group --log-group-name /vpc/migration-flowlogs
aws ec2 create-flow-logs --resource-type VPC --resource-ids $VPC --traffic-type ALL \
  --log-destination-type cloud-watch-logs --log-group-name /vpc/migration-flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::$ACCT:role/flowlogsRole

# Instance profile so migrated servers get SSM automatically
aws iam create-role --role-name MigratedInstanceRole --assume-role-policy-document \
 '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name MigratedInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam attach-role-policy --role-name MigratedInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
aws iam create-instance-profile --instance-profile-name MigratedInstanceProfile
aws iam add-role-to-instance-profile --instance-profile-name MigratedInstanceProfile \
  --role-name MigratedInstanceRole

cat >> ~/lab-target.env <<EOF
VPC=$VPC PUB_A=$PUB_A PUB_B=$PUB_B APP_A=$APP_A APP_B=$APP_B
DAT_A=$DAT_A DAT_B=$DAT_B STG_A=$STG_A
SG_ALB=$SG_ALB SG_APP=$SG_APP SG_DB=$SG_DB SG_STG=$SG_STG
EOF
cat ~/lab-target.env
```

### ✅ Lab 3 checkpoint

- [ ] 7 subnets across 2 AZs; private subnets route via NAT
- [ ] Target CIDR (`10.20.0.0/16`) does **not** overlap source (`172.31.0.0/16`)
- [ ] EBS encryption by default is on
- [ ] Interface endpoints exist for SSM
- [ ] A dedicated MGN staging subnet exists

---

## Lab 4 — Rehost a Linux server with MGN

**This is the core lab.** Everything else is variations on it.

**Goal:** replicate `web01-onprem` into AWS, launch a test instance, validate it, then cut over.

**Time:** 75 min (mostly waiting on initial sync)

### 4.1 Initialize MGN

```bash
export AWS_REGION=$TARGET_REGION
source ~/lab-target.env

aws mgn initialize-service
aws mgn describe-replication-configuration-templates \
  --query 'items[0].replicationConfigurationTemplateID' --output text
```

### 4.2 Configure the replication template

```bash
RCT=$(aws mgn describe-replication-configuration-templates \
  --query 'items[0].replicationConfigurationTemplateID' --output text)

aws mgn update-replication-configuration-template \
  --replication-configuration-template-id $RCT \
  --staging-area-subnet-id $STG_A \
  --replication-server-instance-type t3.small \
  --default-large-staging-disk-type GP3 \
  --ebs-encryption DEFAULT \
  --use-dedicated-replication-server false \
  --create-public-ip true \
  --data-plane-routing PUBLIC_IP \
  --bandwidth-throttling 0 \
  --associate-default-security-group true \
  --staging-area-tags Purpose=mgn-staging,Wave=1
```

> In production you'd use `--data-plane-routing PRIVATE_IP` over VPN/DX and `--create-public-ip false`. We use public here because our "on-prem" is in another AWS region with no VPN.

### 4.3 Configure the launch template

```bash
LCT=$(aws mgn describe-launch-configuration-templates \
  --query 'items[0].launchConfigurationTemplateID' --output text)

aws mgn update-launch-configuration-template \
  --launch-configuration-template-id $LCT \
  --launch-disposition STARTED \
  --target-instance-type-right-sizing-method BASIC \
  --copy-tags true --copy-private-ip false \
  --boot-mode LEGACY_BIOS \
  --enable-map-auto-tagging true

# Post-launch action: install the CloudWatch agent on every launched instance
aws mgn put-template-action \
  --launch-configuration-template-id $LCT \
  --action-id install-cwagent --action-name "Install CloudWatch Agent" \
  --document-identifier AWS-ConfigureAWSPackage \
  --order 1001 --active true --category MONITORING_AND_OBSERVABILITY \
  --parameters '{"action":[{"type":"STRING","value":"Install"}],"name":[{"type":"STRING","value":"AmazonCloudWatchAgent"}]}'
```

### 4.4 Create agent credentials

```bash
aws iam create-user --user-name mgn-agent-user
aws iam attach-user-policy --user-name mgn-agent-user \
  --policy-arn arn:aws:iam::aws:policy/AWSApplicationMigrationAgentPolicy
aws iam create-access-key --user-name mgn-agent-user
```

### 4.5 Install the replication agent on web01-onprem

SSH into `web01-onprem`:

```bash
export REG=<target-region>
wget -O ./aws-replication-installer-init \
  https://aws-application-migration-service-$REG.s3.$REG.amazonaws.com/latest/linux/aws-replication-installer-init
sudo chmod +x aws-replication-installer-init
sudo ./aws-replication-installer-init \
  --region $REG \
  --aws-access-key-id <AKIA...> \
  --aws-secret-access-key <SECRET> \
  --no-prompt

# Watch it work
sudo systemctl status aws-replication-agent
sudo tail -f /var/lib/aws-replication-agent/agent.log.0
```

The agent will list the disks it's replicating and start the initial sync. **The server keeps serving traffic the whole time** — check `curl http://<web01-ip>/index.php` still works.

### 4.6 Watch replication

```bash
watch -n 30 'aws mgn describe-source-servers --query \
 "items[].{Host:sourceProperties.identificationHints.hostname,\
State:dataReplicationInfo.dataReplicationState,\
Pct:dataReplicationInfo.replicatedDisks[0].replicatedStorageBytes,\
Total:dataReplicationInfo.replicatedDisks[0].totalStorageBytes,\
Life:lifeCycle.state}" --output table'
```

Progression: `INITIATING` → `CREATING_SNAPSHOT` → `INITIAL_SYNC` → `CONTINUOUS` (lifecycle `READY_FOR_TEST`). A 20 GB disk over the same-account backbone takes ~10–20 minutes.

**Meanwhile, look at what MGN built for you:**

```bash
aws ec2 describe-instances --filters Name=tag:Name,Values="AWS Application Migration Service Replication Server" \
  --query 'Reservations[].Instances[].{Id:InstanceId,Type:InstanceType,Subnet:SubnetId,State:State.Name}' --output table
aws ec2 describe-volumes --filters Name=tag:Name,Values="*AWS Application Migration Service*" \
  --query 'Volumes[].{Id:VolumeId,Size:Size,Type:VolumeType,Enc:Encrypted}' --output table
```

That's the staging area: a small replication server plus EBS volumes holding your replicated blocks.

### 4.7 Override the launch settings for this server

```bash
SID=$(aws mgn describe-source-servers \
  --query 'items[?sourceProperties.identificationHints.hostname!=`null`]|[0].sourceServerID' --output text)
echo "Source server: $SID"

aws mgn update-launch-configuration --source-server-id $SID \
  --target-instance-type-right-sizing-method NONE \
  --launch-disposition STARTED --copy-tags true

# Point the launch template at the app subnet + SG + instance profile
aws ec2 describe-launch-templates \
  --query 'LaunchTemplates[?contains(LaunchTemplateName,`'"$SID"'`)].[LaunchTemplateId,LaunchTemplateName]' --output text
```

> The EC2 launch template MGN creates per source server (named after the source server ID) is where you set subnet, security group, instance type and IAM profile. Edit it in the console (**EC2 → Launch templates → Modify → Create new version**) — set subnet to your **app-a** subnet, SG to `app-sg`, instance type `t3.small`, IAM profile `MigratedInstanceProfile` — and make sure the new version is the **default**.

```bash
aws ec2 create-launch-template-version --launch-template-id lt-<id> \
  --source-version 1 \
  --launch-template-data '{"InstanceType":"t3.small","IamInstanceProfile":{"Name":"MigratedInstanceProfile"},
   "NetworkInterfaces":[{"DeviceIndex":0,"SubnetId":"'"$APP_A"'","Groups":["'"$SG_APP"'"],"AssociatePublicIpAddress":false,"DeleteOnTermination":true}]}'
aws ec2 modify-launch-template --launch-template-id lt-<id> --default-version 2
```

### 4.8 Launch a TEST instance

This is the step that makes rehosting safe. It's non-disruptive and repeatable.

```bash
aws mgn start-test --source-server-id $SID
JOB=$(aws mgn describe-jobs --query 'items[0].jobID' --output text)
aws mgn describe-job-log-items --job-id $JOB --query 'items[].{Time:logDateTime,Event:event}' --output table
aws mgn describe-jobs --filters jobIDs=$JOB --query 'items[0].{Status:status,Type:type,Servers:participatingServers}'
```

MGN takes a point-in-time snapshot, runs conversion (drivers, bootloader), and launches an EC2 instance — **while replication continues**.

### 4.9 Validate the test instance

```bash
TEST_ID=$(aws ec2 describe-instances \
  --filters Name=tag:Name,Values="*web01*" Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[?SubnetId==`'"$APP_A"'`].InstanceId' --output text)

aws ssm describe-instance-information --filters Key=InstanceIds,Values=$TEST_ID
aws ssm start-session --target $TEST_ID
```

Inside the instance:

```bash
hostname                                  # same hostname as the source
systemctl status nginx                    # should be running
curl -s localhost/index.php               # the app responds
ethtool -i eth0 | grep driver             # expect 'ena' — conversion worked
lsblk                                     # all disks present
df -h                                     # all filesystems mounted
cat /etc/orders-app.conf                  # STILL points at 172.31.1.50 — the hardcoded IP!
crontab -l; ls /etc/cron.d/               # the monthly job came across
journalctl -p err -b --no-pager | tail -20
```

**This is the moment the lab earns its keep.** The app config still points at the on-prem database IP. In a real cutover that's either fine (temporary WAN hop, if routing exists) or a total outage (if it doesn't). You have to decide, in advance, and document it.

```bash
# Fix it properly: externalise config instead of hardcoding
sudo sed -i 's|db.host=172.31.1.50|db.host=db.orders.internal|' /etc/orders-app.conf
# and create the Route 53 private hosted zone record for db.orders.internal
```

Now check it via a load balancer path (optional but realistic):

```bash
aws elbv2 create-load-balancer --name web-alb --type application --scheme internet-facing \
  --subnets $PUB_A $PUB_B --security-groups $SG_ALB
aws elbv2 create-target-group --name web-tg --protocol HTTP --port 80 --vpc-id $VPC \
  --target-type instance --health-check-path /health
aws elbv2 register-targets --target-group-arn <tg-arn> --targets Id=$TEST_ID
aws elbv2 describe-target-health --target-group-arn <tg-arn>
curl -s http://<alb-dns>/index.php
```

### 4.10 Mark ready, then cut over

```bash
# Tell MGN the test passed (this terminates the test instance)
aws mgn change-server-life-cycle-state --source-server-id $SID \
  --life-cycle '{"state":"READY_FOR_CUTOVER"}'

# ---- CUTOVER (in a real migration: stop the app on the source first) ----
# On web01-onprem:  sudo systemctl stop nginx
# Confirm lag is ~0:
aws mgn describe-source-servers --filters sourceServerIDs=$SID \
  --query 'items[0].dataReplicationInfo.{State:dataReplicationState,Lag:lagDuration,Backlog:replicatedDisks[0].backloggedStorageBytes}'

aws mgn start-cutover --source-server-ids $SID
aws mgn describe-jobs --query 'items[0].{Status:status,Type:type}'
```

Validate the cutover instance the same way, register it with the ALB, then:

```bash
# Finalize: removes the staging area and stops replication billing
aws mgn finalize-cutover --source-server-id $SID
aws mgn describe-source-servers --filters sourceServerIDs=$SID \
  --query 'items[0].lifeCycle.state'      # CUTOVER_COMPLETE
```

### 4.11 Practise the rollback

Before finalizing in a real migration, know how to go back:

```bash
# Terminate the launched target, keep replication intact
aws mgn terminate-target-instances --source-server-ids $SID
# Restart services on the source, repoint DNS back — done.
# Replication is still running, so you can retry the cutover later.
```

### ✅ Lab 4 checkpoint

- [ ] `web01` replicated to `CONTINUOUS` state
- [ ] A test instance launched, booted, and served the app
- [ ] You found the hardcoded IP during testing, not during cutover
- [ ] ENA driver present (conversion worked)
- [ ] Cutover completed and finalized
- [ ] You know the rollback command

---

## Lab 5 — Rehost a Windows server with MGN

**Goal:** same flow, but with the Windows-specific traps: boot mode, licensing, domain join, EC2Launch.

**Time:** 60 min

### 5.1 Pre-flight the source (this order matters for Windows)

RDP into `win01-onprem`:

```powershell
# Boot mode MUST match the MGN launch template
Confirm-SecureBootUEFI                  # True → set boot-mode UEFI on the template
Get-Disk | Select Number, PartitionStyle # GPT → usually UEFI; MBR → LEGACY_BIOS

# BitLocker will break replication — suspend or decrypt first
Get-BitLockerVolume

# Note every automatic service; you'll verify these on the target
Get-Service | Where-Object StartType -eq 'Automatic' | Select Name, Status |
  Export-Csv C:\pre-migration-services.csv -NoTypeInformation

# Scheduled tasks with stored credentials WILL break — inventory them
Get-ScheduledTask | Where-Object {$_.Principal.LogonType -eq 'Password'} |
  Select TaskName, @{n='User';e={$_.Principal.UserId}}

# Static IP in the NIC? MGN preserves the OS config, so it will try to keep it
Get-NetIPConfiguration | Select InterfaceAlias, IPv4Address, IPv4DefaultGateway
Get-NetIPInterface -AddressFamily IPv4 | Select InterfaceAlias, Dhcp

# Baseline
Get-Counter '\Processor(_Total)\% Processor Time','\Memory\Available MBytes' `
  -SampleInterval 5 -MaxSamples 60 | Export-Csv C:\baseline.csv
```

### 5.2 Set the launch template for Windows

```bash
aws mgn update-launch-configuration-template \
  --launch-configuration-template-id $LCT \
  --boot-mode UEFI \
  --licensing '{"osByol":false}' \
  --launch-disposition STARTED \
  --target-instance-type-right-sizing-method NONE
```

`osByol=false` = licence-included Windows (AWS bills the licence). `true` = bring your own licence, which needs a Dedicated Host and licence mobility rights.

### 5.3 Install the agent

```powershell
$r = "<target-region>"
Invoke-WebRequest -Uri "https://aws-application-migration-service-$r.s3.$r.amazonaws.com/latest/windows/AwsReplicationWindowsInstaller.exe" `
  -OutFile "C:\Temp\AwsReplicationWindowsInstaller.exe"
C:\Temp\AwsReplicationWindowsInstaller.exe --region $r `
  --aws-access-key-id <AKIA...> --aws-secret-access-key <SECRET> --no-prompt

Get-Service AWSReplicationAgent
Get-Content "C:\Program Files (x86)\AWS Replication Agent\agent.log.0" -Tail 40 -Wait
```

### 5.4 Add Windows post-launch actions

```bash
# Verify the instance is SSM-managed after launch
aws mgn put-template-action --launch-configuration-template-id $LCT \
  --action-id verify-ssm --action-name "Verify SSM Agent" \
  --document-identifier AWSMigration-VerifyMountedVolumes \
  --order 1002 --active true --category VALIDATION

# Domain join (needs an SSM document + AD credentials in Secrets Manager)
aws mgn put-template-action --launch-configuration-template-id $LCT \
  --action-id domain-join --action-name "Join corp.local" \
  --document-identifier AWS-JoinDirectoryServiceDomain \
  --order 1003 --active true --category CONFIGURATION \
  --parameters '{"directoryId":[{"type":"STRING","value":"d-xxxxx"}],"directoryName":[{"type":"STRING","value":"corp.local"}]}'
```

### 5.5 Test launch and validate

```bash
WSID=$(aws mgn describe-source-servers \
  --query 'items[?contains(sourceProperties.identificationHints.hostname,`WIN`)]|[0].sourceServerID' --output text)
aws mgn start-test --source-server-id $WSID
```

Once it's up, connect via Session Manager port-forwarding for RDP (no public IP, no bastion):

```bash
aws ssm start-session --target <test-instance-id> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3389"],"localPortNumber":["13389"]}'
# then RDP to localhost:13389
```

Validate inside Windows:

```powershell
Get-Service AmazonSSMAgent, AWSLiteAgent | Select Name, Status
Get-NetAdapter | Select Name, InterfaceDescription        # expect an Elastic/ENA adapter
Get-NetIPConfiguration                                    # DHCP from the VPC now
Get-Service | Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -ne 'Running' }
Compare-Object (Import-Csv C:\pre-migration-services.csv) `
  (Get-Service | Where StartType -eq 'Automatic' | Select Name,Status) -Property Name
Get-EventLog -LogName System -EntryType Error -Newest 25 | Select TimeGenerated, Source
(Get-WmiObject Win32_ComputerSystem).Domain
Get-ChildItem 'C:\ProgramData\Amazon\EC2Launch\log'
```

### ✅ Lab 5 checkpoint

- [ ] Boot mode matched (instance booted rather than hanging on a black screen)
- [ ] ENA adapter present
- [ ] Every automatic service that was running before is running now
- [ ] Windows activated (`slmgr /dlv` shows licensed)
- [ ] Scheduled tasks reviewed; ones with stored creds noted for fixing

---

## Lab 6 — Replatform a database with DMS

**The second most important lab.** Move `db01-onprem` (MariaDB) to Amazon RDS with continuous replication and near-zero downtime.

**Time:** 90 min

### 6.1 Create the target RDS instance

```bash
export AWS_REGION=$TARGET_REGION
source ~/lab-target.env

aws rds create-db-subnet-group --db-subnet-group-name db-private \
  --db-subnet-group-description "Data tier" --subnet-ids $DAT_A $DAT_B

aws rds create-db-parameter-group --db-parameter-group-name mysql80-migration \
  --db-parameter-group-family mysql8.0 --description "Migration tuning"

# Match source semantics; relax durability temporarily for a fast load
aws rds modify-db-parameter-group --db-parameter-group-name mysql80-migration --parameters \
 "ParameterName=sql_mode,ParameterValue='STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION',ApplyMethod=immediate" \
 "ParameterName=innodb_flush_log_at_trx_commit,ParameterValue=2,ApplyMethod=immediate" \
 "ParameterName=sync_binlog,ParameterValue=0,ApplyMethod=immediate" \
 "ParameterName=max_allowed_packet,ParameterValue=1073741824,ApplyMethod=immediate"

aws rds create-db-instance \
  --db-instance-identifier appdb-target \
  --db-instance-class db.t3.small --engine mysql --engine-version 8.0.36 \
  --master-username admin --master-user-password 'TargetPass123!' \
  --allocated-storage 50 --storage-type gp3 --storage-encrypted \
  --db-subnet-group-name db-private --vpc-security-group-ids $SG_DB \
  --db-parameter-group-name mysql80-migration \
  --backup-retention-period 0 --no-multi-az \
  --enable-cloudwatch-logs-exports '["error","slowquery"]' \
  --tags Key=Application,Value=orders Key=Wave,Value=3

aws rds wait db-instance-available --db-instance-identifier appdb-target
RDS_EP=$(aws rds describe-db-instances --db-instance-identifier appdb-target \
  --query 'DBInstances[0].Endpoint.Address' --output text)
echo "RDS endpoint: $RDS_EP"
```

> `backup-retention-period 0` and `no-multi-az` make the initial load much faster. **Turn both back on before cutover** — see 6.9.

### 6.2 Verify the source is CDC-ready

On `db01-onprem`:

```bash
mysql -e "SHOW VARIABLES LIKE 'log_bin'"           # ON
mysql -e "SHOW VARIABLES LIKE 'binlog_format'"     # ROW
mysql -e "SHOW VARIABLES LIKE 'binlog_row_image'"  # FULL
mysql -e "SHOW BINARY LOGS"
mysql -e "SELECT COUNT(*) FROM appdb.orders; SELECT COUNT(*) FROM appdb.customers"
mysql -e "SHOW GRANTS FOR 'dmsuser'@'%'"
```

Allow the DMS replication instance to reach port 3306 — add the target VPC CIDR to the source security group:

```bash
AWS_REGION=$SOURCE_REGION aws ec2 authorize-security-group-ingress \
  --group-id $SRC_SG --protocol tcp --port 3306 --cidr <nat-gateway-eip>/32
```

### 6.3 Create the DMS replication instance

```bash
for R in dms-vpc-role dms-cloudwatch-logs-role; do
  aws iam create-role --role-name $R --assume-role-policy-document \
   '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"dms.amazonaws.com"},"Action":"sts:AssumeRole"}]}' 2>/dev/null
done
aws iam attach-role-policy --role-name dms-vpc-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSVPCManagementRole
aws iam attach-role-policy --role-name dms-cloudwatch-logs-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSCloudWatchLogsRole

aws dms create-replication-subnet-group \
  --replication-subnet-group-identifier dms-subnets \
  --replication-subnet-group-description "DMS" --subnet-ids $DAT_A $DAT_B

aws dms create-replication-instance \
  --replication-instance-identifier dms-lab \
  --replication-instance-class dms.t3.medium --allocated-storage 50 \
  --replication-subnet-group-identifier dms-subnets \
  --vpc-security-group-ids $SG_DB --no-publicly-accessible --no-multi-az

aws dms wait replication-instance-available --filters Name=replication-instance-id,Values=dms-lab
RI_ARN=$(aws dms describe-replication-instances \
  --query 'ReplicationInstances[?ReplicationInstanceIdentifier==`dms-lab`].ReplicationInstanceArn' --output text)
```

> The DMS instance needs outbound access to the source. In this lab, it reaches the source's public IP via the NAT gateway. In production it would go over VPN/DX.

### 6.4 Create endpoints and test them

```bash
SRC_ARN=$(aws dms create-endpoint \
  --endpoint-identifier src-mariadb --endpoint-type source --engine-name mysql \
  --server-name <db01-public-ip> --port 3306 \
  --username dmsuser --password 'DmsPass123!' \
  --extra-connection-attributes "parallelLoadThreads=4" \
  --query 'Endpoint.EndpointArn' --output text)

TGT_ARN=$(aws dms create-endpoint \
  --endpoint-identifier tgt-rds-mysql --endpoint-type target --engine-name mysql \
  --server-name $RDS_EP --port 3306 \
  --username admin --password 'TargetPass123!' \
  --extra-connection-attributes "parallelLoadThreads=4;maxFileSize=1048576" \
  --query 'Endpoint.EndpointArn' --output text)

# ALWAYS test before creating the task — this catches 80% of DMS problems
aws dms test-connection --replication-instance-arn $RI_ARN --endpoint-arn $SRC_ARN
aws dms test-connection --replication-instance-arn $RI_ARN --endpoint-arn $TGT_ARN
sleep 60
aws dms describe-connections --query 'Connections[].{EP:EndpointIdentifier,Status:Status,Msg:LastFailureMessage}' --output table
```

### 6.5 Table mappings and task settings

```bash
cat > /tmp/mappings.json <<'EOF'
{"rules":[
 {"rule-type":"selection","rule-id":"1","rule-name":"include-appdb",
  "object-locator":{"schema-name":"appdb","table-name":"%"},"rule-action":"include","filters":[]},
 {"rule-type":"selection","rule-id":"2","rule-name":"exclude-audit",
  "object-locator":{"schema-name":"appdb","table-name":"audit_log"},"rule-action":"exclude"},
 {"rule-type":"table-settings","rule-id":"3","rule-name":"parallel-orders",
  "object-locator":{"schema-name":"appdb","table-name":"orders"},
  "parallel-load":{"type":"ranges","columns":["id"],"boundaries":[["12500"],["25000"],["37500"]]}}
]}
EOF

cat > /tmp/settings.json <<'EOF'
{
 "TargetMetadata":{"SupportLobs":true,"FullLobMode":false,"LimitedSizeLobMode":true,"LobMaxSize":32,
                   "BatchApplyEnabled":true,"ParallelLoadThreads":4},
 "FullLoadSettings":{"TargetTablePrepMode":"DROP_AND_CREATE","MaxFullLoadSubTasks":8,
                     "CommitRate":10000,"CreatePkAfterFullLoad":false},
 "ValidationSettings":{"EnableValidation":true,"ValidationMode":"ROW_LEVEL","ThreadCount":5,
                       "PartitionSize":10000,"HandleCollationDiff":true},
 "Logging":{"EnableLogging":true,"LogComponents":[
   {"Id":"SOURCE_UNLOAD","Severity":"LOGGER_SEVERITY_DEFAULT"},
   {"Id":"TARGET_LOAD","Severity":"LOGGER_SEVERITY_DEFAULT"},
   {"Id":"SOURCE_CAPTURE","Severity":"LOGGER_SEVERITY_DEFAULT"},
   {"Id":"TARGET_APPLY","Severity":"LOGGER_SEVERITY_DEFAULT"}]},
 "ErrorBehavior":{"DataErrorPolicy":"LOG_ERROR","TableErrorPolicy":"SUSPEND_TABLE",
                  "FullLoadIgnoreConflicts":true},
 "ControlTablesSettings":{"ControlSchema":"dms_control","StatusTableEnabled":true,
                          "SuspendedTablesTableEnabled":true,"HistoryTableEnabled":true}
}
EOF

TASK_ARN=$(aws dms create-replication-task \
  --replication-task-identifier mariadb-to-rds \
  --source-endpoint-arn $SRC_ARN --target-endpoint-arn $TGT_ARN \
  --replication-instance-arn $RI_ARN \
  --migration-type full-load-and-cdc \
  --table-mappings file:///tmp/mappings.json \
  --replication-task-settings file:///tmp/settings.json \
  --query 'ReplicationTask.ReplicationTaskArn' --output text)
```

### 6.6 Run the task

```bash
aws dms start-replication-task --replication-task-arn $TASK_ARN \
  --start-replication-task-type start-replication

watch -n 20 "aws dms describe-replication-tasks --filters Name=replication-task-arn,Values=$TASK_ARN \
 --query 'ReplicationTasks[0].{Status:Status,Pct:ReplicationTaskStats.FullLoadProgressPercent,\
Loaded:ReplicationTaskStats.TablesLoaded,Loading:ReplicationTaskStats.TablesLoading,\
Errored:ReplicationTaskStats.TablesErrored}'"
```

Expect: `starting` → `running` (full load) → `running` with CDC once `FullLoadProgressPercent` hits 100.

### 6.7 Prove CDC actually works

This is the part people skip and then don't trust. On `db01-onprem`, while the task runs:

```bash
mysql appdb -e "INSERT INTO customers (name,email,country) VALUES ('CDC Test','cdc@example.com','GB')"
mysql appdb -e "UPDATE orders SET status='SHIPPED' WHERE id <= 100"
mysql appdb -e "DELETE FROM orders WHERE id = 50000"
mysql appdb -e "SELECT COUNT(*) FROM orders; SELECT COUNT(*) FROM customers"
```

On the target (from an EC2 instance in the app subnet, or via a bastion):

```bash
mysql -h $RDS_EP -u admin -pTargetPass123! appdb -e \
 "SELECT * FROM customers WHERE email='cdc@example.com';
  SELECT COUNT(*) FROM orders WHERE status='SHIPPED';
  SELECT COUNT(*) FROM orders WHERE id=50000;"
```

Those changes should appear within seconds. Now check the lag and the validation report:

```bash
aws dms describe-table-statistics --replication-task-arn $TASK_ARN \
  --query 'TableStatistics[].{Table:TableName,State:TableState,Full:FullLoadRows,\
Ins:Inserts,Upd:Updates,Del:Deletes,Valid:ValidationState,Failed:ValidationFailedRecords}' --output table

aws cloudwatch get-metric-statistics --namespace AWS/DMS \
  --metric-name CDCLatencyTarget --statistics Average --period 300 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --dimensions Name=ReplicationInstanceIdentifier,Value=dms-lab \
               Name=ReplicationTaskIdentifier,Value=<task-id> \
  --query 'Datapoints[].{T:Timestamp,Lag:Average}' --output table
```

### 6.8 Validate the data properly

```bash
# Row counts
for T in customers orders; do
  S=$(mysql -h <db01-ip> -u app -pAppPass123! -N -e "SELECT COUNT(*) FROM appdb.$T")
  D=$(mysql -h $RDS_EP -u admin -pTargetPass123! -N -e "SELECT COUNT(*) FROM appdb.$T")
  echo "$T: source=$S target=$D $([ "$S" == "$D" ] && echo MATCH || echo MISMATCH)"
done

# Business-value checksums (row counts alone are not enough)
Q="SELECT COUNT(*), SUM(amount), MIN(created_at), MAX(created_at), COUNT(DISTINCT customer_id) FROM appdb.orders"
mysql -h <db01-ip> -u app -pAppPass123! -N -e "$Q"
mysql -h $RDS_EP  -u admin -pTargetPass123! -N -e "$Q"

# Edge cases — unicode, apostrophes, accents
mysql -h $RDS_EP -u admin -pTargetPass123! -e "SELECT name FROM appdb.customers"
```

**What DMS did NOT bring across — check each one:**

```bash
# Users and grants
mysql -h <db01-ip> -u root -N -e "SELECT user,host FROM mysql.user WHERE user NOT LIKE 'mysql%'"
# → recreate 'app'@'%' on the target manually

# Stored procedures, triggers, events, views
mysql -h <db01-ip> -N -e "SELECT routine_name,routine_type FROM information_schema.routines WHERE routine_schema='appdb'"
mysqldump -h <db01-ip> -u root --no-data --routines --triggers --events --skip-add-drop-table appdb > /tmp/objects.sql
mysql -h $RDS_EP -u admin -pTargetPass123! appdb < /tmp/objects.sql

# Auto-increment values — CRITICAL, or your first insert collides
mysql -h <db01-ip> -N -e "SELECT auto_increment FROM information_schema.tables WHERE table_schema='appdb' AND table_name='orders'"
mysql -h $RDS_EP -u admin -pTargetPass123! -N -e "SELECT auto_increment FROM information_schema.tables WHERE table_schema='appdb' AND table_name='orders'"
# If the target is behind:
mysql -h $RDS_EP -u admin -pTargetPass123! -e "ALTER TABLE appdb.orders AUTO_INCREMENT = <source_value + 1000>"

# Secondary indexes and foreign keys
mysql -h $RDS_EP -u admin -pTargetPass123! -e "SHOW INDEX FROM appdb.orders"
```

### 6.9 Cutover the database

```bash
# 1. Restore production settings on the target
aws rds modify-db-instance --db-instance-identifier appdb-target \
  --backup-retention-period 7 --multi-az --apply-immediately
aws rds wait db-instance-available --db-instance-identifier appdb-target

aws rds modify-db-parameter-group --db-parameter-group-name mysql80-migration --parameters \
 "ParameterName=innodb_flush_log_at_trx_commit,ParameterValue=1,ApplyMethod=immediate" \
 "ParameterName=sync_binlog,ParameterValue=1,ApplyMethod=immediate"

# 2. Take a pre-cutover snapshot (your rollback point)
aws rds create-db-snapshot --db-instance-identifier appdb-target \
  --db-snapshot-identifier pre-cutover-$(date +%Y%m%d)

# 3. Freeze writes on the source (the only real downtime)
#    On db01-onprem:  mysql -e "SET GLOBAL read_only = ON; FLUSH TABLES WITH READ LOCK;"

# 4. Wait for lag = 0
aws dms describe-table-statistics --replication-task-arn $TASK_ARN \
  --query 'TableStatistics[].{T:TableName,Ins:Inserts,Upd:Updates,Del:Deletes}' --output table

# 5. Final validation (row counts + checksums, as in 6.8)

# 6. Repoint the application
aws ssm put-parameter --name /orders/prod/db-endpoint --value "$RDS_EP" --type String --overwrite
# and/or update the Route 53 private zone CNAME db.orders.internal → $RDS_EP

# 7. Stop the DMS task
aws dms stop-replication-task --replication-task-arn $TASK_ARN

# 8. Smoke test the app end to end
curl -s http://<alb-dns>/index.php
```

### ✅ Lab 6 checkpoint

- [ ] Full load completed with 0 errored tables
- [ ] CDC verified with a live insert/update/delete
- [ ] DMS validation state = `Validated` for every table
- [ ] Row counts and sum-of-amount match exactly
- [ ] Users, procedures, triggers, views, and **auto-increment values** handled manually
- [ ] Multi-AZ and backups re-enabled before cutover
- [ ] You can name the rollback path (pre-cutover snapshot + reverse task)

---

## Lab 7 — Heterogeneous migration with schema conversion

**Goal:** understand why cross-engine migration is a different sport. We'll walk the SQL Server → Aurora PostgreSQL path.

**Time:** 60 min · This lab is deliberately more reading than typing — the point is the assessment report.

### 7.1 Set up data providers and a migration project

```bash
aws dms create-data-provider --data-provider-name src-sqlserver --engine sqlserver \
  --settings '{"MicrosoftSqlServerSettings":{"ServerName":"<sqlserver-host>","Port":1433,"DatabaseName":"appdb","SslMode":"none"}}'

aws dms create-data-provider --data-provider-name tgt-aurora-pg --engine aurora-postgresql \
  --settings '{"PostgreSqlSettings":{"ServerName":"<aurora-endpoint>","Port":5432,"DatabaseName":"appdb","SslMode":"require"}}'

aws dms create-instance-profile --instance-profile-name sc-profile \
  --subnet-group-identifier dms-subnets --vpc-security-groups $SG_DB

aws dms create-migration-project --migration-project-name sqlserver-to-pg \
  --source-data-provider-descriptors '[{"DataProviderIdentifier":"src-sqlserver","SecretsManagerSecretId":"<arn>","SecretsManagerAccessRoleArn":"<arn>"}]' \
  --target-data-provider-descriptors '[{"DataProviderIdentifier":"tgt-aurora-pg","SecretsManagerSecretId":"<arn>","SecretsManagerAccessRoleArn":"<arn>"}]' \
  --instance-profile-identifier sc-profile
```

### 7.2 Run the assessment — the most valuable output in the whole migration

```bash
aws dms start-metadata-model-assessment --migration-project-identifier <id> \
  --selection-rules file:///tmp/selection.json
aws dms describe-metadata-model-assessments --migration-project-identifier <id>
```

Read the report carefully. You're looking for:

| Report section | What it tells you |
|---|---|
| **Conversion %** | How much converts automatically (typically 70–95% of objects) |
| **Action items — simple** | Minutes each. Data type tweaks, identifier casing. |
| **Action items — medium** | Hours each. Function rewrites, sequence handling. |
| **Action items — significant** | Days each. Packages, hierarchical queries, proprietary features. **These drive your timeline.** |
| **Unsupported objects** | Things that need a new approach entirely |

### 7.3 The gaps you'll always hit (SQL Server → PostgreSQL)

| SQL Server | PostgreSQL | Fix |
|---|---|---|
| `IDENTITY(1,1)` | `GENERATED BY DEFAULT AS IDENTITY` / sequence | SCT converts; reset sequence value after load |
| `DATETIME` | `TIMESTAMP` | Precision differences; check rounding |
| `NVARCHAR` | `VARCHAR` (UTF-8 by default) | Check byte vs char length limits |
| `MONEY` | `NUMERIC(19,4)` | Watch rounding in reports |
| `UNIQUEIDENTIFIER` | `UUID` | Default generation differs (`NEWID()` → `gen_random_uuid()`) |
| `GETDATE()` | `NOW()` / `CURRENT_TIMESTAMP` | Application SQL too |
| `ISNULL()` | `COALESCE()` | Application SQL |
| `TOP n` | `LIMIT n` | Application SQL |
| `[bracketed identifiers]` | `"quoted"` and **case-sensitive** | Biggest source of "it worked yesterday" |
| `MERGE` | `INSERT ... ON CONFLICT` | Rewrite |
| Empty string ≠ NULL | Same, but Oracle differs | Test explicitly |
| T-SQL stored procs | PL/pgSQL | Manual, or use **Babelfish** to keep T-SQL |
| SQL Server Agent jobs | pg_cron / EventBridge + Lambda | Rebuild |
| Linked servers | FDW / application-level | Redesign |

### 7.4 The Babelfish shortcut

If the blocker is application code, **Babelfish for Aurora PostgreSQL** accepts T-SQL over the TDS protocol — so the app's connection string barely changes while the engine underneath becomes PostgreSQL.

```bash
aws rds create-db-cluster --db-cluster-identifier babelfish-cluster \
  --engine aurora-postgresql --engine-version 16.3 \
  --master-username postgres --manage-master-user-password \
  --db-subnet-group-name db-private --vpc-security-group-ids $SG_DB \
  --enable-babelfish \
  --db-cluster-parameter-group-name babelfish-pg16
# App connects on port 1433 with T-SQL; data lives in PostgreSQL
```

Use the **Babelfish Compass** tool first — it reports which T-SQL constructs are supported before you commit.

### 7.5 Then migrate the data

Convert schema → apply to target → run DMS `full-load-and-cdc` exactly as in Lab 6 → validate → repoint. The DMS mechanics are identical. **The effort is all in the schema and code.**

### ✅ Lab 7 checkpoint

- [ ] You can explain the difference between homogeneous and heterogeneous migration to a non-technical stakeholder
- [ ] You've read an assessment report and can point at the items that drive the timeline
- [ ] You know when Babelfish changes the calculus

---

## Lab 8 — Migrate file data with DataSync

**Goal:** move `/export` from `fs01-onprem` into EFS (and a copy into S3), with verification and filters.

**Time:** 45 min

### 8.1 Create the EFS target

```bash
EFS_ID=$(aws efs create-file-system --performance-mode generalPurpose \
  --throughput-mode elastic --encrypted \
  --tags Key=Name,Value=migrated-shares --query FileSystemId --output text)

SG_EFS=$(aws ec2 create-security-group --group-name efs-sg --description "EFS" --vpc-id $VPC --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_EFS --protocol tcp --port 2049 --source-group $SG_APP
aws ec2 authorize-security-group-ingress --group-id $SG_EFS --protocol tcp --port 2049 --cidr 10.20.0.0/16

for S in $APP_A $APP_B; do
  aws efs create-mount-target --file-system-id $EFS_ID --subnet-id $S --security-groups $SG_EFS
done
aws efs describe-mount-targets --file-system-id $EFS_ID \
  --query 'MountTargets[].{Id:MountTargetId,State:LifeCycleState,IP:IpAddress}' --output table
```

### 8.2 Deploy and activate the DataSync agent

In a real migration you deploy the agent OVA into VMware/Hyper-V/KVM. Here, launch the DataSync agent AMI in the **source** region:

```bash
AWS_REGION=$SOURCE_REGION
DS_AMI=$(aws ssm get-parameter --name /aws/service/datasync/ami --query Parameter.Value --output text)
aws ec2 run-instances --image-id $DS_AMI --instance-type m5.2xlarge \
  --subnet-id $SRC_SUBNET --security-group-ids $SRC_SG --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=datasync-agent}]'

# Get the activation key from the agent's local web UI
curl "http://<agent-public-ip>/?activationRegion=$TARGET_REGION"
# → returns activationKey=XXXXX-XXXXX-XXXXX-XXXXX-XXXXX

AWS_REGION=$TARGET_REGION
AGENT_ARN=$(aws datasync create-agent --activation-key <KEY> --agent-name onprem-agent \
  --query AgentArn --output text)
aws datasync describe-agent --agent-arn $AGENT_ARN --query '{Status:Status,Name:Name}'
```

### 8.3 Create locations

```bash
SRC_LOC=$(aws datasync create-location-nfs \
  --server-hostname <fs01-private-ip> --subdirectory /export \
  --on-prem-config AgentArns=$AGENT_ARN \
  --mount-options Version=NFS4_1 --query LocationArn --output text)

EFS_LOC=$(aws datasync create-location-efs \
  --efs-filesystem-arn arn:aws:elasticfilesystem:$TARGET_REGION:$ACCT:file-system/$EFS_ID \
  --ec2-config SubnetArn=arn:aws:ec2:$TARGET_REGION:$ACCT:subnet/$APP_A,SecurityGroupArns=arn:aws:ec2:$TARGET_REGION:$ACCT:security-group/$SG_EFS \
  --subdirectory / --query LocationArn --output text)

# Also send an archival copy to S3
aws s3 mb s3://migration-archive-$ACCT
cat > /tmp/ds-s3-trust.json <<EOF
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"datasync.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name DataSyncS3Role --assume-role-policy-document file:///tmp/ds-s3-trust.json
aws iam attach-role-policy --role-name DataSyncS3Role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

S3_LOC=$(aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::migration-archive-$ACCT --subdirectory /fileserver \
  --s3-config BucketAccessRoleArn=arn:aws:iam::$ACCT:role/DataSyncS3Role \
  --s3-storage-class STANDARD_IA --query LocationArn --output text)
```

### 8.4 Create and run the task, with filters

```bash
aws logs create-log-group --log-group-name /datasync/migration

TASK=$(aws datasync create-task \
  --source-location-arn $SRC_LOC --destination-location-arn $EFS_LOC \
  --name fs01-to-efs \
  --cloud-watch-log-group-arn arn:aws:logs:$TARGET_REGION:$ACCT:log-group:/datasync/migration \
  --options '{"VerifyMode":"POINT_IN_TIME_CONSISTENT","OverwriteMode":"ALWAYS","Atime":"BEST_EFFORT","Mtime":"PRESERVE","Uid":"INT_VALUE","Gid":"INT_VALUE","PreserveDeletedFiles":"PRESERVE","PosixPermissions":"PRESERVE","LogLevel":"TRANSFER","TransferMode":"CHANGED","BytesPerSecond":-1}' \
  --excludes FilterType=SIMPLE_PATTERN,Value="*.tmp|*/Thumbs.db" \
  --query TaskArn --output text)

EXEC=$(aws datasync start-task-execution --task-arn $TASK --query TaskExecutionArn --output text)

watch -n 15 "aws datasync describe-task-execution --task-execution-arn $EXEC \
 --query '{Status:Status,Files:FilesTransferred,Bytes:BytesTransferred,Skipped:FilesSkipped,Verified:FilesVerified}'"
```

Phases: `LAUNCHING` → `PREPARING` (scanning both sides) → `TRANSFERRING` → `VERIFYING` → `SUCCESS`.

### 8.5 Incremental sync — the pattern that makes cutover short

```bash
# Change something on the source
ssh fs01 'echo "new data $(date)" | sudo tee /export/finance/new-file.txt; sudo rm /export/hr/file_1.dat'

# Re-run: only the delta moves
EXEC2=$(aws datasync start-task-execution --task-arn $TASK --query TaskExecutionArn --output text)
aws datasync describe-task-execution --task-execution-arn $EXEC2 \
  --query '{Files:FilesTransferred,Bytes:BytesTransferred,Deleted:FilesDeleted}'
```

**The real-world pattern:** run this nightly for a week, then one final run during the cutover window. The last run takes minutes because everything else already moved.

```bash
aws datasync update-task --task-arn $TASK --schedule ScheduleExpression="cron(0 2 * * ? *)"
```

### 8.6 Verify from a client

Launch a small instance in `app-a` and mount:

```bash
sudo dnf install -y amazon-efs-utils
sudo mkdir -p /mnt/shares
sudo mount -t efs -o tls $EFS_ID:/ /mnt/shares
ls -la /mnt/shares/finance | head
find /mnt/shares -name "*.tmp" | wc -l          # should be 0 — the filter worked
find /mnt/shares -type f | wc -l
stat /mnt/shares/finance/file_1.dat             # check ownership and mtime preserved
echo "$EFS_ID:/ /mnt/shares efs _netdev,tls 0 0" | sudo tee -a /etc/fstab
```

### ✅ Lab 8 checkpoint

- [ ] Task completed with 0 errors and `FilesVerified` > 0
- [ ] `.tmp` and `Thumbs.db` files excluded
- [ ] POSIX permissions and mtimes preserved
- [ ] Incremental run transferred only the delta
- [ ] A client can mount EFS and read the data

---

## Lab 9 — Offline bulk transfer with Snow Family

**Goal:** understand the Snowball workflow and know when to reach for it. (Ordering a real device isn't practical in a lab, so this is a walkthrough plus the decision maths.)

**Time:** 20 min

### 9.1 Do the maths first

```bash
cat > /tmp/transfer-time.sh <<'EOF'
#!/bin/bash
# usage: transfer-time.sh <data_TB> <link_Mbps> [efficiency]
DATA_TB=$1; MBPS=$2; EFF=${3:-0.7}
BITS=$(echo "$DATA_TB * 1024 * 1024 * 8" | bc)     # megabits
SECS=$(echo "scale=0; $BITS / ($MBPS * $EFF)" | bc)
printf "%.1f TB over %s Mbps at %.0f%% efficiency: %.1f days\n" \
  "$DATA_TB" "$MBPS" "$(echo "$EFF*100"|bc)" "$(echo "scale=2; $SECS/86400"|bc)"
EOF
chmod +x /tmp/transfer-time.sh

/tmp/transfer-time.sh 80 100     # ~113 days  → Snowball, obviously
/tmp/transfer-time.sh 80 1000    # ~11 days   → borderline; depends on the window
/tmp/transfer-time.sh 10 1000    # ~1.4 days  → online is fine
```

**Decision rule:** if online transfer takes longer than your window (or would saturate the link the business needs), go offline. A Snowball Edge round trip is roughly 7–10 days end to end regardless of data size.

### 9.2 The workflow

```
1. aws snowball create-address / create-job --job-type IMPORT
2. Device ships (1–5 days). Rack it, connect 10 GbE, power on.
3. snowballEdge configure  +  unlock-device (manifest + unlock code from AWS)
4. Copy data:
     • S3 adapter:   aws s3 cp ... --endpoint http://<device>:8080 --profile snowball
     • NFS mount:    mount -t nfs <device>:/buckets/<bucket> /mnt/snow && rsync -avh
     • DataSync on device (agent runs locally) — best for large file counts
5. Verify checksums, capture a manifest of what you copied
6. Ship back (E Ink label updates automatically)
7. AWS imports to S3 (hours–days). Watch job status.
8. Validate in S3 against your manifest
9. Restore/seed the target from S3
10. Close the delta: DMS CDC from a recorded LSN/binlog position, or a DataSync
    incremental run for files
11. Cutover
12. AWS erases the device to NIST 800-88 standards
```

### 9.3 Commands you'd run on the device

```bash
snowballEdge configure                       # endpoint, manifest path, unlock code, profile
snowballEdge unlock-device --endpoint https://<device-ip> \
  --manifest-file ./JID-manifest.bin --unlock-code <code>
snowballEdge describe-device --endpoint https://<device-ip>
snowballEdge list-services --endpoint https://<device-ip>
snowballEdge start-service --service-id nfs \
  --virtual-network-interface-arns <vni-arn> \
  --service-configuration AllowedHosts=172.31.0.0/16
snowballEdge describe-service --service-id nfs

mount -t nfs <device-ip>:/buckets/migration-archive /mnt/snow
rsync -avh --progress --stats /export/ /mnt/snow/fileserver/
find /export -type f -exec md5sum {} \; > /tmp/source-manifest.txt

snowballEdge get-service-status --service-id nfs
umount /mnt/snow
snowballEdge stop-service --service-id nfs
```

### 9.4 The critical detail: the delta

The data on the device is a **point-in-time copy from when you copied it**. Between then and cutover, the source keeps changing. So:

- **Databases:** record the binlog position / LSN / SCN at the moment of the copy, then start DMS CDC from exactly that position.
- **Files:** run a DataSync incremental job after the import lands in S3.

Getting this wrong means silently losing a week of data. Write the position down. Twice.

### ✅ Lab 9 checkpoint

- [ ] You can calculate whether online or offline is faster
- [ ] You know the hybrid pattern (bulk offline + CDC/incremental online)
- [ ] You know why recording the CDC start position is non-negotiable

---

## Lab 10 — Refactor: containerize with App2Container

**Goal:** take the running `web01` app and turn it into a container on ECS Fargate — a real refactor step, done incrementally.

**Time:** 60 min

### 10.1 App2Container on a Java/.NET app

A2C supports Java and .NET specifically. On a source server running a Tomcat app:

```bash
# Install
curl -o AWSApp2Container-installer-linux.tar.gz \
  https://app2container-release-us-east-1.s3.us-east-1.amazonaws.com/latest/linux/AWSApp2Container-installer-linux.tar.gz
tar xvf AWSApp2Container-installer-linux.tar.gz
sudo ./install.sh

sudo app2container init            # workspace dir, S3 bucket, sign-up for metrics prompt
sudo app2container inventory       # → {"java-tomcat-9e8e4799": {...}}
sudo app2container analyze --application-id java-tomcat-9e8e4799
# Review and edit: ./java-tomcat-9e8e4799/analysis.json
#   - baseImage, ports, environment variables, files to include/exclude
sudo app2container containerize --application-id java-tomcat-9e8e4799
sudo app2container generate app-deployment --application-id java-tomcat-9e8e4799 --deploy-target ecs
sudo app2container generate pipeline --application-id java-tomcat-9e8e4799
```

### 10.2 Manual containerization (works for anything)

For our PHP/nginx app, do it by hand — this is the more common real path:

```bash
mkdir -p ~/orders-container && cd ~/orders-container
cat > Dockerfile <<'EOF'
FROM public.ecr.aws/docker/library/php:8.3-fpm-alpine
RUN apk add --no-cache nginx supervisor && \
    docker-php-ext-install pdo_mysql
COPY app/ /var/www/html/
COPY nginx.conf /etc/nginx/http.d/default.conf
COPY supervisord.conf /etc/supervisord.conf
# Externalise config — no hardcoded IPs. This is the whole point of refactoring.
ENV DB_HOST="" DB_USER="" DB_NAME="appdb"
EXPOSE 8080
HEALTHCHECK --interval=15s --timeout=3s CMD wget -qO- http://localhost:8080/health || exit 1
CMD ["/usr/bin/supervisord","-c","/etc/supervisord.conf"]
EOF
```

Copy the app files off `web01`, then:

```bash
aws ecr create-repository --repository-name orders-api --image-scanning-configuration scanOnPush=true
aws ecr get-login-password | docker login --username AWS --password-stdin $ACCT.dkr.ecr.$TARGET_REGION.amazonaws.com
docker build -t orders-api:v1 .
docker tag orders-api:v1 $ACCT.dkr.ecr.$TARGET_REGION.amazonaws.com/orders-api:v1
docker push $ACCT.dkr.ecr.$TARGET_REGION.amazonaws.com/orders-api:v1
aws ecr describe-image-scan-findings --repository-name orders-api --image-id imageTag=v1
```

### 10.3 Deploy to ECS Fargate

```bash
aws ecs create-cluster --cluster-name migration-cluster --capacity-providers FARGATE

aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document \
 '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

aws logs create-log-group --log-group-name /ecs/orders-api

cat > /tmp/taskdef.json <<EOF
{
 "family":"orders-api","networkMode":"awsvpc","requiresCompatibilities":["FARGATE"],
 "cpu":"512","memory":"1024",
 "executionRoleArn":"arn:aws:iam::$ACCT:role/ecsTaskExecutionRole",
 "containerDefinitions":[{
   "name":"orders-api",
   "image":"$ACCT.dkr.ecr.$TARGET_REGION.amazonaws.com/orders-api:v1",
   "portMappings":[{"containerPort":8080,"protocol":"tcp"}],
   "essential":true,
   "environment":[{"name":"DB_NAME","value":"appdb"}],
   "secrets":[
     {"name":"DB_HOST","valueFrom":"arn:aws:ssm:$TARGET_REGION:$ACCT:parameter/orders/prod/db-endpoint"},
     {"name":"DB_PASSWORD","valueFrom":"arn:aws:ssm:$TARGET_REGION:$ACCT:parameter/orders/prod/db-password"}],
   "logConfiguration":{"logDriver":"awslogs","options":{
     "awslogs-group":"/ecs/orders-api","awslogs-region":"$TARGET_REGION","awslogs-stream-prefix":"ecs"}},
   "healthCheck":{"command":["CMD-SHELL","wget -qO- http://localhost:8080/health || exit 1"],
     "interval":15,"timeout":3,"retries":3,"startPeriod":30}
 }]}
EOF
aws ecs register-task-definition --cli-input-json file:///tmp/taskdef.json

aws elbv2 create-target-group --name orders-ecs-tg --protocol HTTP --port 8080 \
  --vpc-id $VPC --target-type ip --health-check-path /health
ECS_TG=$(aws elbv2 describe-target-groups --names orders-ecs-tg --query 'TargetGroups[0].TargetGroupArn' --output text)

aws ecs create-service --cluster migration-cluster --service-name orders-api \
  --task-definition orders-api --desired-count 2 --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$APP_A,$APP_B],securityGroups=[$SG_APP],assignPublicIp=DISABLED}" \
  --load-balancers targetGroupArn=$ECS_TG,containerName=orders-api,containerPort=8080 \
  --health-check-grace-period-seconds 60

aws ecs describe-services --cluster migration-cluster --services orders-api \
  --query 'services[0].{Running:runningCount,Desired:desiredCount,Events:events[0:3].message}'
```

### 10.4 Shift traffic gradually (the safe way to refactor)

```bash
# 10% to containers, 90% still on the rehosted EC2 instance
aws elbv2 modify-listener --listener-arn <listener-arn> --default-actions '[{
 "Type":"forward","ForwardConfig":{"TargetGroups":[
   {"TargetGroupArn":"'"$EC2_TG"'","Weight":90},
   {"TargetGroupArn":"'"$ECS_TG"'","Weight":10}]}}]'

# Watch error rates and latency, then step up: 25 → 50 → 100
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime --statistics Average p95 --period 300 \
  --dimensions Name=TargetGroup,Value=<tg-suffix> \
  --start-time $(date -u -d '30 min ago' +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)
```

Rollback is one command: set the ECS weight to 0.

### 10.5 What changed, and why it matters

| Before (rehosted EC2) | After (Fargate) |
|---|---|
| Config in `/etc/orders-app.conf` with hardcoded IPs | Env vars from Parameter Store / Secrets Manager |
| Patch the OS yourself | No OS to patch |
| Scale by resizing the instance | Scale by `desired-count` / auto scaling |
| Deploy by copying files over SSH | Deploy by pushing an image + updating the service |
| Logs on local disk | Logs in CloudWatch by default |
| One instance = one failure domain | Tasks across two AZs |

### ✅ Lab 10 checkpoint

- [ ] Image built, scanned, and pushed to ECR
- [ ] ECS service running 2 healthy tasks across 2 AZs
- [ ] No credentials or IPs baked into the image
- [ ] Traffic shifted incrementally and rolled back once, deliberately
- [ ] You can explain when A2C helps and when hand-writing a Dockerfile is faster

---

## Lab 11 — Execute a real cutover

**Goal:** run a complete, timed cutover for the `orders-app` with DNS, validation, and a rehearsed rollback.

**Time:** 60 min

### 11.1 T-1 day: preparation

```bash
# 1. Lower DNS TTL so the switch propagates fast
aws route53 change-resource-record-sets --hosted-zone-id <ZID> --change-batch '{
 "Changes":[{"Action":"UPSERT","ResourceRecordSet":{
   "Name":"orders.example.com","Type":"A","TTL":60,
   "ResourceRecords":[{"Value":"<onprem-public-ip>"}]}}]}'

# 2. Verify replication health for everything in the wave
aws mgn describe-source-servers --query \
 'items[].{H:sourceProperties.identificationHints.hostname,S:dataReplicationInfo.dataReplicationState,L:lifeCycle.state}' --output table
aws dms describe-replication-tasks --query \
 'ReplicationTasks[].{T:ReplicationTaskIdentifier,S:Status,Err:ReplicationTaskStats.TablesErrored}' --output table

# 3. Snapshot everything (your rollback points)
aws rds create-db-snapshot --db-instance-identifier appdb-target --db-snapshot-identifier pre-cutover
aws ec2 create-snapshots --instance-specification InstanceId=<target-instance-id> \
  --description "pre-cutover wave1"

# 4. Capture the pre-cutover baseline for comparison
for i in $(seq 1 50); do curl -s -o /dev/null -w "%{http_code} %{time_total}\n" http://<onprem-ip>/index.php; done \
  | tee /tmp/baseline-onprem.txt
awk '{s+=$2} END {print "avg response:", s/NR"s"}' /tmp/baseline-onprem.txt
```

### 11.2 T-4h: go/no-go

Run through the gate explicitly. Any "no" = postpone.

```
[ ] Test launch signed off by the app owner (in writing)
[ ] Replication CONTINUOUS, DMS lag < 5 s, 0 errored tables
[ ] Rollback plan reviewed by someone who wasn't involved in writing it
[ ] Backups verified RESTORABLE (not just "the job succeeded")
[ ] Change request approved
[ ] Comms sent to users with the window and a status page link
[ ] The right people are actually available for the next 4 hours
[ ] Target monitoring and alarms already configured and firing correctly
```

### 11.3 T-0: execute

```bash
T0=$(date +%s); log(){ echo "T+$((($(date +%s)-T0)/60))m  $1" | tee -a /tmp/cutover.log; }

log "CUTOVER START"

# 1. Maintenance page / stop the source app
ssh web01-onprem 'sudo systemctl stop nginx'
log "source app stopped"

# 2. Freeze writes on the source database
ssh db01-onprem 'mysql -e "SET GLOBAL read_only=ON"'
log "source db read-only"

# 3. Wait for final sync
aws mgn describe-source-servers --filters sourceServerIDs=$SID \
  --query 'items[0].dataReplicationInfo.{Lag:lagDuration,Backlog:replicatedDisks[0].backloggedStorageBytes}'
aws dms describe-table-statistics --replication-task-arn $TASK_ARN \
  --query 'TableStatistics[].{T:TableName,I:Inserts,U:Updates,D:Deletes}' --output table
log "replication drained"

# 4. Launch cutover instances
aws mgn start-cutover --source-server-ids $SID
aws mgn describe-jobs --query 'items[0].status'
log "cutover instances launched"

# 5. Final data validation
for T in customers orders; do
  S=$(ssh db01-onprem "mysql -N -e \"SELECT COUNT(*) FROM appdb.$T\"")
  D=$(mysql -h $RDS_EP -u admin -pTargetPass123! -N -e "SELECT COUNT(*) FROM appdb.$T")
  echo "$T src=$S tgt=$D $([ "$S" == "$D" ] && echo OK || echo FAIL)"
done | tee -a /tmp/cutover.log
log "data validated"

# 6. Register targets and confirm health
aws elbv2 register-targets --target-group-arn $EC2_TG --targets Id=<cutover-instance-id>
aws elbv2 describe-target-health --target-group-arn $EC2_TG \
  --query 'TargetHealthDescriptions[].TargetHealth.State'
log "targets healthy"

# 7. Smoke tests BEFORE flipping DNS
curl -s http://<alb-dns>/index.php | grep -q "Orders App" && log "smoke: page OK"
curl -s -o /dev/null -w "%{http_code}" http://<alb-dns>/health
```

### 11.4 Flip DNS

```bash
aws route53 change-resource-record-sets --hosted-zone-id <ZID> --change-batch '{
 "Changes":[{"Action":"UPSERT","ResourceRecordSet":{
   "Name":"orders.example.com","Type":"A",
   "AliasTarget":{"HostedZoneId":"<alb-zone-id>","DNSName":"<alb-dns>","EvaluateTargetHealth":true}}}]}'

CHANGE=$(aws route53 change-resource-record-sets ... --query ChangeInfo.Id --output text)
aws route53 get-change --id $CHANGE --query 'ChangeInfo.Status'    # wait for INSYNC

# Verify from multiple resolvers
for R in 8.8.8.8 1.1.1.1 9.9.9.9; do echo -n "$R: "; dig +short @$R orders.example.com; done
log "DNS flipped"
```

### 11.5 Post-cutover validation

```bash
# Full smoke test suite
cat > /tmp/smoke.sh <<'EOF'
#!/bin/bash
FAIL=0
check(){ printf "%-45s" "$1"; if eval "$2" >/dev/null 2>&1; then echo "PASS"; else echo "FAIL"; FAIL=1; fi; }

check "App homepage returns 200"        "curl -sf https://orders.example.com/index.php"
check "Health endpoint OK"              "curl -sf https://orders.example.com/health"
check "DB read works"                   "mysql -h \$RDS_EP -u app -p\$APP_PW -e 'SELECT 1 FROM appdb.orders LIMIT 1'"
check "DB write works"                  "mysql -h \$RDS_EP -u app -p\$APP_PW -e \"INSERT INTO appdb.audit_log(event) VALUES('cutover smoke')\""
check "Instance is SSM managed"         "aws ssm describe-instance-information --filters Key=InstanceIds,Values=\$IID --query 'InstanceInformationList[0].PingStatus' --output text | grep -q Online"
check "CloudWatch agent reporting"      "aws cloudwatch list-metrics --namespace CWAgent --dimensions Name=InstanceId,Value=\$IID --query 'Metrics[0]' --output text | grep -q ."
check "Cron jobs present"               "aws ssm send-command --instance-ids \$IID --document-name AWS-RunShellScript --parameters 'commands=[\"ls /etc/cron.d/monthly-close\"]'"
check "Backup plan covers instance"     "aws backup list-protected-resources --query \"Results[?ResourceArn=='arn:aws:ec2:...:instance/\$IID']\" --output text | grep -q ."
check "No unencrypted volumes"          "! aws ec2 describe-volumes --filters Name=attachment.instance-id,Values=\$IID Name=encrypted,Values=false --query 'Volumes[0]' --output text | grep -q vol-"
exit $FAIL
EOF
chmod +x /tmp/smoke.sh && /tmp/smoke.sh

# Performance comparison against the baseline
for i in $(seq 1 50); do curl -s -o /dev/null -w "%{http_code} %{time_total}\n" https://orders.example.com/index.php; done \
  | tee /tmp/post-cutover.txt
echo "on-prem avg: $(awk '{s+=$2} END {print s/NR}' /tmp/baseline-onprem.txt)"
echo "aws avg:     $(awk '{s+=$2} END {print s/NR}' /tmp/post-cutover.txt)"
```

### 11.6 Practise the rollback

Do this at least once, in a test, so it isn't theoretical:

```bash
# 1. Announce
# 2. Revert DNS
aws route53 change-resource-record-sets --hosted-zone-id <ZID> --change-batch '{
 "Changes":[{"Action":"UPSERT","ResourceRecordSet":{"Name":"orders.example.com","Type":"A","TTL":60,
   "ResourceRecords":[{"Value":"<onprem-public-ip>"}]}}]}'

# 3. Restart the source
ssh db01-onprem 'mysql -e "SET GLOBAL read_only=OFF"'
ssh web01-onprem 'sudo systemctl start nginx'

# 4. Verify the source is serving
curl -s http://<onprem-ip>/index.php | grep "ON-PREMISES"

# 5. Stop (do NOT delete) the target
aws ec2 stop-instances --instance-ids <cutover-instance-id>

# 6. Reconcile: did any writes land on the target that need moving back?
mysql -h $RDS_EP -u admin -pTargetPass123! -e \
 "SELECT * FROM appdb.audit_log WHERE ts > '<cutover-time>'"
```

### 11.7 Hypercare

```bash
# Temporarily tighter alarms
aws cloudwatch put-metric-alarm --alarm-name orders-5xx-hypercare \
  --metric-name HTTPCode_Target_5XX_Count --namespace AWS/ApplicationELB \
  --statistic Sum --period 60 --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=LoadBalancer,Value=<lb-id> --alarm-actions <sns-arn>

aws cloudwatch put-metric-alarm --alarm-name orders-latency-hypercare \
  --metric-name TargetResponseTime --namespace AWS/ApplicationELB \
  --extended-statistic p95 --period 300 --evaluation-periods 2 --threshold 1.0 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=LoadBalancer,Value=<lb-id> --alarm-actions <sns-arn>
```

### ✅ Lab 11 checkpoint

- [ ] Cutover completed within the planned window (you timed it)
- [ ] All smoke tests passed before DNS was flipped
- [ ] Performance is within tolerance of the baseline
- [ ] You executed a rollback at least once and it worked
- [ ] Hypercare alarms are live
- [ ] `/tmp/cutover.log` is a usable record of what happened when

---

## Lab 12 — Optimize, decommission and clean up

**Goal:** capture the savings, retire the source, and leave the sandbox at $0.

**Time:** 40 min

### 12.1 Right-size with real data

```bash
aws compute-optimizer update-enrollment-status --status Active
# (needs 14 days of CloudWatch data to produce recommendations)

aws compute-optimizer get-ec2-instance-recommendations --query \
 'instanceRecommendations[].{Instance:instanceName,Current:currentInstanceType,Finding:finding,
Recommended:recommendationOptions[0].instanceType,
Savings:recommendationOptions[0].savingsOpportunity.estimatedMonthlySavings.value}' --output table

aws compute-optimizer get-ebs-volume-recommendations --query \
 'volumeRecommendations[].{Vol:volumeArn,Finding:finding,Rec:volumeRecommendationOptions[0].configuration}'

# gp2 → gp3 across the board
for V in $(aws ec2 describe-volumes --filters Name=volume-type,Values=gp2 --query 'Volumes[].VolumeId' --output text); do
  aws ec2 modify-volume --volume-id $V --volume-type gp3
done
```

### 12.2 Set up the operational baseline

```bash
# Tag-driven backup
aws backup create-backup-vault --backup-vault-name migrated-workloads
aws backup create-backup-plan --backup-plan '{
 "BackupPlanName":"migrated-daily",
 "Rules":[{"RuleName":"daily-35d","TargetBackupVaultName":"migrated-workloads",
  "ScheduleExpression":"cron(0 18 ? * * *)","StartWindowMinutes":60,
  "CompletionWindowMinutes":180,"Lifecycle":{"DeleteAfterDays":35}}]}'

# Tag the instance so the plan picks it up
aws ec2 create-tags --resources <instance-id> --tags Key=Backup,Value=daily

# PROVE the restore works — a backup you haven't restored is a hope, not a backup
aws backup start-backup-job --backup-vault-name migrated-workloads \
  --resource-arn arn:aws:ec2:$TARGET_REGION:$ACCT:instance/<instance-id> \
  --iam-role-arn arn:aws:iam::$ACCT:role/service-role/AWSBackupDefaultServiceRole
aws backup list-recovery-points-by-backup-vault --backup-vault-name migrated-workloads
aws backup start-restore-job --recovery-point-arn <rp-arn> \
  --iam-role-arn <role> --metadata file:///tmp/restore-metadata.json

# Patching
aws ssm create-maintenance-window --name migrated-patching \
  --schedule "cron(0 2 ? * SUN *)" --duration 3 --cutoff 1 --allow-unassociated-targets
```

### 12.3 Check the cost against the business case

```bash
aws ce get-cost-and-usage --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE --query \
 'ResultsByTime[0].Groups[?Metrics.UnblendedCost.Amount>`0.5`].{Service:Keys[0],Cost:Metrics.UnblendedCost.Amount}' --output table

aws ce get-cost-and-usage --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost --group-by Type=TAG,Key=Wave

aws ce get-savings-plans-purchase-recommendation --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR --payment-option NO_UPFRONT --lookback-period-in-days THIRTY_DAYS \
  --query 'SavingsPlansPurchaseRecommendation.SavingsPlansPurchaseRecommendationSummary'
```

### 12.4 Decommission the source (Retire pattern in action)

```bash
export AWS_REGION=$SOURCE_REGION

# 1. Final archive
aws ec2 create-snapshots --instance-specification InstanceId=<web01-id> \
  --description "final-archive-web01-$(date +%F)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=RetainUntil,Value=2027-07-30},{Key=Reason,Value=migration-archive}]'

# 2. Export any data the business may need
ssh db01-onprem 'mysqldump --all-databases | gzip' > /tmp/db01-final.sql.gz
aws s3 cp /tmp/db01-final.sql.gz s3://migration-archive-$ACCT/retired/db01/ --storage-class DEEP_ARCHIVE

# 3. Remove agents
ssh web01-onprem 'sudo /var/lib/aws-replication-agent/uninstall.sh'
ssh web01-onprem 'sudo /opt/aws/discovery/uninstall'

# 4. Stop (dark period) — do NOT terminate yet
aws ec2 stop-instances --instance-ids <web01-id> <db01-id> <fs01-id> <old-reports01-id>
aws ec2 create-tags --resources <web01-id> \
  --tags Key=Status,Value=decommissioned Key=DeleteAfter,Value=$(date -d '+45 days' +%F)
```

Then the paperwork that actually captures the savings:

```
[ ] Monitoring/backup jobs pointing at the source disabled
[ ] Old DNS records removed
[ ] Firewall rules for the old servers removed
[ ] Licences reclaimed, support contracts cancelled
[ ] CMDB and diagrams updated
[ ] Finance notified so the DC cost line actually drops
```

### 12.5 MGN and DMS teardown

```bash
export AWS_REGION=$TARGET_REGION

aws mgn finalize-cutover --source-server-id $SID
aws mgn disconnect-from-service --source-server-id $SID
aws mgn delete-source-server --source-server-id $SID

aws dms stop-replication-task --replication-task-arn $TASK_ARN
aws dms delete-replication-task --replication-task-arn $TASK_ARN
aws dms delete-endpoint --endpoint-arn $SRC_ARN
aws dms delete-endpoint --endpoint-arn $TGT_ARN
aws dms delete-replication-instance --replication-instance-arn $RI_ARN

aws datasync delete-task --task-arn $TASK
aws datasync delete-location --location-arn $SRC_LOC
aws datasync delete-location --location-arn $EFS_LOC
aws datasync delete-agent --agent-arn $AGENT_ARN
```

### 12.6 Full lab cleanup (do this or you'll get a bill)

```bash
# ---- TARGET REGION ----
export AWS_REGION=$TARGET_REGION

aws ecs update-service --cluster migration-cluster --service orders-api --desired-count 0
aws ecs delete-service --cluster migration-cluster --service orders-api --force
aws ecs delete-cluster --cluster migration-cluster
aws ecr delete-repository --repository-name orders-api --force

aws elbv2 delete-listener --listener-arn <listener-arn>
aws elbv2 delete-load-balancer --load-balancer-arn <alb-arn>
aws elbv2 delete-target-group --target-group-arn $EC2_TG
aws elbv2 delete-target-group --target-group-arn $ECS_TG

aws rds delete-db-instance --db-instance-identifier appdb-target \
  --skip-final-snapshot --delete-automated-backups
aws rds wait db-instance-deleted --db-instance-identifier appdb-target
aws rds delete-db-subnet-group --db-subnet-group-name db-private
aws rds delete-db-parameter-group --db-parameter-group-name mysql80-migration

for MT in $(aws efs describe-mount-targets --file-system-id $EFS_ID --query 'MountTargets[].MountTargetId' --output text); do
  aws efs delete-mount-target --mount-target-id $MT; done
sleep 60
aws efs delete-file-system --file-system-id $EFS_ID

aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances \
  --filters Name=vpc-id,Values=$VPC Name=instance-state-name,Values=running,stopped \
  --query 'Reservations[].Instances[].InstanceId' --output text)
aws ec2 wait instance-terminated --instance-ids <ids>

for EP in $(aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$VPC \
  --query 'VpcEndpoints[].VpcEndpointId' --output text); do
  aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $EP; done

aws ec2 delete-nat-gateway --nat-gateway-id $NAT
sleep 120
aws ec2 release-address --allocation-id $EIP
aws ec2 detach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC
aws ec2 delete-internet-gateway --internet-gateway-id $IGW
for S in $PUB_A $PUB_B $APP_A $APP_B $DAT_A $DAT_B $STG_A; do aws ec2 delete-subnet --subnet-id $S; done
aws ec2 delete-vpc --vpc-id $VPC

# Orphans left behind by MGN
aws ec2 describe-volumes --filters Name=status,Values=available --query 'Volumes[].VolumeId' --output text
aws ec2 describe-snapshots --owner-id self --query 'Snapshots[].SnapshotId' --output text

# ---- SOURCE REGION ----
export AWS_REGION=$SOURCE_REGION
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances \
  --filters Name=vpc-id,Values=$SRC_VPC --query 'Reservations[].Instances[].InstanceId' --output text)
# then subnets, IGW, route tables, SGs, VPC as above

# ---- IAM ----
for U in ads-agent-user mgn-agent-user; do
  for K in $(aws iam list-access-keys --user-name $U --query 'AccessKeyMetadata[].AccessKeyId' --output text); do
    aws iam delete-access-key --user-name $U --access-key-id $K; done
  for P in $(aws iam list-attached-user-policies --user-name $U --query 'AttachedPolicies[].PolicyArn' --output text); do
    aws iam detach-user-policy --user-name $U --policy-arn $P; done
  aws iam delete-user --user-name $U
done

# ---- FINAL CHECK: is anything still running? ----
for R in $SOURCE_REGION $TARGET_REGION; do
  echo "=== $R ==="
  aws ec2 describe-instances --region $R --filters Name=instance-state-name,Values=running \
    --query 'Reservations[].Instances[].[InstanceId,InstanceType]' --output text
  aws rds describe-db-instances --region $R --query 'DBInstances[].DBInstanceIdentifier' --output text
  aws dms describe-replication-instances --region $R --query 'ReplicationInstances[].ReplicationInstanceIdentifier' --output text
  aws elbv2 describe-load-balancers --region $R --query 'LoadBalancers[].LoadBalancerName' --output text
  aws ec2 describe-nat-gateways --region $R --filter Name=state,Values=available --query 'NatGateways[].NatGatewayId' --output text
done
aws ce get-cost-and-usage --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost --query 'ResultsByTime[0].Total'
```

### ✅ Lab 12 checkpoint

- [ ] Right-sizing recommendations reviewed and applied
- [ ] Backup plan applied **and a restore tested**
- [ ] Actual cost compared against your Lab 2 estimate
- [ ] Source archived, powered off, and in a documented dark period
- [ ] Every lab resource deleted; the final check returns nothing
- [ ] Budget dashboard shows spend flattening out

---

## What to do next

You've now done every R at least once:

| R | Where you did it |
|---|---|
| Retire | Lab 2 (identified), Lab 12 (executed) |
| Retain | Lab 2 (the decision and the review date) |
| Rehost | Labs 4 & 5 |
| Replatform | Labs 6, 7 & 8 |
| Repurchase | Discussed — try migrating a mailbox to M365 for real practice |
| Refactor | Lab 10 |
| Relocate | Lab 9 discussion — needs a VMware environment |

**Build on it:**
1. Automate Lab 3 with Terraform or CloudFormation. A landing zone you build by hand once is a landing zone you rebuild by hand forever.
2. Write the MGN wave process as an SSM Automation runbook so a wave becomes one command.
3. Add a real Site-to-Site VPN between your two "sites" and switch MGN to `PRIVATE_IP` routing.
4. Do Lab 6 again but heterogeneous: MySQL → Aurora PostgreSQL, and feel the difference.
5. Break things deliberately: kill the agent mid-sync, fill the staging disk, revoke an IAM permission — then fix it. That's how [troubleshooting.md](troubleshooting.md) becomes muscle memory.

---

*Back to → [README.md](README.md) · [commands-cheatsheet.md](commands-cheatsheet.md) · Next → [troubleshooting.md](troubleshooting.md)*
