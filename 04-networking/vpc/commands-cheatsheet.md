# AWS VPC — CLI Commands Cheat Sheet

> Every command you'll realistically need, grouped by resource, copy-paste ready.
> Replace `vpc-0abc`, `subnet-0aaa`, etc. with your own IDs — or use the shell variables in §0.

**Conventions used here**
- `$VAR` = a shell variable you set once (see §0)
- `--query` uses [JMESPath](https://jmespath.org/) to pull just the field you want
- `--output text` for scripting, `--output table` for reading
- Everything assumes **AWS CLI v2**

---

## Table of Contents

0. [Setup, Variables & Global Flags](#0-setup-variables--global-flags)
1. [VPC](#1-vpc)
2. [Subnets](#2-subnets)
3. [Internet Gateway](#3-internet-gateway)
4. [Route Tables & Routes](#4-route-tables--routes)
5. [NAT Gateway](#5-nat-gateway)
6. [Elastic IPs](#6-elastic-ips)
7. [Security Groups](#7-security-groups)
8. [Network ACLs](#8-network-acls)
9. [Elastic Network Interfaces](#9-elastic-network-interfaces)
10. [VPC Endpoints & PrivateLink](#10-vpc-endpoints--privatelink)
11. [VPC Peering](#11-vpc-peering)
12. [Transit Gateway](#12-transit-gateway)
13. [Site-to-Site VPN, VGW & Customer Gateway](#13-site-to-site-vpn-vgw--customer-gateway)
14. [Client VPN](#14-client-vpn)
15. [Direct Connect](#15-direct-connect)
16. [DHCP Option Sets](#16-dhcp-option-sets)
17. [IPv6 & Egress-Only IGW](#17-ipv6--egress-only-igw)
18. [Prefix Lists](#18-prefix-lists)
19. [VPC Flow Logs](#19-vpc-flow-logs)
20. [Reachability & Network Access Analyzer](#20-reachability--network-access-analyzer)
21. [AWS Network Firewall](#21-aws-network-firewall)
22. [Traffic Mirroring](#22-traffic-mirroring)
23. [IPAM](#23-ipam)
24. [VPC Sharing with RAM](#24-vpc-sharing-with-ram)
25. [Quotas, Tags & Housekeeping](#25-quotas-tags--housekeeping)
26. [In-Instance Networking Commands](#26-in-instance-networking-commands)
27. [One-Liner Audit & Diagnostic Recipes](#27-one-liner-audit--diagnostic-recipes)
28. [Full Build & Full Teardown Scripts](#28-full-build--full-teardown-scripts)

---

## 0. Setup, Variables & Global Flags

```bash
# --- Identity & region ---
aws sts get-caller-identity
aws configure get region
export AWS_REGION=us-east-1
export AWS_DEFAULT_OUTPUT=json

# --- Useful global flags (append to any command) ---
--region us-east-1              # override region
--profile prod                  # use a named profile
--output table|json|text|yaml   # output format
--query 'Vpcs[].VpcId'          # JMESPath filter
--dry-run                       # test permissions without acting (EC2 APIs)
--no-cli-pager                  # stop CLI v2 opening a pager
--debug                         # verbose request/response

# --- Variables used throughout this document ---
export VPC_CIDR=10.0.0.0/16
export AZ1=us-east-1a
export AZ2=us-east-1b
# populate these as you create resources:
export VPC_ID=  IGW_ID=  RTB_PUB=  RTB_PRIV=  NAT_ID=  EIP_ALLOC=
export SUB_PUB1=  SUB_PUB2=  SUB_APP1=  SUB_APP2=  SUB_DB1=  SUB_DB2=
export SG_ALB=  SG_APP=  SG_DB=  SG_EP=
```

**Availability zones (use AZ IDs for multi-account consistency):**

```bash
aws ec2 describe-availability-zones --region us-east-1 \
  --query 'AvailabilityZones[].[ZoneName,ZoneId,State]' --output table
```

---

## 1. VPC

### Create

```bash
# Basic
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# With tags, capture the ID
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc},{Key=Env,Value=prod}]' \
  --query 'Vpc.VpcId' --output text)
echo $VPC_ID

# With an Amazon-provided IPv6 /56
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --amazon-provided-ipv6-cidr-block

# From an IPAM pool
aws ec2 create-vpc --ipv4-ipam-pool-id ipam-pool-0abc --netmask-length 16

# Dedicated tenancy (rarely needed, expensive)
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --instance-tenancy dedicated
```

### Describe / list

```bash
aws ec2 describe-vpcs
aws ec2 describe-vpcs --vpc-ids $VPC_ID
aws ec2 describe-vpcs --filters "Name=tag:Env,Values=prod"
aws ec2 describe-vpcs --filters "Name=isDefault,Values=true"

# Clean table of all VPCs
aws ec2 describe-vpcs --query \
 'Vpcs[].{ID:VpcId,CIDR:CidrBlock,Default:IsDefault,State:State,Name:Tags[?Key==`Name`]|[0].Value}' \
 --output table
```

### Modify attributes

```bash
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support   '{"Value":true}'
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'

# Read them back (must query one at a time)
aws ec2 describe-vpc-attribute --vpc-id $VPC_ID --attribute enableDnsSupport
aws ec2 describe-vpc-attribute --vpc-id $VPC_ID --attribute enableDnsHostnames
```

### Secondary CIDR blocks

```bash
aws ec2 associate-vpc-cidr-block --vpc-id $VPC_ID --cidr-block 10.1.0.0/16
aws ec2 associate-vpc-cidr-block --vpc-id $VPC_ID --amazon-provided-ipv6-cidr-block

# List associations
aws ec2 describe-vpcs --vpc-ids $VPC_ID --query 'Vpcs[].CidrBlockAssociationSet'

aws ec2 disassociate-vpc-cidr-block --association-id vpc-cidr-assoc-0abc
```

### Delete

```bash
# Fails unless every dependency (subnets, IGW, SGs, endpoints...) is gone first
aws ec2 delete-vpc --vpc-id $VPC_ID
```

---

## 2. Subnets

### Create

```bash
SUB_PUB1=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.0.0/20 --availability-zone $AZ1 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a},{Key=Tier,Value=public}]' \
  --query 'Subnet.SubnetId' --output text)

# Using AZ ID instead of name (recommended cross-account)
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.16.0/20 --availability-zone-id use1-az2

# Dual-stack subnet
aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.32.0/20 --ipv6-cidr-block 2600:1f18:abcd:0100::/64 --availability-zone $AZ1

# IPv6-only subnet
aws ec2 create-subnet --vpc-id $VPC_ID \
  --ipv6-cidr-block 2600:1f18:abcd:0200::/64 --ipv6-native --availability-zone $AZ1
```

### Describe

```bash
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID"

# Readable summary
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query \
 'Subnets[].{Name:Tags[?Key==`Name`]|[0].Value,ID:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone,Free:AvailableIpAddressCount,AutoPubIP:MapPublicIpOnLaunch}' \
 --output table

# Only subnets in a given AZ
aws ec2 describe-subnets --filters "Name=availability-zone,Values=$AZ1"

# Only subnets tagged as public
aws ec2 describe-subnets --filters "Name=tag:Tier,Values=public"
```

### Modify

```bash
aws ec2 modify-subnet-attribute --subnet-id $SUB_PUB1 --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $SUB_APP1 --no-map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $SUB_PUB1 --assign-ipv6-address-on-creation
aws ec2 modify-subnet-attribute --subnet-id $SUB_PUB1 --enable-dns64
aws ec2 modify-subnet-attribute --subnet-id $SUB_PUB1 \
  --private-dns-hostname-type-on-launch resource-name
```

### Delete

```bash
aws ec2 delete-subnet --subnet-id $SUB_PUB1
```

---

## 3. Internet Gateway

```bash
# Create
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=prod-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)

# Attach / detach
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# Describe
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID"

# Find the IGW for a VPC in one line
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
  --query 'InternetGateways[0].InternetGatewayId' --output text

# Delete (must be detached first)
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
```

---

## 4. Route Tables & Routes

### Route tables

```bash
RTB_PUB=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rtb-public}]' \
  --query 'RouteTable.RouteTableId' --output text)

# List all in a VPC
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID"

# Find the MAIN route table
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
  "Name=association.main,Values=true" --query 'RouteTables[0].RouteTableId' --output text

# Which route table serves a given subnet?
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$SUB_APP1" \
  --query 'RouteTables[].RouteTableId' --output text

# Readable dump of routes
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" --query \
 'RouteTables[].{RTB:RouteTableId,Name:Tags[?Key==`Name`]|[0].Value,Routes:Routes[].[DestinationCidrBlock,GatewayId,NatGatewayId,TransitGatewayId,VpcPeeringConnectionId,State]}'

aws ec2 delete-route-table --route-table-id $RTB_PUB
```

### Associations

```bash
ASSOC=$(aws ec2 associate-route-table --route-table-id $RTB_PUB --subnet-id $SUB_PUB1 \
  --query AssociationId --output text)

aws ec2 disassociate-route-table --association-id $ASSOC

# Swap a subnet's route table without disassociating
aws ec2 replace-route-table-association --association-id $ASSOC --route-table-id $RTB_PRIV

# Associate to the IGW itself (edge association — for inline inspection)
aws ec2 associate-route-table --route-table-id rtb-0edge --gateway-id $IGW_ID
```

### Routes — one per target type

```bash
# Internet Gateway
aws ec2 create-route --route-table-id $RTB_PUB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# NAT Gateway
aws ec2 create-route --route-table-id $RTB_PRIV --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_ID

# Egress-only IGW (IPv6)
aws ec2 create-route --route-table-id $RTB_PRIV --destination-ipv6-cidr-block ::/0 --egress-only-internet-gateway-id eigw-0abc

# IPv6 via IGW
aws ec2 create-route --route-table-id $RTB_PUB --destination-ipv6-cidr-block ::/0 --gateway-id $IGW_ID

# VPC Peering
aws ec2 create-route --route-table-id $RTB_PRIV --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id pcx-0abc

# Transit Gateway
aws ec2 create-route --route-table-id $RTB_PRIV --destination-cidr-block 10.0.0.0/8 --transit-gateway-id tgw-0abc

# Virtual Private Gateway
aws ec2 create-route --route-table-id $RTB_PRIV --destination-cidr-block 192.168.0.0/16 --gateway-id vgw-0abc

# ENI (NAT instance / appliance)
aws ec2 create-route --route-table-id $RTB_PRIV --destination-cidr-block 0.0.0.0/0 --network-interface-id eni-0abc

# Gateway Load Balancer endpoint
aws ec2 create-route --route-table-id $RTB_PRIV --destination-cidr-block 0.0.0.0/0 --vpc-endpoint-id vpce-0abc

# Prefix list destination
aws ec2 create-route --route-table-id $RTB_PRIV --destination-prefix-list-id pl-0abc --transit-gateway-id tgw-0abc

# Replace / delete
aws ec2 replace-route --route-table-id $RTB_PRIV --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-0new
aws ec2 delete-route  --route-table-id $RTB_PRIV --destination-cidr-block 0.0.0.0/0
```

### Route propagation (VGW/TGW)

```bash
aws ec2 enable-vgw-route-propagation  --route-table-id $RTB_PRIV --gateway-id vgw-0abc
aws ec2 disable-vgw-route-propagation --route-table-id $RTB_PRIV --gateway-id vgw-0abc
```

---

## 5. NAT Gateway

```bash
# Allocate an EIP first (public NAT GW)
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)

# Create a PUBLIC NAT Gateway — note: subnet must be a PUBLIC subnet
NAT_ID=$(aws ec2 create-nat-gateway --subnet-id $SUB_PUB1 --allocation-id $EIP_ALLOC \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-1a}]' \
  --query 'NatGateway.NatGatewayId' --output text)

# Create a PRIVATE NAT Gateway (no EIP — for on-prem/peered translation)
aws ec2 create-nat-gateway --subnet-id $SUB_APP1 --connectivity-type private

# Wait until usable
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_ID

# Describe
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID"
aws ec2 describe-nat-gateways --nat-gateway-ids $NAT_ID --query \
 'NatGateways[].{ID:NatGatewayId,State:State,Subnet:SubnetId,PublicIP:NatGatewayAddresses[0].PublicIp,PrivateIP:NatGatewayAddresses[0].PrivateIp}' --output table

# Add / remove secondary addresses (raises the 55k-per-destination port limit)
aws ec2 associate-nat-gateway-address --nat-gateway-id $NAT_ID --allocation-ids eipalloc-0xyz
aws ec2 disassociate-nat-gateway-address --nat-gateway-id $NAT_ID --association-ids eipassoc-0xyz

# Delete (EIP is NOT released — release it separately)
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_ID
aws ec2 wait nat-gateway-deleted --nat-gateway-ids $NAT_ID
```

### NAT Gateway CloudWatch metrics

```bash
aws cloudwatch get-metric-statistics --namespace AWS/NATGateway \
  --metric-name ErrorPortAllocation \
  --dimensions Name=NatGatewayId,Value=$NAT_ID \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Sum
```

Metrics worth watching: `ErrorPortAllocation`, `PacketsDropCount`, `IdleTimeoutCount`, `BytesOutToDestination`, `ActiveConnectionCount`, `ConnectionEstablishedCount`.

---

## 6. Elastic IPs

```bash
aws ec2 allocate-address --domain vpc
aws ec2 allocate-address --domain vpc --network-border-group us-east-1
aws ec2 allocate-address --domain vpc --public-ipv4-pool ipv4pool-ec2-0abc   # BYOIP

aws ec2 describe-addresses
aws ec2 describe-addresses --query \
 'Addresses[].{IP:PublicIp,Alloc:AllocationId,Instance:InstanceId,ENI:NetworkInterfaceId}' --output table

# Find UNATTACHED EIPs (you're paying for these)
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]' --output table

# Associate / disassociate
aws ec2 associate-address --allocation-id $EIP_ALLOC --instance-id i-0abc
aws ec2 associate-address --allocation-id $EIP_ALLOC --network-interface-id eni-0abc \
  --private-ip-address 10.0.0.25 --allow-reassociation
aws ec2 disassociate-address --association-id eipassoc-0abc

# Release
aws ec2 release-address --allocation-id $EIP_ALLOC
```

---

## 7. Security Groups

### Create & describe

```bash
SG_APP=$(aws ec2 create-security-group --group-name sg-app --description "App tier" \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=sg-app}]' \
  --query GroupId --output text)

aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID"
aws ec2 describe-security-groups --group-ids $SG_APP
aws ec2 describe-security-groups --filters "Name=group-name,Values=sg-app"

# Find the DEFAULT security group
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" \
  "Name=group-name,Values=default" --query 'SecurityGroups[0].GroupId' --output text
```

### Ingress rules

```bash
# Simple form: CIDR source
aws ec2 authorize-security-group-ingress --group-id $SG_ALB --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ALB --protocol tcp --port 80  --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_APP --protocol tcp --port 22  --cidr 203.0.113.0/24

# Port range
aws ec2 authorize-security-group-ingress --group-id $SG_APP --protocol tcp --port 8000-8100 --cidr 10.0.0.0/16

# ICMP (ping)
aws ec2 authorize-security-group-ingress --group-id $SG_APP --protocol icmp --port -1 --cidr 10.0.0.0/16

# ALL traffic
aws ec2 authorize-security-group-ingress --group-id $SG_APP --protocol -1 --port -1 --cidr 10.0.0.0/16

# SOURCE = ANOTHER SECURITY GROUP (do this, not CIDRs)
aws ec2 authorize-security-group-ingress --group-id $SG_APP \
  --ip-permissions 'IpProtocol=tcp,FromPort=8080,ToPort=8080,UserIdGroupPairs=[{GroupId='$SG_ALB',Description="from ALB"}]'

# SELF-REFERENCING (cluster members)
aws ec2 authorize-security-group-ingress --group-id $SG_APP \
  --ip-permissions 'IpProtocol=-1,UserIdGroupPairs=[{GroupId='$SG_APP',Description="intra-cluster"}]'

# With a description on a CIDR rule
aws ec2 authorize-security-group-ingress --group-id $SG_ALB \
  --ip-permissions 'IpProtocol=tcp,FromPort=443,ToPort=443,IpRanges=[{CidrIp=0.0.0.0/0,Description="public HTTPS"}]'

# IPv6 source
aws ec2 authorize-security-group-ingress --group-id $SG_ALB \
  --ip-permissions 'IpProtocol=tcp,FromPort=443,ToPort=443,Ipv6Ranges=[{CidrIpv6=::/0,Description="public HTTPS v6"}]'

# Prefix list source
aws ec2 authorize-security-group-ingress --group-id $SG_APP \
  --ip-permissions 'IpProtocol=tcp,FromPort=22,ToPort=22,PrefixListIds=[{PrefixListId=pl-0office,Description="office ranges"}]'

# Cross-account SG reference (over peering)
aws ec2 authorize-security-group-ingress --group-id $SG_APP \
  --ip-permissions 'IpProtocol=tcp,FromPort=443,ToPort=443,UserIdGroupPairs=[{GroupId=sg-0peer,UserId=111122223333}]'
```

### Egress rules

```bash
aws ec2 authorize-security-group-egress --group-id $SG_APP --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-egress --group-id $SG_APP \
  --ip-permissions 'IpProtocol=tcp,FromPort=3306,ToPort=3306,UserIdGroupPairs=[{GroupId='$SG_DB'}]'

# Remove the default allow-all-outbound rule (do this on locked-down SGs)
aws ec2 revoke-security-group-egress --group-id $SG_APP --protocol -1 --port -1 --cidr 0.0.0.0/0
```

### Revoke, modify, delete

```bash
aws ec2 revoke-security-group-ingress --group-id $SG_ALB --protocol tcp --port 80 --cidr 0.0.0.0/0

# Update just a rule's description
aws ec2 update-security-group-rule-descriptions-ingress --group-id $SG_APP \
  --ip-permissions 'IpProtocol=tcp,FromPort=8080,ToPort=8080,UserIdGroupPairs=[{GroupId='$SG_ALB',Description="updated note"}]'

# Modify a rule in place by rule ID (newer API)
aws ec2 describe-security-group-rules --filters "Name=group-id,Values=$SG_APP"
aws ec2 modify-security-group-rules --group-id $SG_APP \
  --security-group-rules 'SecurityGroupRuleId=sgr-0abc,SecurityGroupRule={IpProtocol=tcp,FromPort=9090,ToPort=9090,CidrIpv4=10.0.0.0/16}'

aws ec2 delete-security-group --group-id $SG_APP

# Change which SGs an instance uses
aws ec2 modify-instance-attribute --instance-id i-0abc --groups $SG_APP $SG_EP
```

### Audit

```bash
# Anything open to the world
aws ec2 describe-security-groups --filters "Name=ip-permission.cidr,Values=0.0.0.0/0" \
  --query 'SecurityGroups[].{Name:GroupName,ID:GroupId,VPC:VpcId}' --output table

# SSH open to the world — the classic finding
aws ec2 describe-security-groups \
  --filters "Name=ip-permission.from-port,Values=22" "Name=ip-permission.cidr,Values=0.0.0.0/0" \
  --query 'SecurityGroups[].[GroupId,GroupName]' --output table

# Unused security groups
comm -23 \
  <(aws ec2 describe-security-groups --query 'SecurityGroups[].GroupId' --output text | tr '\t' '\n' | sort) \
  <(aws ec2 describe-network-interfaces --query 'NetworkInterfaces[].Groups[].GroupId' --output text | tr '\t' '\n' | sort -u)
```

---

## 8. Network ACLs

```bash
# Create
NACL_ID=$(aws ec2 create-network-acl --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=nacl-public}]' \
  --query 'NetworkAcl.NetworkAclId' --output text)

# Describe
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC_ID"
aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=$SUB_PUB1"

# Readable rules dump
aws ec2 describe-network-acls --network-acl-ids $NACL_ID --query \
 'NetworkAcls[].Entries[].{Rule:RuleNumber,Egress:Egress,Proto:Protocol,Ports:PortRange,CIDR:CidrBlock,Action:RuleAction}' \
 --output table
```

### Entries

```bash
# INBOUND allow HTTPS
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 100 --protocol 6 --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow --ingress

# INBOUND allow ephemeral (return traffic) — REQUIRED, NACLs are stateless
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 130 --protocol 6 --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow --ingress

# INBOUND deny a bad actor (low number = evaluated first)
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 10 --protocol -1 --cidr-block 198.51.100.0/24 --rule-action deny --ingress

# OUTBOUND allow all
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 100 --protocol -1 --cidr-block 0.0.0.0/0 --rule-action allow --egress

# IPv6
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 200 --protocol 6 --port-range From=443,To=443 \
  --ipv6-cidr-block ::/0 --rule-action allow --ingress

# ICMP (protocol 1) — type 8 = echo request
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 140 --protocol 1 --icmp-type-code Type=8,Code=-1 \
  --cidr-block 10.0.0.0/16 --rule-action allow --ingress

# Replace / delete an entry
aws ec2 replace-network-acl-entry --network-acl-id $NACL_ID --rule-number 100 \
  --protocol 6 --port-range From=443,To=443 --cidr-block 10.0.0.0/8 --rule-action allow --ingress
aws ec2 delete-network-acl-entry --network-acl-id $NACL_ID --rule-number 100 --ingress
```

**Protocol numbers:** `-1` = all, `1` = ICMP, `6` = TCP, `17` = UDP, `58` = ICMPv6.

### Associate & delete

```bash
# Find the current association ID for a subnet
ASSOC_ID=$(aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=$SUB_PUB1" \
  --query "NetworkAcls[0].Associations[?SubnetId=='$SUB_PUB1'].NetworkAclAssociationId" --output text)

aws ec2 replace-network-acl-association --association-id $ASSOC_ID --network-acl-id $NACL_ID

aws ec2 delete-network-acl --network-acl-id $NACL_ID     # default NACL can't be deleted
```

---

## 9. Elastic Network Interfaces

```bash
# Create
ENI_ID=$(aws ec2 create-network-interface --subnet-id $SUB_APP1 \
  --groups $SG_APP --description "app secondary NIC" \
  --private-ip-address 10.0.32.100 \
  --query 'NetworkInterface.NetworkInterfaceId' --output text)

# Attach / detach
aws ec2 attach-network-interface --network-interface-id $ENI_ID --instance-id i-0abc --device-index 1
aws ec2 detach-network-interface --attachment-id eni-attach-0abc --force

# Describe
aws ec2 describe-network-interfaces --filters "Name=vpc-id,Values=$VPC_ID"
aws ec2 describe-network-interfaces --filters "Name=subnet-id,Values=$SUB_APP1" --query \
 'NetworkInterfaces[].{ENI:NetworkInterfaceId,IP:PrivateIpAddress,Type:InterfaceType,Desc:Description,Status:Status}' \
 --output table

# WHO OWNS THIS IP? (invaluable when a subnet won't delete)
aws ec2 describe-network-interfaces --filters "Name=addresses.private-ip-address,Values=10.0.32.100"

# Manage secondary private IPs
aws ec2 assign-private-ip-addresses --network-interface-id $ENI_ID --secondary-private-ip-address-count 3
aws ec2 unassign-private-ip-addresses --network-interface-id $ENI_ID --private-ip-addresses 10.0.32.101

# IPv6
aws ec2 assign-ipv6-addresses --network-interface-id $ENI_ID --ipv6-address-count 1

# Change SGs / enable delete-on-termination
aws ec2 modify-network-interface-attribute --network-interface-id $ENI_ID --groups $SG_APP $SG_DB
aws ec2 modify-network-interface-attribute --network-interface-id $ENI_ID \
  --attachment AttachmentId=eni-attach-0abc,DeleteOnTermination=true

# Source/destination check (disable for NAT instances & routers)
aws ec2 modify-network-interface-attribute --network-interface-id $ENI_ID --no-source-dest-check
aws ec2 modify-instance-attribute --instance-id i-0abc --no-source-dest-check

aws ec2 delete-network-interface --network-interface-id $ENI_ID
```

---

## 10. VPC Endpoints & PrivateLink

### Discover available services

```bash
aws ec2 describe-vpc-endpoint-services --query 'ServiceNames' --output text | tr '\t' '\n' | sort

# Filter for a service and see its type + AZs
aws ec2 describe-vpc-endpoint-services \
  --filters "Name=service-name,Values=com.amazonaws.us-east-1.s3" \
  --query 'ServiceDetails[].{Name:ServiceName,Types:ServiceType[].ServiceType,AZs:AvailabilityZones,PrivateDNS:PrivateDnsName}'
```

### Gateway endpoints (S3, DynamoDB — free)

```bash
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
  --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids $RTB_PRIV \
  --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=vpce-s3}]'

aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.us-east-1.dynamodb --route-table-ids $RTB_PRIV
```

### Interface endpoints (PrivateLink)

```bash
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --subnet-ids $SUB_APP1 $SUB_APP2 \
  --security-group-ids $SG_EP \
  --private-dns-enabled

# Bulk-create the Session Manager trio
for SVC in ssm ssmmessages ec2messages; do
  aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --vpc-endpoint-type Interface \
    --service-name com.amazonaws.$AWS_REGION.$SVC \
    --subnet-ids $SUB_APP1 $SUB_APP2 --security-group-ids $SG_EP --private-dns-enabled \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=vpce-$SVC}]"
done
```

### Manage

```bash
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC_ID" --query \
 'VpcEndpoints[].{ID:VpcEndpointId,Service:ServiceName,Type:VpcEndpointType,State:State,DNS:PrivateDnsEnabled}' \
 --output table

# Get the endpoint's private DNS names
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids vpce-0abc \
  --query 'VpcEndpoints[].DnsEntries[].DnsName' --output text

# Add/remove subnets, SGs, route tables
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-0abc --add-subnet-ids $SUB_APP2
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-0abc --add-security-group-ids $SG_EP
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-0abc --add-route-table-ids $RTB_DB
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-0abc --private-dns-enabled

# Attach an endpoint policy
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-0abc --policy-document file://endpoint-policy.json

aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-0abc
```

**`endpoint-policy.json` — restrict S3 access to approved buckets:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowApprovedBucketsOnly",
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-app-bucket",
        "arn:aws:s3:::my-app-bucket/*"
      ]
    }
  ]
}
```

### Publish your own PrivateLink service

```bash
# 1. Create the endpoint service behind an NLB
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns arn:aws:elasticloadbalancing:...:loadbalancer/net/my-nlb/abc \
  --acceptance-required \
  --tag-specifications 'ResourceType=vpc-endpoint-service,Tags=[{Key=Name,Value=my-saas}]'

# 2. Allow specific consumer accounts
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-0abc \
  --add-allowed-principals arn:aws:iam::111122223333:root

# 3. Accept incoming connections
aws ec2 describe-vpc-endpoint-connections --filters "Name=service-id,Values=vpce-svc-0abc"
aws ec2 accept-vpc-endpoint-connections --service-id vpce-svc-0abc --vpc-endpoint-ids vpce-0consumer

# 4. Optional: attach a private DNS name (requires domain ownership verification)
aws ec2 modify-vpc-endpoint-service-configuration --service-id vpce-svc-0abc \
  --private-dns-name api.mysaas.example.com
aws ec2 start-vpc-endpoint-service-private-dns-verification --service-id vpce-svc-0abc

aws ec2 delete-vpc-endpoint-service-configurations --service-ids vpce-svc-0abc
```

---

## 11. VPC Peering

```bash
# Create (requester side)
PCX=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $VPC_ID --peer-vpc-id vpc-0target \
  --peer-owner-id 111122223333 --peer-region us-west-2 \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=prod-to-shared}]' \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

# Accept (accepter side — run in the peer account/region)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PCX --region us-west-2

# Describe
aws ec2 describe-vpc-peering-connections --query \
 'VpcPeeringConnections[].{ID:VpcPeeringConnectionId,Status:Status.Code,Requester:RequesterVpcInfo.CidrBlock,Accepter:AccepterVpcInfo.CidrBlock}' \
 --output table

# Routes on BOTH sides (this is the step everyone forgets)
aws ec2 create-route --route-table-id $RTB_PRIV --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id $PCX
# ...and the mirror-image route in the peer VPC

# Enable DNS resolution across the peering (resolve peer's private hostnames)
aws ec2 modify-vpc-peering-connection-options --vpc-peering-connection-id $PCX \
  --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
  --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true

aws ec2 reject-vpc-peering-connection --vpc-peering-connection-id $PCX
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id $PCX
```

---

## 12. Transit Gateway

### Create the gateway

```bash
TGW=$(aws ec2 create-transit-gateway --description "Prod hub" \
  --options 'AmazonSideAsn=64512,AutoAcceptSharedAttachments=disable,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable,DnsSupport=enable,VpnEcmpSupport=enable,MulticastSupport=disable' \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=tgw-prod}]' \
  --query 'TransitGateway.TransitGatewayId' --output text)

aws ec2 describe-transit-gateways
aws ec2 wait transit-gateway-available --transit-gateway-ids $TGW   # (if supported in your CLI version)
```

> For manual segmentation, set `DefaultRouteTableAssociation=disable` and `DefaultRouteTablePropagation=disable`, then wire route tables explicitly.

### Attachments

```bash
# VPC attachment — use small dedicated /28 subnets, one per AZ
ATT=$(aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW --vpc-id $VPC_ID \
  --subnet-ids $SUB_TGW1 $SUB_TGW2 \
  --options 'DnsSupport=enable,Ipv6Support=disable,ApplianceModeSupport=disable' \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)

# Modify (add subnets, enable appliance mode for stateful firewalls)
aws ec2 modify-transit-gateway-vpc-attachment --transit-gateway-attachment-id $ATT \
  --add-subnet-ids $SUB_TGW3 --options ApplianceModeSupport=enable

# TGW peering (cross-region)
aws ec2 create-transit-gateway-peering-attachment \
  --transit-gateway-id $TGW --peer-transit-gateway-id tgw-0remote \
  --peer-account-id 111122223333 --peer-region eu-west-1
aws ec2 accept-transit-gateway-peering-attachment --transit-gateway-attachment-id tgw-attach-0abc

# Connect attachment (GRE/BGP to SD-WAN appliances)
aws ec2 create-transit-gateway-connect --transport-transit-gateway-attachment-id $ATT \
  --options Protocol=gre

aws ec2 describe-transit-gateway-attachments --filters "Name=transit-gateway-id,Values=$TGW"
aws ec2 accept-transit-gateway-vpc-attachment --transit-gateway-attachment-id $ATT
aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id $ATT
```

### TGW route tables

```bash
TGW_RTB=$(aws ec2 create-transit-gateway-route-table --transit-gateway-id $TGW \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=rtb-prod}]' \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' --output text)

# Association = which table this attachment USES
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-route-table-id $TGW_RTB --transit-gateway-attachment-id $ATT

# Propagation = which tables LEARN this attachment's routes
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id $TGW_RTB --transit-gateway-attachment-id $ATT
aws ec2 disable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id $TGW_RTB --transit-gateway-attachment-id $ATT

# Static routes
aws ec2 create-transit-gateway-route --transit-gateway-route-table-id $TGW_RTB \
  --destination-cidr-block 0.0.0.0/0 --transit-gateway-attachment-id $ATT_EGRESS
aws ec2 create-transit-gateway-route --transit-gateway-route-table-id $TGW_RTB \
  --destination-cidr-block 10.99.0.0/16 --blackhole      # deliberately drop
aws ec2 delete-transit-gateway-route --transit-gateway-route-table-id $TGW_RTB \
  --destination-cidr-block 10.99.0.0/16

# Inspect the routing table
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id $TGW_RTB \
  --filters "Name=state,Values=active" --query 'Routes[].[DestinationCidrBlock,Type,State]' --output table

# Which attachments are associated / propagated?
aws ec2 get-transit-gateway-route-table-associations --transit-gateway-route-table-id $TGW_RTB
aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id $TGW_RTB
aws ec2 get-transit-gateway-attachment-propagations --transit-gateway-attachment-id $ATT
```

### Share the TGW cross-account

```bash
aws ram create-resource-share --name tgw-share \
  --resource-arns arn:aws:ec2:us-east-1:111122223333:transit-gateway/$TGW \
  --principals 444455556666
```

---

## 13. Site-to-Site VPN, VGW & Customer Gateway

```bash
# Virtual Private Gateway
VGW=$(aws ec2 create-vpn-gateway --type ipsec.1 --amazon-side-asn 64512 \
  --query 'VpnGateway.VpnGatewayId' --output text)
aws ec2 attach-vpn-gateway --vpn-gateway-id $VGW --vpc-id $VPC_ID
aws ec2 detach-vpn-gateway --vpn-gateway-id $VGW --vpc-id $VPC_ID
aws ec2 delete-vpn-gateway --vpn-gateway-id $VGW

# Customer Gateway (your on-prem device)
CGW=$(aws ec2 create-customer-gateway --type ipsec.1 \
  --public-ip 203.0.113.10 --bgp-asn 65000 \
  --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=hq-router}]' \
  --query 'CustomerGateway.CustomerGatewayId' --output text)

# VPN connection to a VGW (dynamic BGP)
aws ec2 create-vpn-connection --type ipsec.1 \
  --customer-gateway-id $CGW --vpn-gateway-id $VGW \
  --options '{"StaticRoutesOnly":false}'

# VPN connection to a Transit Gateway
aws ec2 create-vpn-connection --type ipsec.1 \
  --customer-gateway-id $CGW --transit-gateway-id $TGW \
  --options '{"StaticRoutesOnly":false,"EnableAcceleration":true}'

# Static routing variant
aws ec2 create-vpn-connection --type ipsec.1 --customer-gateway-id $CGW --vpn-gateway-id $VGW \
  --options '{"StaticRoutesOnly":true}'
aws ec2 create-vpn-connection-route --vpn-connection-id vpn-0abc --destination-cidr-block 192.168.1.0/24

# Tunnel status — check BOTH tunnels
aws ec2 describe-vpn-connections --vpn-connection-ids vpn-0abc --query \
 'VpnConnections[].VgwTelemetry[].{Outside:OutsideIpAddress,Status:Status,Routes:AcceptedRouteCount,Msg:StatusMessage}' \
 --output table

# Download the device configuration for your router vendor
aws ec2 get-vpn-connection-device-types --query 'VpnConnectionDeviceTypes[].[Vendor,Platform,Software]' --output table
aws ec2 get-vpn-connection-device-sample-configuration \
  --vpn-connection-id vpn-0abc --vpn-connection-device-type-id 9c44c1e0 \
  --output text > vpn-config.txt

# Tunnel options (DPD, IKE versions, PSK rotation)
aws ec2 modify-vpn-tunnel-options --vpn-connection-id vpn-0abc \
  --vpn-tunnel-outside-ip-address 52.1.2.3 \
  --tunnel-options '{"DPDTimeoutSeconds":30,"IKEVersions":[{"Value":"ikev2"}]}'

aws ec2 delete-vpn-connection --vpn-connection-id vpn-0abc
```

---

## 14. Client VPN

```bash
aws ec2 create-client-vpn-endpoint \
  --client-cidr-block 172.16.0.0/22 \
  --server-certificate-arn arn:aws:acm:...:certificate/abc \
  --authentication-options 'Type=certificate-authentication,MutualAuthentication={ClientRootCertificateChainArn=arn:aws:acm:...:certificate/def}' \
  --connection-log-options 'Enabled=true,CloudwatchLogGroup=/aws/clientvpn' \
  --split-tunnel --vpc-id $VPC_ID --security-group-ids $SG_APP

aws ec2 associate-client-vpn-target-network --client-vpn-endpoint-id cvpn-endpoint-0abc --subnet-id $SUB_APP1

aws ec2 authorize-client-vpn-ingress --client-vpn-endpoint-id cvpn-endpoint-0abc \
  --target-network-cidr 10.0.0.0/16 --authorize-all-groups

aws ec2 create-client-vpn-route --client-vpn-endpoint-id cvpn-endpoint-0abc \
  --destination-cidr-block 0.0.0.0/0 --target-vpc-subnet-id $SUB_APP1

aws ec2 export-client-vpn-client-configuration --client-vpn-endpoint-id cvpn-endpoint-0abc \
  --output text > client-config.ovpn

aws ec2 describe-client-vpn-connections --client-vpn-endpoint-id cvpn-endpoint-0abc
aws ec2 terminate-client-vpn-connections --client-vpn-endpoint-id cvpn-endpoint-0abc --connection-id cvpn-connection-0abc
aws ec2 delete-client-vpn-endpoint --client-vpn-endpoint-id cvpn-endpoint-0abc
```

---

## 15. Direct Connect

```bash
aws directconnect describe-locations
aws directconnect describe-connections
aws directconnect describe-virtual-interfaces
aws directconnect describe-virtual-gateways
aws directconnect describe-direct-connect-gateways

aws directconnect create-connection --location EqDC2 --bandwidth 1Gbps --connection-name "dx-primary"

aws directconnect create-private-virtual-interface --connection-id dxcon-abc \
  --new-private-virtual-interface 'virtualInterfaceName=vif-prod,vlan=101,asn=65000,virtualGatewayId=vgw-0abc,addressFamily=ipv4'

aws directconnect create-transit-virtual-interface --connection-id dxcon-abc \
  --new-transit-virtual-interface 'virtualInterfaceName=vif-tgw,vlan=102,asn=65000,directConnectGatewayId=dxgw-abc'

aws directconnect create-direct-connect-gateway --direct-connect-gateway-name dxgw-global --amazon-side-asn 64512
aws directconnect create-direct-connect-gateway-association --direct-connect-gateway-id dxgw-abc --gateway-id $TGW \
  --add-allowed-prefixes-to-direct-connect-gateway cidr=10.0.0.0/8

# Download the LOA-CFA to give to your colo provider
aws directconnect describe-loa --connection-id dxcon-abc --output text > loa.pdf
```

---

## 16. DHCP Option Sets

```bash
DOPT=$(aws ec2 create-dhcp-options --dhcp-configurations \
  'Key=domain-name-servers,Values=10.0.0.10,10.0.0.11' \
  'Key=domain-name,Values=corp.internal' \
  'Key=ntp-servers,Values=169.254.169.123' \
  'Key=netbios-name-servers,Values=10.0.0.10' \
  'Key=netbios-node-type,Values=2' \
  --query 'DhcpOptions.DhcpOptionsId' --output text)

aws ec2 associate-dhcp-options --dhcp-options-id $DOPT --vpc-id $VPC_ID

# Revert to AmazonProvidedDNS
aws ec2 associate-dhcp-options --dhcp-options-id default --vpc-id $VPC_ID

aws ec2 describe-dhcp-options
aws ec2 delete-dhcp-options --dhcp-options-id $DOPT
```

> DHCP option sets are **immutable** — to change values, create a new set and re-associate. Instances pick it up on lease renewal or reboot.

---

## 17. IPv6 & Egress-Only IGW

```bash
# Add an Amazon-provided /56 to the VPC
aws ec2 associate-vpc-cidr-block --vpc-id $VPC_ID --amazon-provided-ipv6-cidr-block

# See what you got
aws ec2 describe-vpcs --vpc-ids $VPC_ID --query 'Vpcs[].Ipv6CidrBlockAssociationSet[].Ipv6CidrBlock' --output text

# Give a subnet a /64
aws ec2 associate-subnet-cidr-block --subnet-id $SUB_PUB1 --ipv6-cidr-block 2600:1f18:abcd:0100::/64
aws ec2 modify-subnet-attribute --subnet-id $SUB_PUB1 --assign-ipv6-address-on-creation

# Egress-only IGW (IPv6 outbound-only, free)
EIGW=$(aws ec2 create-egress-only-internet-gateway --vpc-id $VPC_ID \
  --query 'EgressOnlyInternetGateway.EgressOnlyInternetGatewayId' --output text)

aws ec2 create-route --route-table-id $RTB_PRIV --destination-ipv6-cidr-block ::/0 \
  --egress-only-internet-gateway-id $EIGW
aws ec2 create-route --route-table-id $RTB_PUB --destination-ipv6-cidr-block ::/0 --gateway-id $IGW_ID

aws ec2 describe-egress-only-internet-gateways
aws ec2 delete-egress-only-internet-gateway --egress-only-internet-gateway-id $EIGW
```

---

## 18. Prefix Lists

```bash
PL=$(aws ec2 create-managed-prefix-list --prefix-list-name office-ranges \
  --max-entries 20 --address-family IPv4 \
  --entries 'Cidr=203.0.113.0/24,Description=HQ' 'Cidr=198.51.100.0/24,Description=Branch' \
  --query 'PrefixList.PrefixListId' --output text)

aws ec2 modify-managed-prefix-list --prefix-list-id $PL --current-version 1 \
  --add-entries 'Cidr=192.0.2.0/24,Description=NewOffice'

aws ec2 describe-managed-prefix-lists
aws ec2 get-managed-prefix-list-entries --prefix-list-id $PL --output table
aws ec2 get-managed-prefix-list-associations --prefix-list-id $PL

# AWS-managed prefix lists (S3, DynamoDB, CloudFront origin-facing)
aws ec2 describe-managed-prefix-lists --filters "Name=owner-id,Values=AWS" \
  --query 'PrefixLists[].[PrefixListId,PrefixListName]' --output table

# Roll back a bad change
aws ec2 restore-managed-prefix-list-version --prefix-list-id $PL --previous-version 1 --current-version 2

aws ec2 delete-managed-prefix-list --prefix-list-id $PL
```

---

## 19. VPC Flow Logs

### To S3 (recommended — cheapest)

```bash
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flowlogs-bucket/vpc/ \
  --max-aggregation-interval 60 \
  --destination-options 'FileFormat=parquet,HiveCompatiblePartitions=true,PerHourPartition=true'
```

### To CloudWatch Logs

```bash
# IAM role trust policy: vpc-flow-logs.amazonaws.com
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type REJECT \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::111122223333:role/flowlogsRole
```

### Custom format (the fields that actually help)

```bash
aws ec2 create-flow-logs --resource-type VPC --resource-ids $VPC_ID --traffic-type ALL \
  --log-destination-type s3 --log-destination arn:aws:s3:::my-flowlogs-bucket/custom/ \
  --log-format '${version} ${vpc-id} ${subnet-id} ${instance-id} ${interface-id} ${account-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${tcp-flags} ${type} ${pkt-srcaddr} ${pkt-dstaddr} ${flow-direction} ${traffic-path} ${az-id} ${log-status}'
```

### Manage & query

```bash
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=$VPC_ID"
aws ec2 delete-flow-logs --flow-log-ids fl-0abc

# CloudWatch Logs Insights — top rejected talkers
aws logs start-query --log-group-name /aws/vpc/flowlogs \
  --start-time $(date -d '1 hour ago' +%s) --end-time $(date +%s) \
  --query-string 'fields @timestamp, srcaddr, dstaddr, dstport
                  | filter action="REJECT"
                  | stats count(*) as hits by srcaddr, dstport
                  | sort hits desc | limit 20'
aws logs get-query-results --query-id <id-from-above>
```

**Handy Insights queries:**

```sql
-- Top talkers by bytes
fields @timestamp, srcaddr, dstaddr, bytes
| stats sum(bytes) as total by srcaddr, dstaddr
| sort total desc | limit 20

-- Traffic to a specific port
fields @timestamp, srcaddr, dstaddr, dstport, action
| filter dstport = 3306
| sort @timestamp desc

-- Connections that were never established (SYN with no ACK)
fields srcaddr, dstaddr, dstport, tcp_flags
| filter tcp_flags = 2
| stats count(*) by srcaddr, dstaddr, dstport

-- Outbound to the internet (excluding RFC1918)
fields @timestamp, srcaddr, dstaddr, bytes
| filter flow_direction = "egress" and not isIpv4InSubnet(dstaddr, "10.0.0.0/8")
| stats sum(bytes) as total by dstaddr | sort total desc
```

---

## 20. Reachability & Network Access Analyzer

### Reachability Analyzer — "can A reach B, and if not, why?"

```bash
PATH_ID=$(aws ec2 create-network-insights-path \
  --source i-0appinstance --destination i-0dbinstance \
  --destination-port 3306 --protocol tcp \
  --query 'NetworkInsightsPath.NetworkInsightsPathId' --output text)

ANALYSIS=$(aws ec2 start-network-insights-analysis --network-insights-path-id $PATH_ID \
  --query 'NetworkInsightsAnalysis.NetworkInsightsAnalysisId' --output text)

aws ec2 describe-network-insights-analyses --network-insights-analysis-ids $ANALYSIS \
  --query 'NetworkInsightsAnalyses[].{Reachable:NetworkPathFound,Status:Status,Explanations:Explanations}'

# From an internet gateway to an instance
aws ec2 create-network-insights-path --source igw-0abc --destination i-0web --destination-port 443 --protocol tcp

aws ec2 delete-network-insights-analysis --network-insights-analysis-id $ANALYSIS
aws ec2 delete-network-insights-path --network-insights-path-id $PATH_ID
```

### Network Access Analyzer — "prove nothing unintended is exposed"

```bash
aws ec2 create-network-insights-access-scope \
  --match-paths '[{"Source":{"ResourceStatement":{"Types":["AWS::EC2::InternetGateway"]}},"Destination":{"ResourceStatement":{"Types":["AWS::EC2::NetworkInterface"]}}}]'

aws ec2 start-network-insights-access-scope-analysis --network-insights-access-scope-id nis-0abc
aws ec2 get-network-insights-access-scope-analysis-findings --network-insights-access-scope-analysis-id nisa-0abc
```

---

## 21. AWS Network Firewall

```bash
# Rule group (stateful, Suricata syntax)
aws network-firewall create-rule-group --rule-group-name block-bad-domains \
  --type STATEFUL --capacity 100 \
  --rule-group '{"RulesSource":{"RulesString":"drop tls any any -> any any (tls.sni; content:\"evil.example.com\"; sid:1;)"}}'

# Domain allow-list
aws network-firewall create-rule-group --rule-group-name allow-aws-only \
  --type STATEFUL --capacity 100 \
  --rule-group '{"RulesSource":{"RulesSourceList":{"Targets":[".amazonaws.com",".ubuntu.com"],"TargetTypes":["TLS_SNI","HTTP_HOST"],"GeneratedRulesType":"ALLOWLIST"}}}'

# Policy
aws network-firewall create-firewall-policy --firewall-policy-name prod-policy \
  --firewall-policy '{"StatelessDefaultActions":["aws:forward_to_sfe"],"StatelessFragmentDefaultActions":["aws:forward_to_sfe"],"StatefulRuleGroupReferences":[{"ResourceArn":"arn:aws:network-firewall:...:stateful-rulegroup/block-bad-domains"}]}'

# Firewall — one dedicated subnet per AZ, nothing else in them
aws network-firewall create-firewall --firewall-name prod-fw \
  --firewall-policy-arn arn:aws:network-firewall:...:firewall-policy/prod-policy \
  --vpc-id $VPC_ID \
  --subnet-mappings SubnetId=$SUB_FW1 SubnetId=$SUB_FW2 \
  --delete-protection --subnet-change-protection

# Get the endpoint IDs to use as route targets
aws network-firewall describe-firewall --firewall-name prod-fw \
  --query 'FirewallStatus.SyncStates.*.Attachment.EndpointId'

aws network-firewall describe-logging-configuration --firewall-name prod-fw
aws network-firewall update-logging-configuration --firewall-name prod-fw \
  --logging-configuration '{"LogDestinationConfigs":[{"LogType":"ALERT","LogDestinationType":"S3","LogDestination":{"bucketName":"my-fw-logs"}}]}'
```

---

## 22. Traffic Mirroring

```bash
aws ec2 create-traffic-mirror-target --network-load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/net/ids-nlb/abc
aws ec2 create-traffic-mirror-target --network-interface-id eni-0ids

FILTER=$(aws ec2 create-traffic-mirror-filter --description "mirror inbound web" \
  --query 'TrafficMirrorFilter.TrafficMirrorFilterId' --output text)

aws ec2 create-traffic-mirror-filter-rule --traffic-mirror-filter-id $FILTER \
  --traffic-direction ingress --rule-number 100 --rule-action accept \
  --protocol 6 --destination-cidr-block 0.0.0.0/0 --source-cidr-block 0.0.0.0/0 \
  --destination-port-range FromPort=443,ToPort=443

aws ec2 create-traffic-mirror-session --network-interface-id eni-0source \
  --traffic-mirror-target-id tmt-0abc --traffic-mirror-filter-id $FILTER \
  --session-number 1 --packet-length 8500

aws ec2 describe-traffic-mirror-sessions
aws ec2 delete-traffic-mirror-session --traffic-mirror-session-id tms-0abc
```

---

## 23. IPAM

```bash
IPAM=$(aws ec2 create-ipam --operating-regions RegionName=us-east-1 RegionName=eu-west-1 \
  --query 'Ipam.IpamId' --output text)

SCOPE=$(aws ec2 describe-ipams --ipam-ids $IPAM --query 'Ipams[0].PrivateDefaultScopeId' --output text)

# Top-level pool
TOP=$(aws ec2 create-ipam-pool --ipam-scope-id $SCOPE --address-family ipv4 \
  --query 'IpamPool.IpamPoolId' --output text)
aws ec2 provision-ipam-pool-cidr --ipam-pool-id $TOP --cidr 10.0.0.0/8

# Regional child pool
REG=$(aws ec2 create-ipam-pool --ipam-scope-id $SCOPE --address-family ipv4 \
  --source-ipam-pool-id $TOP --locale us-east-1 \
  --allocation-default-netmask-length 16 --auto-import \
  --query 'IpamPool.IpamPoolId' --output text)
aws ec2 provision-ipam-pool-cidr --ipam-pool-id $REG --cidr 10.0.0.0/12

# Create a VPC from the pool — IPAM picks a free block
aws ec2 create-vpc --ipv4-ipam-pool-id $REG --netmask-length 16

aws ec2 get-ipam-pool-allocations --ipam-pool-id $REG
aws ec2 get-ipam-address-history --cidr 10.0.0.0/16 --ipam-scope-id $SCOPE
aws ec2 get-ipam-resource-cidrs --ipam-scope-id $SCOPE
```

---

## 24. VPC Sharing with RAM

```bash
aws ram create-resource-share --name shared-subnets \
  --resource-arns arn:aws:ec2:us-east-1:111122223333:subnet/$SUB_APP1 \
                  arn:aws:ec2:us-east-1:111122223333:subnet/$SUB_APP2 \
  --principals ou-abcd-12345678 \
  --allow-external-principals

aws ram get-resource-shares --resource-owner SELF
aws ram list-resources --resource-owner SELF
aws ram associate-resource-share --resource-share-arn arn:aws:ram:... --principals 444455556666

# In the participant account
aws ram get-resource-share-invitations
aws ram accept-resource-share-invitation --resource-share-invitation-arn arn:aws:ram:...
aws ec2 describe-subnets --filters "Name=owner-id,Values=111122223333"
```

---

## 25. Quotas, Tags & Housekeeping

```bash
# Service quotas
aws service-quotas list-service-quotas --service-code vpc --output table
aws service-quotas get-service-quota --service-code vpc --quota-code L-F678F1CE   # VPCs per region
aws service-quotas request-service-quota-increase --service-code vpc --quota-code L-F678F1CE --desired-value 10

# Network Address Usage (how close to IP exhaustion?)
aws ec2 describe-vpcs --vpc-ids $VPC_ID --query 'Vpcs[].{NAU:.}' 2>/dev/null
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name NetworkAddressUsage \
  --dimensions Name=Per-VPC-Metric,Value=$VPC_ID --start-time $(date -u -d '1 day ago' +%FT%TZ) \
  --end-time $(date -u +%FT%TZ) --period 3600 --statistics Maximum

# Tags
aws ec2 create-tags --resources $VPC_ID $SUB_PUB1 --tags Key=Env,Value=prod Key=Owner,Value=platform
aws ec2 delete-tags --resources $VPC_ID --tags Key=Temp
aws ec2 describe-tags --filters "Name=resource-id,Values=$VPC_ID"

# Find UNTAGGED VPC resources
aws resourcegroupstaggingapi get-resources --resource-type-filters ec2:vpc ec2:subnet \
  --query 'ResourceTagMappingList[?length(Tags)==`0`].ResourceARN'
```

---

## 26. In-Instance Networking Commands

Run these **inside** an EC2 instance (via SSH or Session Manager) to diagnose from the guest side.

```bash
# --- Identity & addressing (IMDSv2) ---
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
        -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/mac
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/network/interfaces/macs/

# --- Interfaces & routes ---
ip addr show
ip route show
ip -6 route show
ip neigh                     # ARP table

# --- What public IP am I seen as? (should be the NAT GW EIP) ---
curl -s https://checkip.amazonaws.com

# --- Connectivity ---
ping -c 4 10.0.64.20                    # ICMP — remember to allow it in the SG!
traceroute -n 8.8.8.8
mtr -rw 8.8.8.8
nc -zv 10.0.64.20 3306                  # TCP port test — THE most useful one
nc -zvu 10.0.0.2 53                     # UDP port test
telnet db.internal 3306
timeout 5 bash -c '</dev/tcp/10.0.64.20/3306' && echo OPEN || echo CLOSED

# --- DNS ---
dig +short amazonaws.com
dig @10.0.0.2 secretsmanager.us-east-1.amazonaws.com    # should return a PRIVATE IP if the endpoint works
dig +trace example.com
nslookup my-db.abc.us-east-1.rds.amazonaws.com
cat /etc/resolv.conf
resolvectl status                        # systemd-resolved

# --- Listening sockets ---
ss -tulnp
ss -tan state established
netstat -tulnp

# --- MTU / path MTU ---
ip link show eth0 | grep mtu
ping -M do -s 8972 10.0.64.20            # test 9001 MTU (8972 + 28 header)
ping -M do -s 1472 8.8.8.8               # test 1500 MTU

# --- Packet capture ---
sudo tcpdump -i eth0 -nn port 3306
sudo tcpdump -i any -nn 'host 10.0.64.20 and tcp[tcpflags] & tcp-syn != 0'
sudo tcpdump -i eth0 -nn -c 100 -w capture.pcap

# --- Host firewall (don't forget this layer) ---
sudo iptables -L -n -v
sudo firewall-cmd --list-all
sudo nft list ruleset

# --- AWS API reachability from the instance ---
aws sts get-caller-identity
aws s3 ls                                # tests the S3 gateway endpoint
```

---

## 27. One-Liner Audit & Diagnostic Recipes

```bash
# 1. Which subnets are PUBLIC? (have a route to an IGW)
for RT in $(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
            --query 'RouteTables[?Routes[?starts_with(GatewayId,`igw-`)]].RouteTableId' --output text); do
  aws ec2 describe-route-tables --route-table-ids $RT \
    --query 'RouteTables[].Associations[].SubnetId' --output text
done

# 2. Instances with public IPs (audit for accidental exposure)
aws ec2 describe-instances --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Reservations[].Instances[?PublicIpAddress!=null].[InstanceId,PublicIpAddress,SubnetId]' --output table

# 3. Subnets running low on IPs
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[?AvailableIpAddressCount<`50`].[SubnetId,CidrBlock,AvailableIpAddressCount]' --output table

# 4. Subnets with NO explicit route table (using the main table — often a bug)
MAIN=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.main,Values=true" --query 'RouteTables[0].RouteTableId' --output text)
echo "Main RT: $MAIN"
aws ec2 describe-route-tables --route-table-ids $MAIN --query 'RouteTables[].Associations'

# 5. Every ENI in the VPC and what it belongs to (essential before deleting anything)
aws ec2 describe-network-interfaces --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'NetworkInterfaces[].[NetworkInterfaceId,InterfaceType,PrivateIpAddress,Description,Status]' --output table

# 6. Cost check: unattached EIPs and idle NAT Gateways
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].PublicIp' --output text
aws ec2 describe-nat-gateways --filter "Name=state,Values=available" \
  --query 'NatGateways[].[NatGatewayId,VpcId,SubnetId]' --output table

# 7. All VPC endpoints across all regions
for R in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  echo "== $R =="
  aws ec2 describe-vpc-endpoints --region $R --query 'VpcEndpoints[].[VpcEndpointId,ServiceName]' --output text
done

# 8. Full VPC inventory dump (paste into a ticket)
{
  echo "=== VPC ==="        ; aws ec2 describe-vpcs --vpc-ids $VPC_ID
  echo "=== SUBNETS ==="    ; aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID"
  echo "=== ROUTE TABLES ==="; aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID"
  echo "=== IGW ==="        ; aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID"
  echo "=== NAT ==="        ; aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID"
  echo "=== SGs ==="        ; aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID"
  echo "=== NACLs ==="      ; aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC_ID"
  echo "=== ENDPOINTS ===" ; aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC_ID"
} > vpc-inventory-$(date +%F).json
```

---

## 28. Full Build & Full Teardown Scripts

### Build a complete 2-AZ, 3-tier VPC

```bash
#!/usr/bin/env bash
set -euo pipefail
REGION=us-east-1; AZ1=${REGION}a; AZ2=${REGION}b; NAME=demo

VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$NAME-vpc}]" \
  --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'

mk_subnet () { # $1=cidr $2=az $3=name
  aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block $1 --availability-zone $2 \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$NAME-$3}]" \
    --query Subnet.SubnetId --output text
}
PUB1=$(mk_subnet 10.0.0.0/20  $AZ1 public-1a)
PUB2=$(mk_subnet 10.0.16.0/20 $AZ2 public-1b)
APP1=$(mk_subnet 10.0.32.0/20 $AZ1 app-1a)
APP2=$(mk_subnet 10.0.48.0/20 $AZ2 app-1b)
DB1=$(mk_subnet  10.0.64.0/20 $AZ1 db-1a)
DB2=$(mk_subnet  10.0.80.0/20 $AZ2 db-1b)

aws ec2 modify-subnet-attribute --subnet-id $PUB1 --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $PUB2 --map-public-ip-on-launch

IGW=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC_ID

RT_PUB=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $RT_PUB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $PUB1
aws ec2 associate-route-table --route-table-id $RT_PUB --subnet-id $PUB2

# NAT per AZ
for i in 1 2; do
  eval PUB=\$PUB$i; eval APP=\$APP$i
  ALLOC=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
  NAT=$(aws ec2 create-nat-gateway --subnet-id $PUB --allocation-id $ALLOC \
        --query NatGateway.NatGatewayId --output text)
  aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT
  RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
  aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT
  aws ec2 associate-route-table --route-table-id $RT --subnet-id $APP
  eval RT_APP$i=$RT
done

# DB route table — deliberately no default route
RT_DB=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 associate-route-table --route-table-id $RT_DB --subnet-id $DB1
aws ec2 associate-route-table --route-table-id $RT_DB --subnet-id $DB2

# Free S3 gateway endpoint everywhere
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --service-name com.amazonaws.$REGION.s3 \
  --route-table-ids $RT_APP1 $RT_APP2 $RT_DB

echo "VPC_ID=$VPC_ID"
```

### Teardown (deletion order matters)

```bash
#!/usr/bin/env bash
# Delete in dependency order. Save yourself the "DependencyViolation" pain.
set -x
VPC_ID=vpc-0abc

# 1. EC2 instances
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=instance-state-name,Values=running,stopped" \
  --query 'Reservations[].Instances[].InstanceId' --output text)

# 2. Load balancers (they hold ENIs)
# aws elbv2 delete-load-balancer --load-balancer-arn ...

# 3. VPC endpoints
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $(aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=$VPC_ID" --query 'VpcEndpoints[].VpcEndpointId' --output text)

# 4. NAT gateways (then release EIPs)
for N in $(aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" \
           "Name=state,Values=available" --query 'NatGateways[].NatGatewayId' --output text); do
  aws ec2 delete-nat-gateway --nat-gateway-id $N
  aws ec2 wait nat-gateway-deleted --nat-gateway-ids $N
done
for A in $(aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].AllocationId' --output text); do
  aws ec2 release-address --allocation-id $A
done

# 5. Peering connections
for P in $(aws ec2 describe-vpc-peering-connections \
           --filters "Name=requester-vpc-info.vpc-id,Values=$VPC_ID" \
           --query 'VpcPeeringConnections[].VpcPeeringConnectionId' --output text); do
  aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id $P
done

# 6. Detach & delete IGW
IGW=$(aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
      --query 'InternetGateways[0].InternetGatewayId' --output text)
aws ec2 detach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW

# 7. Leftover ENIs (the usual blocker)
for E in $(aws ec2 describe-network-interfaces --filters "Name=vpc-id,Values=$VPC_ID" \
           "Name=status,Values=available" --query 'NetworkInterfaces[].NetworkInterfaceId' --output text); do
  aws ec2 delete-network-interface --network-interface-id $E
done

# 8. Subnets
for S in $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
           --query 'Subnets[].SubnetId' --output text); do
  aws ec2 delete-subnet --subnet-id $S
done

# 9. Non-main route tables
for R in $(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
           --query 'RouteTables[?!(Associations[?Main==`true`])].RouteTableId' --output text); do
  aws ec2 delete-route-table --route-table-id $R
done

# 10. Non-default security groups & NACLs
for G in $(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" \
           --query 'SecurityGroups[?GroupName!=`default`].GroupId' --output text); do
  aws ec2 delete-security-group --group-id $G
done
for N in $(aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC_ID" \
           --query 'NetworkAcls[?!IsDefault].NetworkAclId' --output text); do
  aws ec2 delete-network-acl --network-acl-id $N
done

# 11. Flow logs
aws ec2 delete-flow-logs --flow-log-ids $(aws ec2 describe-flow-logs \
  --filter "Name=resource-id,Values=$VPC_ID" --query 'FlowLogs[].FlowLogId' --output text)

# 12. Finally, the VPC
aws ec2 delete-vpc --vpc-id $VPC_ID
```

---

## Quick Reference Card

| I want to… | Command |
|---|---|
| List my VPCs | `aws ec2 describe-vpcs --output table` |
| Find a subnet's route table | `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=SUBNET"` |
| See why a port is blocked | `aws ec2 create-network-insights-path ... && start-network-insights-analysis` |
| Check NAT Gateway state | `aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=VPC"` |
| See who owns an IP | `aws ec2 describe-network-interfaces --filters "Name=addresses.private-ip-address,Values=IP"` |
| Find world-open SGs | `aws ec2 describe-security-groups --filters "Name=ip-permission.cidr,Values=0.0.0.0/0"` |
| Test a port from an instance | `nc -zv HOST PORT` |
| Confirm outbound IP | `curl -s https://checkip.amazonaws.com` |
| Check DNS is working | `dig @10.0.0.2 amazonaws.com` |
| Find unattached EIPs | `aws ec2 describe-addresses --query 'Addresses[?AssociationId==null]'` |

---

**Next:** [`hands-on-labs.md`](hands-on-labs.md) to build it · [`troubleshooting.md`](troubleshooting.md) when it breaks
