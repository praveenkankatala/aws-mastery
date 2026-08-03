# AWS VPC — Hands-On Labs

> Twelve labs that take you from an empty AWS account to a production-shaped, multi-AZ, hardened VPC — then tear it all down cleanly.
> Everything here is designed to be typed, broken, and fixed. That's how networking sticks.

---

## Before You Start

### What you need

- An AWS account with admin (or `AmazonVPCFullAccess` + `AmazonEC2FullAccess` + `IAMFullAccess`)
- AWS CLI v2 configured (`aws sts get-caller-identity` should work)
- A terminal; `jq` helps but isn't required
- **A billing alarm.** Set one now.

### 💰 Cost warning — read this

Most of this is Free Tier eligible. These are **not**:

| Resource | Approx. cost | Labs affected |
|---|---|---|
| NAT Gateway | ~$0.045/hr + $0.045/GB | 5, 6, 7, 9, 10 |
| Interface VPC Endpoint | ~$0.01/hr per AZ | 7 |
| Transit Gateway attachment | ~$0.05/hr each | 9 |
| Elastic IP (unattached or attached) | ~$0.005/hr | 5+ |
| Public IPv4 on an instance | ~$0.005/hr | 3+ |

**Running Labs 1–10 continuously for a week costs roughly $30–50.** Do a lab, verify it, then run the teardown in §Lab 12 or delete the specific expensive resource. **Never leave a NAT Gateway or Transit Gateway running overnight by accident.**

### Naming convention used throughout

```
Region       : us-east-1
VPC          : lab-vpc          10.0.0.0/16
Subnets      : lab-public-1a    10.0.0.0/24     us-east-1a
               lab-public-1b    10.0.1.0/24     us-east-1b
               lab-app-1a       10.0.10.0/24    us-east-1a
               lab-app-1b       10.0.11.0/24    us-east-1b
               lab-db-1a        10.0.20.0/24    us-east-1a
               lab-db-1b        10.0.21.0/24    us-east-1b
```

Small `/24`s deliberately — easier to read, and you'll get to feel IP exhaustion in Lab 12.

### Set up your shell

```bash
export AWS_REGION=us-east-1
export AZ1=us-east-1a
export AZ2=us-east-1b
export PREFIX=lab

# Save IDs as you go so you can re-source them:
touch ~/lab-vars.sh
# add lines like: export VPC_ID=vpc-0abc
# then: source ~/lab-vars.sh
```

---

## Lab Index

| # | Lab | Time | Cost | Teaches |
|---|---|---|---|---|
| 1 | Build the VPC and subnets | 15 min | Free | CIDR, AZs, subnet design |
| 2 | Internet Gateway + public subnet | 15 min | ~$0 | Routing, what "public" means |
| 3 | Launch a web server, see it work | 20 min | Free tier | SGs, user data, verification |
| 4 | Private subnet + a bastion-free admin path | 25 min | Free tier | Session Manager, IAM roles |
| 5 | NAT Gateway for private egress | 20 min | **$$** | SNAT, HA per AZ |
| 6 | Security Groups vs NACLs — break it on purpose | 30 min | Free | Stateful vs stateless, ephemeral ports |
| 7 | VPC Endpoints — remove the internet entirely | 30 min | **$$** | Gateway vs interface, endpoint policy |
| 8 | VPC Peering between two VPCs | 25 min | Free | Peering, routes both sides, non-transitivity |
| 9 | Transit Gateway hub-and-spoke | 40 min | **$$$** | Attachments, TGW route tables, segmentation |
| 10 | Flow Logs + Reachability Analyzer | 30 min | ~$ | Observability, root-cause tooling |
| 11 | Rebuild everything as Terraform + CloudFormation | 45 min | Same as 1–7 | IaC, reproducibility |
| 12 | Break/fix drills + full teardown | 45 min | — | Diagnosis under pressure |

---

# Lab 1 — Build the VPC and Subnets

**Goal:** create a VPC with six subnets across two AZs, and understand exactly what you've built.

### 1.1 Create the VPC

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$PREFIX-vpc}]" \
  --query 'Vpc.VpcId' --output text)

echo "export VPC_ID=$VPC_ID" >> ~/lab-vars.sh
echo "Created $VPC_ID"
```

### 1.2 Turn on DNS hostnames

Custom VPCs have `enableDnsHostnames = false` by default. Many things (private hosted zones, interface endpoint private DNS, RDS endpoints) quietly misbehave without it.

```bash
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support   '{"Value":true}'
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'

# Verify
aws ec2 describe-vpc-attribute --vpc-id $VPC_ID --attribute enableDnsHostnames
```

### 1.3 Create six subnets

```bash
mk_subnet () {  # $1=cidr  $2=az  $3=name  $4=tier
  aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block $1 --availability-zone $2 \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$3},{Key=Tier,Value=$4}]" \
    --query 'Subnet.SubnetId' --output text
}

PUB1=$(mk_subnet 10.0.0.0/24  $AZ1 $PREFIX-public-1a public)
PUB2=$(mk_subnet 10.0.1.0/24  $AZ2 $PREFIX-public-1b public)
APP1=$(mk_subnet 10.0.10.0/24 $AZ1 $PREFIX-app-1a    app)
APP2=$(mk_subnet 10.0.11.0/24 $AZ2 $PREFIX-app-1b    app)
DB1=$(mk_subnet  10.0.20.0/24 $AZ1 $PREFIX-db-1a     db)
DB2=$(mk_subnet  10.0.21.0/24 $AZ2 $PREFIX-db-1b     db)

cat >> ~/lab-vars.sh <<EOF
export PUB1=$PUB1 PUB2=$PUB2
export APP1=$APP1 APP2=$APP2
export DB1=$DB1  DB2=$DB2
EOF
```

### ✅ Verify

```bash
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query \
 'Subnets[].{Name:Tags[?Key==`Name`]|[0].Value,CIDR:CidrBlock,AZ:AvailabilityZone,Free:AvailableIpAddressCount}' \
 --output table
```

You should see six rows, each with **251 free IPs** — not 256.

### 🧠 Understand what you just built

**Question 1:** Why 251 and not 256?
<details><summary>Answer</summary>
AWS reserves 5 addresses in every subnet: `.0` network, `.1` VPC router, `.2` DNS resolver, `.3` reserved for future use, `.255` broadcast.
</details>

**Question 2:** Are any of these subnets public right now?
<details><summary>Answer</summary>
No. None of them. There's no Internet Gateway and no route table with a `0.0.0.0/0` route. The names are just tags — labels for humans. Right now all six subnets are functionally identical.
</details>

**Question 3:** What route table are these subnets using?
<details><summary>Answer</summary>
The VPC **main** route table, created implicitly with the VPC. It has only the `local` route. Check:
```bash
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[].{RTB:RouteTableId,Main:Associations[0].Main,Routes:Routes}'
```
</details>

---

# Lab 2 — Internet Gateway and a Real Public Subnet

**Goal:** attach an IGW and turn `lab-public-*` into genuinely public subnets.

### 2.1 Create and attach the IGW

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=$PREFIX-igw}]" \
  --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
echo "export IGW_ID=$IGW_ID" >> ~/lab-vars.sh
```

### 2.2 Public route table

```bash
RT_PUB=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=$PREFIX-rt-public}]" \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $RT_PUB \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $PUB1
aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $PUB2

echo "export RT_PUB=$RT_PUB" >> ~/lab-vars.sh
```

### 2.3 Auto-assign public IPs in those subnets

```bash
aws ec2 modify-subnet-attribute --subnet-id $PUB1 --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $PUB2 --map-public-ip-on-launch
```

### ✅ Verify

```bash
aws ec2 describe-route-tables --route-table-ids $RT_PUB --query \
 'RouteTables[].Routes[].[DestinationCidrBlock,GatewayId,State]' --output table
```

Expected:

```
|  10.0.0.0/16  |  local        |  active  |
|  0.0.0.0/0    |  igw-0abc...  |  active  |
```

### 🧠 Understand

**The four conditions for internet access.** Write them down from memory, then check:

<details><summary>Answer</summary>

1. IGW attached to the VPC ✅ (done in 2.1)
2. Route table has `0.0.0.0/0 → igw` ✅ (2.2)
3. Instance has a public IP or EIP ✅ (2.3 makes this automatic)
4. Security group + NACL allow the traffic — coming in Lab 3

</details>

**What happens to the app and db subnets?** Nothing. They're still on the main route table with only `local`. They're private — by omission, which is exactly how it should work.

---

# Lab 3 — Launch a Web Server and Prove It Works

**Goal:** a real EC2 instance serving HTTP from the public subnet.

### 3.1 Security group for the web tier

```bash
SG_WEB=$(aws ec2 create-security-group --group-name $PREFIX-sg-web \
  --description "Public web tier" --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=$PREFIX-sg-web}]" \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_WEB \
  --ip-permissions 'IpProtocol=tcp,FromPort=80,ToPort=80,IpRanges=[{CidrIp=0.0.0.0/0,Description="public HTTP"}]'

echo "export SG_WEB=$SG_WEB" >> ~/lab-vars.sh
```

> Notice we did **not** open port 22. We'll use Session Manager instead. If you insist on SSH, restrict it to your own IP:
> ```bash
> MYIP=$(curl -s https://checkip.amazonaws.com)
> aws ec2 authorize-security-group-ingress --group-id $SG_WEB --protocol tcp --port 22 --cidr $MYIP/32
> ```

### 3.2 IAM role for Session Manager

```bash
cat > trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF

aws iam create-role --role-name LabSSMRole --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name LabSSMRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam create-instance-profile --instance-profile-name LabSSMProfile
aws iam add-role-to-instance-profile --instance-profile-name LabSSMProfile --role-name LabSSMRole
```

### 3.3 Launch the instance

```bash
AMI=$(aws ssm get-parameters --names \
  /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameters[0].Value' --output text)

cat > userdata.sh <<'EOF'
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 600")
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4)
echo "<h1>Hello from $AZ</h1><p>Private IP: $IP</p>" > /var/www/html/index.html
EOF

WEB_ID=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --subnet-id $PUB1 --security-group-ids $SG_WEB \
  --iam-instance-profile Name=LabSSMProfile \
  --user-data file://userdata.sh \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$PREFIX-web-1a}]" \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $WEB_ID
echo "export WEB_ID=$WEB_ID" >> ~/lab-vars.sh
```

### ✅ Verify

```bash
PUB_IP=$(aws ec2 describe-instances --instance-ids $WEB_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "http://$PUB_IP"

sleep 60   # let user-data finish
curl -s http://$PUB_IP
```

Expected: `<h1>Hello from us-east-1a</h1>`

### 3.4 Get a shell without SSH

```bash
aws ssm start-session --target $WEB_ID
```

Inside the instance:

```bash
ip addr show                      # note: NO public IP here — only 10.0.0.x
curl -s https://checkip.amazonaws.com   # returns the instance's public IP
ip route                          # default via 10.0.0.1 (the VPC router)
```

### 🧠 Understand

**Why doesn't `ip addr` show the public IP?** Because the Internet Gateway performs the 1:1 NAT. The instance's OS genuinely never sees its public address. This trips up software that tries to auto-detect its own external IP.

### 🔬 Experiment: break it, then fix it

```bash
# Break: remove the internet route
aws ec2 delete-route --route-table-id $RT_PUB --destination-cidr-block 0.0.0.0/0
curl -m 5 http://$PUB_IP        # → times out

# Fix
aws ec2 create-route --route-table-id $RT_PUB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
curl -s http://$PUB_IP          # → works again
```

That's the entire mechanism of a "public subnet" demonstrated in two commands.

---

# Lab 4 — Private Subnet with No Internet at All

**Goal:** launch an instance in `lab-app-1a`, confirm it has zero internet access, and still administer it.

### 4.1 App-tier security group (referencing the web SG)

```bash
SG_APP=$(aws ec2 create-security-group --group-name $PREFIX-sg-app \
  --description "Private app tier" --vpc-id $VPC_ID --query GroupId --output text)

# Only accept 8080 from the web tier — no CIDRs anywhere
aws ec2 authorize-security-group-ingress --group-id $SG_APP \
  --ip-permissions "IpProtocol=tcp,FromPort=8080,ToPort=8080,UserIdGroupPairs=[{GroupId=$SG_WEB,Description=\"from web tier\"}]"

echo "export SG_APP=$SG_APP" >> ~/lab-vars.sh
```

### 4.2 Launch a private instance

```bash
APP_INSTANCE=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --subnet-id $APP1 --security-group-ids $SG_APP \
  --iam-instance-profile Name=LabSSMProfile \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$PREFIX-app-1a}]" \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $APP_INSTANCE
echo "export APP_INSTANCE=$APP_INSTANCE" >> ~/lab-vars.sh
```

### ✅ Verify it's genuinely private

```bash
aws ec2 describe-instances --instance-ids $APP_INSTANCE \
  --query 'Reservations[0].Instances[0].[PublicIpAddress,PrivateIpAddress]'
```

Expected: `[null, "10.0.10.x"]` — **no public IP**.

### 4.3 Try to reach it

```bash
aws ssm start-session --target $APP_INSTANCE
```

**This will fail.** Error: `TargetNotConnected`.

**Why?** The SSM Agent needs to reach `ssm.us-east-1.amazonaws.com` over HTTPS to register. This instance has no internet route and no VPC endpoints. It's completely isolated.

This is the correct security posture — but it's also completely unmanageable. Labs 5 and 7 give you the two ways to fix it.

### 🧠 Understand

You've just built the fundamental tension of cloud networking: **isolation vs manageability**. Every design decision from here is about resolving it without opening an inbound path.

Two solutions exist:
- **Lab 5 — NAT Gateway:** give it outbound internet (simple, costs money per GB)
- **Lab 7 — VPC Endpoints:** give it a private path to just the AWS APIs it needs (more secure, no internet at all)

Production usually uses both.

---

# Lab 5 — NAT Gateway for Private Egress

**Goal:** let private instances reach the internet outbound-only, with per-AZ HA.

> 💰 **This lab starts costing money.** ~$0.045/hr per NAT Gateway. Two of them = ~$2.20/day. Delete them at the end of the lab if you're not continuing immediately.

### 5.1 One NAT Gateway per AZ

```bash
for i in 1 2; do
  eval PUB=\$PUB$i
  ALLOC=$(aws ec2 allocate-address --domain vpc \
    --tag-specifications "ResourceType=elastic-ip,Tags=[{Key=Name,Value=$PREFIX-eip-nat-$i}]" \
    --query AllocationId --output text)
  NAT=$(aws ec2 create-nat-gateway --subnet-id $PUB --allocation-id $ALLOC \
    --tag-specifications "ResourceType=natgateway,Tags=[{Key=Name,Value=$PREFIX-nat-$i}]" \
    --query 'NatGateway.NatGatewayId' --output text)
  eval NAT$i=$NAT
  eval ALLOC$i=$ALLOC
  echo "export NAT$i=$NAT ALLOC$i=$ALLOC" >> ~/lab-vars.sh
  echo "Created $NAT in $PUB"
done

aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT1 $NAT2
```

> **The #1 mistake:** creating the NAT Gateway in a *private* subnet. It must be in a subnet whose route table points at the IGW. Double-check: `$PUB1` and `$PUB2` are associated with `$RT_PUB`, which has the `0.0.0.0/0 → igw` route. ✅

### 5.2 A private route table **per AZ**

This is the part people get wrong. Both private subnets routing to one NAT Gateway means an AZ failure takes out both, *and* you pay cross-AZ transfer for every byte.

```bash
for i in 1 2; do
  eval APP=\$APP$i; eval NAT=\$NAT$i
  RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=$PREFIX-rt-app-$i}]" \
    --query 'RouteTable.RouteTableId' --output text)
  aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT
  aws ec2 associate-route-table --route-table-id $RT --subnet-id $APP
  eval RT_APP$i=$RT
  echo "export RT_APP$i=$RT" >> ~/lab-vars.sh
done
```

### ✅ Verify

Wait ~2 minutes for the SSM agent to register, then:

```bash
aws ssm start-session --target $APP_INSTANCE
```

Inside:

```bash
curl -s https://checkip.amazonaws.com
```

**This returns the NAT Gateway's Elastic IP** — not the instance's (it has none).

```bash
# Confirm it matches
aws ec2 describe-nat-gateways --nat-gateway-ids $NAT1 \
  --query 'NatGateways[0].NatGatewayAddresses[0].PublicIp' --output text
```

More checks inside the instance:

```bash
sudo dnf update -y            # works — outbound is fine
curl -s https://api.github.com/zen
ip addr show                  # still no public IP
```

### 5.3 Prove it's outbound-only

From your laptop:

```bash
PRIV_IP=$(aws ec2 describe-instances --instance-ids $APP_INSTANCE \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
ping -c 2 $PRIV_IP      # unreachable — it's a private RFC1918 address
```

There is no public address anywhere that maps to this instance. That's the guarantee.

### 🧠 Understand

**What the NAT Gateway did.** It rewrote the source IP on the way out (SNAT) and kept a connection-tracking table so return packets get translated back. Inbound connections have no entry in that table, so they're dropped. Directionality is enforced by state, not by rules.

### 🔬 Experiment: measure the cost of a wrong route

```bash
# Deliberately point AZ-1b's app subnet at AZ-1a's NAT Gateway
aws ec2 replace-route --route-table-id $RT_APP2 --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT1
```

Everything still works — which is exactly why this bug survives in production for years. Every byte now crosses AZs and is billed twice ($0.01/GB transfer + $0.045/GB NAT processing). Put it back:

```bash
aws ec2 replace-route --route-table-id $RT_APP2 --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT2
```

### 🧹 If you're stopping here

```bash
aws ec2 delete-nat-gateway --nat-gateway-id $NAT1
aws ec2 delete-nat-gateway --nat-gateway-id $NAT2
# EIPs are NOT released automatically:
aws ec2 release-address --allocation-id $ALLOC1
aws ec2 release-address --allocation-id $ALLOC2
```

---

# Lab 6 — Security Groups vs NACLs (Break It On Purpose)

**Goal:** feel the difference between stateful and stateless firewalls, and never confuse them again.

### 6.1 Baseline

```bash
curl -s http://$PUB_IP    # should still work from Lab 3
```

### 6.2 Experiment A — the stateful nature of security groups

```bash
# Remove ALL outbound rules from the web SG
aws ec2 revoke-security-group-egress --group-id $SG_WEB --protocol -1 --port -1 --cidr 0.0.0.0/0

curl -s http://$PUB_IP
```

**Result: it still works.** Inbound HTTP requests still get responses even with zero egress rules.

**Why?** Security groups are stateful. The response to an allowed inbound connection is automatically permitted — it doesn't need an egress rule.

```bash
# But the instance can no longer INITIATE anything
aws ssm start-session --target $WEB_ID
# → will eventually fail, because the SSM agent needs outbound 443
```

Restore:

```bash
aws ec2 authorize-security-group-egress --group-id $SG_WEB --protocol -1 --port -1 --cidr 0.0.0.0/0
```

### 6.3 Experiment B — the stateless nature of NACLs

Create a NACL that allows inbound HTTP but forgets the return path.

```bash
NACL=$(aws ec2 create-network-acl --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=network-acl,Tags=[{Key=Name,Value=$PREFIX-nacl-broken}]" \
  --query 'NetworkAcl.NetworkAclId' --output text)

# Inbound: allow HTTP
aws ec2 create-network-acl-entry --network-acl-id $NACL --rule-number 100 \
  --protocol 6 --port-range From=80,To=80 --cidr-block 0.0.0.0/0 --rule-action allow --ingress

# Outbound: allow ONLY port 80 (this is the bug)
aws ec2 create-network-acl-entry --network-acl-id $NACL --rule-number 100 \
  --protocol 6 --port-range From=80,To=80 --cidr-block 0.0.0.0/0 --rule-action allow --egress

# Attach it to the public subnet
ASSOC=$(aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=$PUB1" \
  --query "NetworkAcls[0].Associations[?SubnetId=='$PUB1'].NetworkAclAssociationId" --output text)
aws ec2 replace-network-acl-association --association-id $ASSOC --network-acl-id $NACL
```

Now test:

```bash
curl -m 10 http://$PUB_IP
```

**Result: it hangs and times out.**

**Why?** Your browser connected *from* an ephemeral port (say 51234) *to* port 80. The server's reply goes **from port 80 to port 51234**. The NACL's outbound rule only allows destination port 80 — the reply is dropped. The request arrived perfectly; the answer never left.

### 6.4 Fix it

```bash
aws ec2 create-network-acl-entry --network-acl-id $NACL --rule-number 110 \
  --protocol 6 --port-range From=1024,To=65535 --cidr-block 0.0.0.0/0 --rule-action allow --egress

curl -s http://$PUB_IP     # works now
```

### 6.5 Experiment C — rule ordering

NACLs evaluate **lowest number first and stop at the first match**.

```bash
MYIP=$(curl -s https://checkip.amazonaws.com)

# Rule 50 DENIES your IP — lower number than the rule 100 ALLOW
aws ec2 create-network-acl-entry --network-acl-id $NACL --rule-number 50 \
  --protocol -1 --cidr-block $MYIP/32 --rule-action deny --ingress

curl -m 5 http://$PUB_IP    # blocked — 50 wins over 100
```

Now flip it:

```bash
aws ec2 delete-network-acl-entry --network-acl-id $NACL --rule-number 50 --ingress
aws ec2 create-network-acl-entry --network-acl-id $NACL --rule-number 200 \
  --protocol -1 --cidr-block $MYIP/32 --rule-action deny --ingress

curl -s http://$PUB_IP      # works — rule 100 (allow HTTP) matched first, evaluation stopped
```

**This is the single most important NACL behaviour.** A deny rule placed after an allow rule does nothing.

### 6.6 Restore the default NACL

```bash
DEFAULT_NACL=$(aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC_ID" \
  "Name=default,Values=true" --query 'NetworkAcls[0].NetworkAclId' --output text)
ASSOC=$(aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=$PUB1" \
  --query "NetworkAcls[0].Associations[?SubnetId=='$PUB1'].NetworkAclAssociationId" --output text)
aws ec2 replace-network-acl-association --association-id $ASSOC --network-acl-id $DEFAULT_NACL
aws ec2 delete-network-acl --network-acl-id $NACL
```

### 📋 Summary card

| | Security Group | NACL |
|---|---|---|
| Level | ENI | Subnet |
| State | Stateful — return traffic auto-allowed | Stateless — allow both directions explicitly |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules, order irrelevant | Lowest number first, first match wins |
| Ephemeral ports | Never needed | **Always** needed |
| Best used for | Everyday access control | Coarse subnet-wide blocks, compliance |

---

# Lab 7 — VPC Endpoints: Remove the Internet Entirely

**Goal:** make the private instance fully functional with **no NAT Gateway and no internet route**.

### 7.1 Remove the NAT route from AZ-1a

```bash
aws ec2 delete-route --route-table-id $RT_APP1 --destination-cidr-block 0.0.0.0/0
```

Confirm the instance is now isolated (Session Manager will drop within a few minutes).

### 7.2 Gateway endpoint for S3 — free, instant

```bash
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
  --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.$AWS_REGION.s3 \
  --route-table-ids $RT_APP1 $RT_APP2 \
  --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=$PREFIX-vpce-s3}]"
```

Look at what it did to the route table:

```bash
aws ec2 describe-route-tables --route-table-ids $RT_APP1 --query \
 'RouteTables[].Routes[].[DestinationCidrBlock,DestinationPrefixListId,GatewayId]' --output table
```

You'll see a route with a **prefix list** destination (`pl-63a5400a`) pointing at the `vpce-` ID. That's the entire mechanism — no ENI, no cost, no bandwidth limit.

### 7.3 Security group for interface endpoints

```bash
SG_EP=$(aws ec2 create-security-group --group-name $PREFIX-sg-endpoints \
  --description "VPC interface endpoints" --vpc-id $VPC_ID --query GroupId --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_EP \
  --ip-permissions 'IpProtocol=tcp,FromPort=443,ToPort=443,IpRanges=[{CidrIp=10.0.0.0/16,Description="VPC HTTPS"}]'

echo "export SG_EP=$SG_EP" >> ~/lab-vars.sh
```

> **Forget this and everything times out with no error message.** The endpoint's ENI has a security group; if it doesn't allow 443 from your subnets, nothing works.

### 7.4 Interface endpoints for Session Manager

```bash
for SVC in ssm ssmmessages ec2messages; do
  aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --vpc-endpoint-type Interface \
    --service-name com.amazonaws.$AWS_REGION.$SVC \
    --subnet-ids $APP1 $APP2 --security-group-ids $SG_EP --private-dns-enabled \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=$PREFIX-vpce-$SVC}]"
  echo "created endpoint for $SVC"
done
```

Wait ~3 minutes for them to become `available`:

```bash
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'VpcEndpoints[].[ServiceName,State]' --output table
```

### ✅ Verify

```bash
aws ssm start-session --target $APP_INSTANCE
```

**It connects** — with no internet route in the route table at all. Inside the instance:

```bash
ip route                                        # NO 0.0.0.0/0 entry
curl -m 5 https://www.google.com                # fails — genuinely no internet
dig +short ssm.us-east-1.amazonaws.com          # returns a 10.0.x.x PRIVATE IP ← the magic
aws s3 ls                                        # works, via the gateway endpoint
```

That `dig` result is the whole point of private DNS on an interface endpoint: your SDK code is unchanged, but the hostname resolves to an ENI inside your subnet.

### 7.5 Endpoint policies — network-layer data-loss prevention

```bash
VPCE_S3=$(aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC_ID" \
  "Name=service-name,Values=com.amazonaws.$AWS_REGION.s3" \
  --query 'VpcEndpoints[0].VpcEndpointId' --output text)

cat > s3-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowApprovedBucketOnly",
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["s3:GetObject","s3:PutObject","s3:ListBucket"],
    "Resource": ["arn:aws:s3:::my-approved-bucket","arn:aws:s3:::my-approved-bucket/*"]
  }]
}
EOF

aws ec2 modify-vpc-endpoint --vpc-endpoint-id $VPCE_S3 --policy-document file://s3-policy.json
```

Now `aws s3 ls` fails from the instance (it can't list all buckets), but that one bucket works. Even an instance with full `s3:*` IAM permissions can only reach approved buckets — the network says no. This is how you stop data exfiltration to a personal bucket.

Reset when done:

```bash
aws ec2 modify-vpc-endpoint --vpc-endpoint-id $VPCE_S3 --reset-policy
```

### 🧠 Understand

| | Gateway endpoint | Interface endpoint |
|---|---|---|
| Implemented as | Route table entry + prefix list | ENI with a private IP |
| Cost | **Free** | ~$0.01/hr/AZ + per-GB |
| Works from on-prem / peered VPC | ❌ | ✅ |
| Security group | ❌ | ✅ |
| Services | S3, DynamoDB only | 100+ |

**Cost note:** three interface endpoints × two AZs = six ENIs ≈ $43/month. Cheaper than a NAT Gateway if you only need AWS APIs; more expensive if you barely use them. Measure before choosing.

---

# Lab 8 — VPC Peering

**Goal:** connect two VPCs, prove routing works both ways, and prove peering isn't transitive.

### 8.1 Create a second VPC

```bash
VPC_B=$(aws ec2 create-vpc --cidr-block 10.1.0.0/16 \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$PREFIX-vpc-b}]" \
  --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_B --enable-dns-hostnames '{"Value":true}'

SUB_B=$(aws ec2 create-subnet --vpc-id $VPC_B --cidr-block 10.1.0.0/24 \
  --availability-zone $AZ1 \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$PREFIX-b-1a}]" \
  --query 'Subnet.SubnetId' --output text)

SG_B=$(aws ec2 create-security-group --group-name $PREFIX-sg-b \
  --description "VPC B" --vpc-id $VPC_B --query GroupId --output text)

# Allow ICMP and SSH from VPC A's CIDR
aws ec2 authorize-security-group-ingress --group-id $SG_B --protocol icmp --port -1 --cidr 10.0.0.0/16

echo "export VPC_B=$VPC_B SUB_B=$SUB_B SG_B=$SG_B" >> ~/lab-vars.sh
```

### 8.2 Create and accept the peering

```bash
PCX=$(aws ec2 create-vpc-peering-connection --vpc-id $VPC_ID --peer-vpc-id $VPC_B \
  --tag-specifications "ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=$PREFIX-pcx-a-b}]" \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PCX
echo "export PCX=$PCX" >> ~/lab-vars.sh
```

### 8.3 Routes — on **both** sides

```bash
# VPC A → VPC B
aws ec2 create-route --route-table-id $RT_APP1 --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id $PCX

# VPC B → VPC A  (using VPC B's main route table)
RT_B=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_B" \
  "Name=association.main,Values=true" --query 'RouteTables[0].RouteTableId' --output text)
aws ec2 create-route --route-table-id $RT_B --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id $PCX
```

> **Forgetting the return route is the #1 peering bug.** Traffic goes out, nothing comes back, and it looks like a firewall problem.

### 8.4 Launch an instance in VPC B and test

```bash
B_INSTANCE=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --subnet-id $SUB_B --security-group-ids $SG_B \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$PREFIX-b-host}]" \
  --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-running --instance-ids $B_INSTANCE

B_IP=$(aws ec2 describe-instances --instance-ids $B_INSTANCE \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
echo "VPC B host: $B_IP"
```

Also allow ICMP outbound from the app SG (it allows all egress by default, so nothing to do) and inbound ICMP from VPC B on `$SG_APP`:

```bash
aws ec2 authorize-security-group-ingress --group-id $SG_APP --protocol icmp --port -1 --cidr 10.1.0.0/16
```

### ✅ Verify

```bash
aws ssm start-session --target $APP_INSTANCE
# inside:
ping -c 4 10.1.0.x        # replace with $B_IP → should succeed
```

### 8.5 Prove non-transitivity

```bash
# From VPC B's host, try to reach the internet via VPC A's NAT Gateway
# (route added on VPC B pointing 0.0.0.0/0 at the peering — this will NOT work)
aws ec2 create-route --route-table-id $RT_B --destination-cidr-block 0.0.0.0/0 --vpc-peering-connection-id $PCX
```

**Result:** AWS may accept the route, but traffic is dropped. Using another VPC's IGW, NAT Gateway, VPN, or gateway endpoint across a peering is **edge-to-edge routing**, and it's explicitly unsupported.

```bash
aws ec2 delete-route --route-table-id $RT_B --destination-cidr-block 0.0.0.0/0
```

### 🧠 Understand

- Peering has **no bandwidth limit, no single point of failure, and no hourly charge** — only data transfer.
- It is **not transitive**. A↔B and B↔C never gives A↔C.
- Full mesh scaling: **n(n-1)/2**. Ten VPCs need 45 peerings. That's Lab 9's motivation.
- **CIDRs must not overlap.** Try creating a peering to a VPC that also uses `10.0.0.0/16` and watch it fail immediately.

---

# Lab 9 — Transit Gateway Hub-and-Spoke

**Goal:** replace the peering mesh with a hub, then use TGW route tables to enforce segmentation.

> 💰 **~$0.05/hr per attachment.** Three attachments ≈ $3.60/day. Do this lab in one sitting and tear it down.

### 9.1 Create a third VPC (a "dev" spoke)

```bash
VPC_C=$(aws ec2 create-vpc --cidr-block 10.2.0.0/16 \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$PREFIX-vpc-c-dev}]" \
  --query 'Vpc.VpcId' --output text)
SUB_C=$(aws ec2 create-subnet --vpc-id $VPC_C --cidr-block 10.2.0.0/24 \
  --availability-zone $AZ1 --query 'Subnet.SubnetId' --output text)
echo "export VPC_C=$VPC_C SUB_C=$SUB_C" >> ~/lab-vars.sh
```

### 9.2 Create the Transit Gateway with manual routing

```bash
TGW=$(aws ec2 create-transit-gateway --description "$PREFIX hub" \
  --options 'AmazonSideAsn=64512,DefaultRouteTableAssociation=disable,DefaultRouteTablePropagation=disable,DnsSupport=enable' \
  --tag-specifications "ResourceType=transit-gateway,Tags=[{Key=Name,Value=$PREFIX-tgw}]" \
  --query 'TransitGateway.TransitGatewayId' --output text)

# Wait for available (usually ~5 minutes)
while [ "$(aws ec2 describe-transit-gateways --transit-gateway-ids $TGW \
  --query 'TransitGateways[0].State' --output text)" != "available" ]; do
  echo "waiting..."; sleep 30
done
echo "export TGW=$TGW" >> ~/lab-vars.sh
```

Disabling default association/propagation is what lets you build real segmentation.

### 9.3 Attach the three VPCs

```bash
ATT_A=$(aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id $TGW \
  --vpc-id $VPC_ID --subnet-ids $APP1 \
  --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=att-a-prod}]" \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)

ATT_B=$(aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id $TGW \
  --vpc-id $VPC_B --subnet-ids $SUB_B \
  --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=att-b-shared}]" \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)

ATT_C=$(aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id $TGW \
  --vpc-id $VPC_C --subnet-ids $SUB_C \
  --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=att-c-dev}]" \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)

echo "export ATT_A=$ATT_A ATT_B=$ATT_B ATT_C=$ATT_C" >> ~/lab-vars.sh
sleep 120   # attachments take a couple of minutes
```

### 9.4 Build two TGW route tables for segmentation

**Design goal:**
- Prod (A) ↔ Shared (B) ✅
- Dev (C) ↔ Shared (B) ✅
- Prod (A) ↔ Dev (C) ❌ **blocked**

```bash
RTB_PROD=$(aws ec2 create-transit-gateway-route-table --transit-gateway-id $TGW \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=tgw-rtb-prod}]' \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' --output text)

RTB_DEV=$(aws ec2 create-transit-gateway-route-table --transit-gateway-id $TGW \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=tgw-rtb-dev}]' \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' --output text)

sleep 30

# ASSOCIATIONS — which table each attachment USES
aws ec2 associate-transit-gateway-route-table --transit-gateway-route-table-id $RTB_PROD --transit-gateway-attachment-id $ATT_A
aws ec2 associate-transit-gateway-route-table --transit-gateway-route-table-id $RTB_PROD --transit-gateway-attachment-id $ATT_B
aws ec2 associate-transit-gateway-route-table --transit-gateway-route-table-id $RTB_DEV  --transit-gateway-attachment-id $ATT_C

# PROPAGATIONS — which tables LEARN each attachment's routes
aws ec2 enable-transit-gateway-route-table-propagation --transit-gateway-route-table-id $RTB_PROD --transit-gateway-attachment-id $ATT_A
aws ec2 enable-transit-gateway-route-table-propagation --transit-gateway-route-table-id $RTB_PROD --transit-gateway-attachment-id $ATT_B
aws ec2 enable-transit-gateway-route-table-propagation --transit-gateway-route-table-id $RTB_DEV  --transit-gateway-attachment-id $ATT_B
aws ec2 enable-transit-gateway-route-table-propagation --transit-gateway-route-table-id $RTB_DEV  --transit-gateway-attachment-id $ATT_C
# NOTE: prod (ATT_A) is NOT propagated into RTB_DEV, and dev (ATT_C) is NOT propagated into RTB_PROD
```

**The mental model:**
- **Association** = "when this attachment sends traffic, which routing table does it consult?" (exactly one)
- **Propagation** = "which routing tables get to see this attachment's CIDRs?" (many)

Segmentation is built entirely from these two knobs.

### 9.5 VPC route tables must point at the TGW

```bash
aws ec2 create-route --route-table-id $RT_APP1 --destination-cidr-block 10.0.0.0/8 --transit-gateway-id $TGW
aws ec2 create-route --route-table-id $RT_B    --destination-cidr-block 10.0.0.0/8 --transit-gateway-id $TGW
RT_C=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_C" \
  "Name=association.main,Values=true" --query 'RouteTables[0].RouteTableId' --output text)
aws ec2 create-route --route-table-id $RT_C --destination-cidr-block 10.0.0.0/8 --transit-gateway-id $TGW
```

> Remove the Lab 8 peering routes first if they conflict — the more specific `/16` peering route would win over the `/8` TGW route.

### ✅ Verify the segmentation

```bash
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id $RTB_PROD \
  --filters "Name=state,Values=active" --query 'Routes[].[DestinationCidrBlock,Type]' --output table
```

`RTB_PROD` should list `10.0.0.0/16` and `10.1.0.0/16` — **but not** `10.2.0.0/16`.

```bash
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id $RTB_DEV \
  --filters "Name=state,Values=active" --query 'Routes[].[DestinationCidrBlock,Type]' --output table
```

`RTB_DEV` should list `10.1.0.0/16` and `10.2.0.0/16` — **but not** `10.0.0.0/16`.

Test with pings from instances in each VPC. Prod→Shared works; Prod→Dev does not.

### 🧹 Tear down Lab 9 immediately

```bash
for A in $ATT_A $ATT_B $ATT_C; do
  aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id $A
done
sleep 180
aws ec2 delete-transit-gateway-route-table --transit-gateway-route-table-id $RTB_PROD
aws ec2 delete-transit-gateway-route-table --transit-gateway-route-table-id $RTB_DEV
aws ec2 delete-transit-gateway --transit-gateway-id $TGW
```

---

# Lab 10 — Flow Logs and Reachability Analyzer

**Goal:** build the tooling that answers "why can't A reach B?" without guessing.

### 10.1 Flow Logs to CloudWatch

```bash
# IAM role for the delivery
cat > flowlogs-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"vpc-flow-logs.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF

aws iam create-role --role-name LabFlowLogsRole --assume-role-policy-document file://flowlogs-trust.json

cat > flowlogs-policy.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":[
 "logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents",
 "logs:DescribeLogGroups","logs:DescribeLogStreams"],"Resource":"*"}]}
EOF

aws iam put-role-policy --role-name LabFlowLogsRole \
  --policy-name FlowLogsDelivery --policy-document file://flowlogs-policy.json

ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
aws logs create-log-group --log-group-name /aws/vpc/$PREFIX-flowlogs

aws ec2 create-flow-logs --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type ALL --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/$PREFIX-flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::$ACCOUNT:role/LabFlowLogsRole \
  --max-aggregation-interval 60 \
  --log-format '${version} ${vpc-id} ${subnet-id} ${instance-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${tcp-flags} ${flow-direction} ${pkt-srcaddr} ${pkt-dstaddr} ${log-status}'
```

### 10.2 Generate traffic — including rejected traffic

```bash
curl -s http://$PUB_IP > /dev/null              # ACCEPT
curl -m 3 http://$PUB_IP:9999 || true            # REJECT — nothing listening / SG blocks it
nc -zv -w 2 $PUB_IP 3306 || true                 # REJECT
```

Wait 2–3 minutes for delivery.

### 10.3 Query the logs

```bash
QID=$(aws logs start-query \
  --log-group-name /aws/vpc/$PREFIX-flowlogs \
  --start-time $(($(date +%s) - 900)) --end-time $(date +%s) \
  --query-string 'fields @timestamp, srcaddr, dstaddr, dstport, action, tcp_flags
                  | filter action = "REJECT"
                  | sort @timestamp desc | limit 20' \
  --query queryId --output text)

sleep 10
aws logs get-query-results --query-id $QID
```

**Read a record:**

```
2 vpc-0abc subnet-0def i-0web eni-0ghi 203.0.113.5 10.0.0.42 51234 9999 6 1 40 ... REJECT 2 ingress ...
                                        └─src IP──┘ └─dst IP─┘ └src┘ └dst┘ │        │      └ tcp-flags: 2 = SYN only
                                                                    protocol 6 = TCP  └ no reply was ever sent
```

`tcp_flags = 2` means only a SYN was seen — the connection never completed. That's the fingerprint of a firewall block, as opposed to an application error (which would show a full handshake).

### 10.4 Reachability Analyzer — the definitive answer

```bash
# Ask: can the web instance reach the app instance on 8080?
PATH_ID=$(aws ec2 create-network-insights-path \
  --source $WEB_ID --destination $APP_INSTANCE \
  --destination-port 8080 --protocol tcp \
  --query 'NetworkInsightsPath.NetworkInsightsPathId' --output text)

ANALYSIS=$(aws ec2 start-network-insights-analysis --network-insights-path-id $PATH_ID \
  --query 'NetworkInsightsAnalysis.NetworkInsightsAnalysisId' --output text)

sleep 30
aws ec2 describe-network-insights-analyses --network-insights-analysis-ids $ANALYSIS \
  --query 'NetworkInsightsAnalyses[0].{Found:NetworkPathFound,Explanations:Explanations}'
```

Should return `Found: true` (Lab 4 allowed 8080 from `$SG_WEB`).

**Now break it and re-run:**

```bash
aws ec2 revoke-security-group-ingress --group-id $SG_APP \
  --ip-permissions "IpProtocol=tcp,FromPort=8080,ToPort=8080,UserIdGroupPairs=[{GroupId=$SG_WEB}]"

ANALYSIS2=$(aws ec2 start-network-insights-analysis --network-insights-path-id $PATH_ID \
  --query 'NetworkInsightsAnalysis.NetworkInsightsAnalysisId' --output text)
sleep 30
aws ec2 describe-network-insights-analyses --network-insights-analysis-ids $ANALYSIS2 \
  --query 'NetworkInsightsAnalyses[0].{Found:NetworkPathFound,Why:Explanations[].ExplanationCode}'
```

Now `Found: false` and the explanation names the exact security group. **No packets were sent** — this is pure configuration analysis, which means you can run it against production safely.

Restore:

```bash
aws ec2 authorize-security-group-ingress --group-id $SG_APP \
  --ip-permissions "IpProtocol=tcp,FromPort=8080,ToPort=8080,UserIdGroupPairs=[{GroupId=$SG_WEB}]"
```

### 🧠 Understand — the diagnosis workflow

```
1. Reachability Analyzer  → is the CONFIGURATION correct?  (SG, NACL, route)
       ↓ if it says reachable but it still doesn't work
2. Flow Logs              → are packets ARRIVING?  ACCEPT or REJECT?
       ↓ if ACCEPT but no response
3. On the instance        → ss -tulnp / tcpdump / iptables
                            (the app isn't listening, or the host firewall blocks it)
```

Three layers, in order. Skipping straight to `tcpdump` wastes hours.

---

# Lab 11 — Rebuild It All As Code

**Goal:** produce a reproducible artifact. If you can't rebuild it from a file, you don't own it.

### 11.1 Terraform

```hcl
# main.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = var.region
}

variable "region"   { default = "us-east-1" }
variable "vpc_cidr" { default = "10.0.0.0/16" }
variable "prefix"   { default = "lab" }
variable "azs"      { default = ["us-east-1a", "us-east-1b"] }

# ---------- VPC ----------
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "${var.prefix}-vpc" }
}

# ---------- Subnets ----------
resource "aws_subnet" "public" {
  count                   = length(var.azs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)          # 10.0.0.0/24, 10.0.1.0/24
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "${var.prefix}-public-${var.azs[count.index]}", Tier = "public" }
}

resource "aws_subnet" "app" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)           # 10.0.10.0/24, 10.0.11.0/24
  availability_zone = var.azs[count.index]
  tags = { Name = "${var.prefix}-app-${var.azs[count.index]}", Tier = "app" }
}

resource "aws_subnet" "db" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 20)           # 10.0.20.0/24, 10.0.21.0/24
  availability_zone = var.azs[count.index]
  tags = { Name = "${var.prefix}-db-${var.azs[count.index]}", Tier = "db" }
}

# ---------- Internet Gateway ----------
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.prefix}-igw" }
}

# ---------- NAT: one per AZ ----------
resource "aws_eip" "nat" {
  count  = length(var.azs)
  domain = "vpc"
  tags   = { Name = "${var.prefix}-eip-nat-${count.index}" }
}

resource "aws_nat_gateway" "main" {
  count         = length(var.azs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  depends_on    = [aws_internet_gateway.main]
  tags          = { Name = "${var.prefix}-nat-${count.index}" }
}

# ---------- Route tables ----------
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = { Name = "${var.prefix}-rt-public" }
}

resource "aws_route_table_association" "public" {
  count          = length(var.azs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "app" {
  count  = length(var.azs)
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id   # in-AZ NAT — this matters
  }
  tags = { Name = "${var.prefix}-rt-app-${count.index}" }
}

resource "aws_route_table_association" "app" {
  count          = length(var.azs)
  subnet_id      = aws_subnet.app[count.index].id
  route_table_id = aws_route_table.app[count.index].id
}

resource "aws_route_table" "db" {
  vpc_id = aws_vpc.main.id            # deliberately NO default route
  tags   = { Name = "${var.prefix}-rt-db" }
}

resource "aws_route_table_association" "db" {
  count          = length(var.azs)
  subnet_id      = aws_subnet.db[count.index].id
  route_table_id = aws_route_table.db.id
}

# ---------- Free S3 gateway endpoint ----------
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = concat(aws_route_table.app[*].id, [aws_route_table.db.id])
  tags              = { Name = "${var.prefix}-vpce-s3" }
}

# ---------- Security groups: chained by reference ----------
resource "aws_security_group" "alb" {
  name   = "${var.prefix}-sg-alb"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "public HTTPS"
  }
  egress {
    from_port = 0, to_port = 0, protocol = "-1", cidr_blocks = ["0.0.0.0/0"]
  }
  tags = { Name = "${var.prefix}-sg-alb" }
}

resource "aws_security_group" "app" {
  name   = "${var.prefix}-sg-app"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]     # reference, not CIDR
    description     = "from ALB"
  }
  egress {
    from_port = 0, to_port = 0, protocol = "-1", cidr_blocks = ["0.0.0.0/0"]
  }
  tags = { Name = "${var.prefix}-sg-app" }
}

resource "aws_security_group" "db" {
  name   = "${var.prefix}-sg-db"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "from app tier"
  }
  tags = { Name = "${var.prefix}-sg-db" }
}

# ---------- Outputs ----------
output "vpc_id"          { value = aws_vpc.main.id }
output "public_subnets"  { value = aws_subnet.public[*].id }
output "app_subnets"     { value = aws_subnet.app[*].id }
output "db_subnets"      { value = aws_subnet.db[*].id }
output "nat_public_ips"  { value = aws_eip.nat[*].public_ip }
```

```bash
terraform init
terraform plan
terraform apply
# ...
terraform destroy      # the killer feature: complete, ordered cleanup
```

### 11.2 CloudFormation equivalent (abbreviated)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Lab VPC — 2 AZ, 3 tier

Parameters:
  Prefix:   { Type: String, Default: lab }
  VpcCidr:  { Type: String, Default: 10.0.0.0/16 }

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags: [{ Key: Name, Value: !Sub '${Prefix}-vpc' }]

  IGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags: [{ Key: Name, Value: !Sub '${Prefix}-igw' }]

  AttachIGW:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties: { VpcId: !Ref VPC, InternetGatewayId: !Ref IGW }

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.0.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags: [{ Key: Name, Value: !Sub '${Prefix}-public-1a' }]

  AppSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.10.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags: [{ Key: Name, Value: !Sub '${Prefix}-app-1a' }]

  NatEIP1:
    Type: AWS::EC2::EIP
    DependsOn: AttachIGW
    Properties: { Domain: vpc }

  NatGW1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatEIP1.AllocationId
      SubnetId: !Ref PublicSubnet1

  PublicRT:
    Type: AWS::EC2::RouteTable
    Properties: { VpcId: !Ref VPC }

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachIGW
    Properties:
      RouteTableId: !Ref PublicRT
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref IGW

  PublicAssoc1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: { SubnetId: !Ref PublicSubnet1, RouteTableId: !Ref PublicRT }

  AppRT1:
    Type: AWS::EC2::RouteTable
    Properties: { VpcId: !Ref VPC }

  AppRoute1:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref AppRT1
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGW1

  AppAssoc1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: { SubnetId: !Ref AppSubnet1, RouteTableId: !Ref AppRT1 }

Outputs:
  VpcId:
    Value: !Ref VPC
    Export: { Name: !Sub '${AWS::StackName}-VpcId' }
```

```bash
aws cloudformation deploy --template-file vpc.yaml --stack-name lab-vpc
aws cloudformation delete-stack --stack-name lab-vpc
```

### 🧠 Understand

- Terraform's `cidrsubnet()` removes hand-calculated CIDR errors.
- `depends_on` / `DependsOn` matters: a NAT Gateway created before the IGW attachment completes will fail.
- **Never mix console clicks with IaC.** Drift is the beginning of every outage story.
- Consider the community module `terraform-aws-modules/vpc/aws` for real projects — but build it by hand once first, or you won't understand what it generates.

---

# Lab 12 — Break/Fix Drills and Teardown

**Goal:** practice diagnosis under time pressure, then clean everything up.

## Part A — Drills

For each drill: apply the break, then diagnose using **only** `describe-*` commands, Reachability Analyzer, and Flow Logs. Time yourself. Answers are collapsed.

### Drill 1 — "The website went down"

```bash
aws ec2 revoke-security-group-ingress --group-id $SG_WEB --protocol tcp --port 80 --cidr 0.0.0.0/0
```
<details><summary>Diagnosis path</summary>

`curl` hangs (no `connection refused` → not the app). Reachability Analyzer from `igw-` to the instance on port 80 returns not-reachable, explanation code `ENI_SG_RULES_MISMATCH`. Flow Logs show REJECT with `tcp_flags=2`.

Fix: `aws ec2 authorize-security-group-ingress --group-id $SG_WEB --protocol tcp --port 80 --cidr 0.0.0.0/0`
</details>

### Drill 2 — "Private instances can't download packages"

```bash
aws ec2 delete-route --route-table-id $RT_APP1 --destination-cidr-block 0.0.0.0/0
```
<details><summary>Diagnosis path</summary>

`describe-route-tables` on the subnet's route table shows only the `local` route. No default route → no egress path.

Fix: recreate `0.0.0.0/0 → $NAT1`.
</details>

### Drill 3 — "The NAT Gateway exists but nothing works"

```bash
# Move the NAT Gateway's subnet association conceptually: remove the public subnet's IGW route
aws ec2 delete-route --route-table-id $RT_PUB --destination-cidr-block 0.0.0.0/0
```
<details><summary>Diagnosis path</summary>

Private instances have a route to the NAT Gateway. The NAT Gateway is `available`. But the *public* subnet the NAT lives in has no IGW route, so the NAT Gateway itself has nowhere to send traffic. Always check the NAT's own subnet's route table.

Fix: `aws ec2 create-route --route-table-id $RT_PUB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID`
</details>

### Drill 4 — "Intermittent connection failures"

```bash
# Attach a NACL that allows only a narrow ephemeral range
NACL=$(aws ec2 create-network-acl --vpc-id $VPC_ID --query 'NetworkAcl.NetworkAclId' --output text)
aws ec2 create-network-acl-entry --network-acl-id $NACL --rule-number 100 --protocol -1 \
  --cidr-block 0.0.0.0/0 --rule-action allow --ingress
aws ec2 create-network-acl-entry --network-acl-id $NACL --rule-number 100 --protocol 6 \
  --port-range From=32768,To=40000 --cidr-block 0.0.0.0/0 --rule-action allow --egress
ASSOC=$(aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=$PUB1" \
  --query "NetworkAcls[0].Associations[?SubnetId=='$PUB1'].NetworkAclAssociationId" --output text)
aws ec2 replace-network-acl-association --association-id $ASSOC --network-acl-id $NACL
```
<details><summary>Diagnosis path</summary>

Some clients work, some don't — depending on which ephemeral port their OS picked. Linux uses 32768–60999, Windows 49152–65535. The NACL only allows outbound to 32768–40000, so roughly half of connections silently fail.

Fix: widen to `1024–65535`. **Lesson: intermittent = look at NACLs and ephemeral ports first.**
</details>

### Drill 5 — "I can't delete the subnet"

```bash
aws ec2 delete-subnet --subnet-id $APP1
# → DependencyViolation
```
<details><summary>Diagnosis path</summary>

Something still holds an ENI in that subnet — an instance, a NAT Gateway, a VPC endpoint, an RDS instance, a Lambda function, or a load balancer.

```bash
aws ec2 describe-network-interfaces --filters "Name=subnet-id,Values=$APP1" \
  --query 'NetworkInterfaces[].[NetworkInterfaceId,InterfaceType,Description,Status]' --output table
```
The `Description` field tells you what owns it ("VPC Endpoint Interface vpce-...", "ELB app/...", "AWS Lambda VPC ENI ...").
</details>

### Drill 6 — "IP address exhaustion"

```bash
# In a /24 subnet, launch enough instances to hit the wall (or just simulate)
aws ec2 describe-subnets --subnet-ids $APP1 --query 'Subnets[0].AvailableIpAddressCount'
```
<details><summary>Diagnosis path</summary>

Error on launch: `InsufficientFreeAddressesInSubnet`. Subnet CIDRs can't be resized. Options: add a secondary CIDR to the VPC and create new larger subnets; or clean up unused ENIs. **This is why you plan CIDR sizes before you build.**
</details>

## Part B — Full teardown

Order matters. Run this from top to bottom.

```bash
source ~/lab-vars.sh

# 1. Instances
aws ec2 terminate-instances --instance-ids $WEB_ID $APP_INSTANCE $B_INSTANCE 2>/dev/null
aws ec2 wait instance-terminated --instance-ids $WEB_ID $APP_INSTANCE 2>/dev/null

# 2. VPC endpoints
EPS=$(aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC_ID" \
      --query 'VpcEndpoints[].VpcEndpointId' --output text)
[ -n "$EPS" ] && aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $EPS

# 3. NAT Gateways, then release EIPs
for N in $NAT1 $NAT2; do
  aws ec2 delete-nat-gateway --nat-gateway-id $N 2>/dev/null
done
sleep 120
for A in $ALLOC1 $ALLOC2; do
  aws ec2 release-address --allocation-id $A 2>/dev/null
done

# 4. Peering
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id $PCX 2>/dev/null

# 5. Flow logs
FL=$(aws ec2 describe-flow-logs --filter "Name=resource-id,Values=$VPC_ID" \
     --query 'FlowLogs[].FlowLogId' --output text)
[ -n "$FL" ] && aws ec2 delete-flow-logs --flow-log-ids $FL
aws logs delete-log-group --log-group-name /aws/vpc/$PREFIX-flowlogs 2>/dev/null

# 6. IGW
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# 7. Leftover ENIs
for E in $(aws ec2 describe-network-interfaces --filters "Name=vpc-id,Values=$VPC_ID" \
           "Name=status,Values=available" --query 'NetworkInterfaces[].NetworkInterfaceId' --output text); do
  aws ec2 delete-network-interface --network-interface-id $E
done

# 8. Subnets
for S in $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
           --query 'Subnets[].SubnetId' --output text); do
  aws ec2 delete-subnet --subnet-id $S
done

# 9. Route tables (skip main)
for R in $(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
           --query 'RouteTables[?!(Associations[?Main==`true`])].RouteTableId' --output text); do
  aws ec2 delete-route-table --route-table-id $R
done

# 10. Security groups (skip default) — may need two passes due to cross-references
for PASS in 1 2; do
  for G in $(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" \
             --query 'SecurityGroups[?GroupName!=`default`].GroupId' --output text); do
    aws ec2 delete-security-group --group-id $G 2>/dev/null
  done
done

# 11. Custom NACLs
for N in $(aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC_ID" \
           --query 'NetworkAcls[?!IsDefault].NetworkAclId' --output text); do
  aws ec2 delete-network-acl --network-acl-id $N
done

# 12. The VPCs
aws ec2 delete-vpc --vpc-id $VPC_ID
aws ec2 delete-vpc --vpc-id $VPC_B 2>/dev/null
aws ec2 delete-vpc --vpc-id $VPC_C 2>/dev/null

# 13. IAM
aws iam remove-role-from-instance-profile --instance-profile-name LabSSMProfile --role-name LabSSMRole
aws iam delete-instance-profile --instance-profile-name LabSSMProfile
aws iam detach-role-policy --role-name LabSSMRole --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name LabSSMRole
aws iam delete-role-policy --role-name LabFlowLogsRole --policy-name FlowLogsDelivery
aws iam delete-role --role-name LabFlowLogsRole

rm -f ~/lab-vars.sh trust.json userdata.sh flowlogs-*.json s3-policy.json
```

### ✅ Final verification — nothing left running

```bash
aws ec2 describe-vpcs --query 'Vpcs[?!IsDefault].[VpcId,CidrBlock]' --output table
aws ec2 describe-nat-gateways --filter "Name=state,Values=available" --output table
aws ec2 describe-addresses --output table
aws ec2 describe-transit-gateways --query 'TransitGateways[?State!=`deleted`]' --output table
```

All four should be empty. Then check **Billing → Cost Explorer** tomorrow to confirm nothing is still accruing.

---

## What You've Built and Learned

By finishing these labs you have:

- ✅ Designed and built a multi-AZ, three-tier VPC with a defensible IP plan
- ✅ Understood exactly what makes a subnet public
- ✅ Configured outbound-only egress with per-AZ NAT Gateways
- ✅ Felt the difference between stateful and stateless firewalls, by breaking both
- ✅ Removed the internet entirely using gateway and interface endpoints
- ✅ Connected VPCs with peering and proven non-transitivity
- ✅ Built segmented hub-and-spoke routing with a Transit Gateway
- ✅ Instrumented the network with Flow Logs and Reachability Analyzer
- ✅ Codified everything in Terraform and CloudFormation
- ✅ Diagnosed six realistic failure modes from first principles
- ✅ Torn it all down without leaving a bill behind

### Next challenges

1. Add an **Application Load Balancer** in front of two app instances across AZs, health checks and all.
2. Put an **RDS Multi-AZ** instance in the db subnets with a DB subnet group and prove it has no internet path.
3. Add **IPv6** end-to-end, including an Egress-Only Internet Gateway.
4. Deploy **AWS Network Firewall** with a domain allow-list and route traffic through it.
5. Build **centralised egress**: a shared egress VPC behind a TGW serving three spokes.
6. Run **Network Access Analyzer** and produce a report proving no unintended internet path exists.

---

**Stuck?** → [`troubleshooting.md`](troubleshooting.md) · **Need a command?** → [`commands-cheatsheet.md`](commands-cheatsheet.md) · **Need the theory?** → [`README.md`](README.md)
