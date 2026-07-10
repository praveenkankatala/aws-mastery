# AWS VPC — CLI Commands Cheatsheet

> Quick-reference AWS CLI v2 commands for every VPC component. Replace placeholder IDs (`vpc-xxxx`, `subnet-xxxx`, etc.) with real resource IDs. All commands assume a default region is configured (`aws configure`) or pass `--region <region>` explicitly.

## Table of Contents

- [VPC](#vpc)
- [Subnets](#subnets)
- [Internet Gateway](#internet-gateway)
- [NAT Gateway & Elastic IP](#nat-gateway--elastic-ip)
- [Route Tables](#route-tables)
- [Security Groups](#security-groups)
- [Network ACLs](#network-acls)
- [VPC Peering](#vpc-peering)
- [Transit Gateway](#transit-gateway)
- [VPC Endpoints](#vpc-endpoints)
- [Site-to-Site VPN](#site-to-site-vpn)
- [Client VPN](#client-vpn)
- [VPC Flow Logs](#vpc-flow-logs)
- [DHCP Option Sets](#dhcp-option-sets)
- [Diagnostics / Reachability](#diagnostics--reachability)
- [Cleanup / Teardown Order](#cleanup--teardown-order)

---

## VPC

```bash
# Create a VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=production-vpc}]'

# List all VPCs
aws ec2 describe-vpcs

# Describe a specific VPC
aws ec2 describe-vpcs --vpc-ids vpc-xxxx

# Add a secondary CIDR block
aws ec2 associate-vpc-cidr-block --vpc-id vpc-xxxx --cidr-block 10.1.0.0/16

# Enable DNS hostnames / support
aws ec2 modify-vpc-attribute --vpc-id vpc-xxxx --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id vpc-xxxx --enable-dns-support

# Delete a VPC (must be empty of dependencies first)
aws ec2 delete-vpc --vpc-id vpc-xxxx
```

---

## Subnets

```bash
# Create a subnet
aws ec2 create-subnet --vpc-id vpc-xxxx --cidr-block 10.0.101.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-a}]'

# List subnets for a VPC
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxxx"

# Enable auto-assign public IPv4 on a subnet (makes it usable as a "public" subnet)
aws ec2 modify-subnet-attribute --subnet-id subnet-xxxx --map-public-ip-on-launch

# Delete a subnet
aws ec2 delete-subnet --subnet-id subnet-xxxx
```

---

## Internet Gateway

```bash
# Create an Internet Gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=production-igw}]'

# Attach it to a VPC
aws ec2 attach-internet-gateway --vpc-id vpc-xxxx --internet-gateway-id igw-xxxx

# List Internet Gateways
aws ec2 describe-internet-gateways

# Detach and delete
aws ec2 detach-internet-gateway --vpc-id vpc-xxxx --internet-gateway-id igw-xxxx
aws ec2 delete-internet-gateway --internet-gateway-id igw-xxxx
```

---

## NAT Gateway & Elastic IP

```bash
# Allocate an Elastic IP for the NAT Gateway
aws ec2 allocate-address --domain vpc

# Create a NAT Gateway in a public subnet
aws ec2 create-nat-gateway --subnet-id subnet-xxxx --allocation-id eipalloc-xxxx \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-a}]'

# Check NAT Gateway status (wait for "available")
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=vpc-xxxx"

# Delete a NAT Gateway
aws ec2 delete-nat-gateway --nat-gateway-id nat-xxxx

# Release the Elastic IP (after NAT GW deletion completes)
aws ec2 release-address --allocation-id eipalloc-xxxx
```

---

## Route Tables

```bash
# Create a route table
aws ec2 create-route-table --vpc-id vpc-xxxx \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'

# Add a route to Internet Gateway
aws ec2 create-route --route-table-id rtb-xxxx \
  --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxxx

# Add a route to NAT Gateway
aws ec2 create-route --route-table-id rtb-yyyy \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-xxxx

# Associate a route table with a subnet
aws ec2 associate-route-table --route-table-id rtb-xxxx --subnet-id subnet-xxxx

# List route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-xxxx"

# Replace a subnet's route table association
aws ec2 replace-route-table-association --association-id rtbassoc-xxxx --route-table-id rtb-yyyy

# Delete a route
aws ec2 delete-route --route-table-id rtb-xxxx --destination-cidr-block 0.0.0.0/0
```

---

## Security Groups

```bash
# Create a Security Group
aws ec2 create-security-group --group-name web-sg \
  --description "Web tier SG" --vpc-id vpc-xxxx

# Allow inbound HTTPS from anywhere
aws ec2 authorize-security-group-ingress --group-id sg-web \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# Allow inbound from another Security Group (SG-to-SG chaining)
aws ec2 authorize-security-group-ingress --group-id sg-app \
  --protocol tcp --port 8080 --source-group sg-web

# List Security Groups
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=vpc-xxxx"

# Revoke a rule
aws ec2 revoke-security-group-ingress --group-id sg-web \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# Delete a Security Group
aws ec2 delete-security-group --group-id sg-web
```

---

## Network ACLs

```bash
# Create a NACL
aws ec2 create-network-acl --vpc-id vpc-xxxx \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=public-nacl}]'

# Add an inbound allow rule (rule numbers determine evaluation order)
aws ec2 create-network-acl-entry --network-acl-id acl-xxxx \
  --ingress --rule-number 100 --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Add an outbound rule for ephemeral ports (required since NACLs are stateless)
aws ec2 create-network-acl-entry --network-acl-id acl-xxxx \
  --egress --rule-number 100 --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Add an explicit DENY rule (something Security Groups cannot do)
aws ec2 create-network-acl-entry --network-acl-id acl-xxxx \
  --ingress --rule-number 90 --protocol tcp --port-range From=0,To=65535 \
  --cidr-block 203.0.113.0/24 --rule-action deny

# Associate NACL with a subnet
aws ec2 replace-network-acl-association --association-id aclassoc-xxxx --network-acl-id acl-xxxx

# List NACLs
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=vpc-xxxx"

# Delete a rule
aws ec2 delete-network-acl-entry --network-acl-id acl-xxxx --rule-number 100 --ingress
```

---

## VPC Peering

```bash
# Request a peering connection
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-requester --peer-vpc-id vpc-accepter --peer-region ap-south-1

# Accept the peering connection (from the accepter account/region)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-xxxx

# Add routes on both sides so traffic actually flows
aws ec2 create-route --route-table-id rtb-a \
  --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id pcx-xxxx
aws ec2 create-route --route-table-id rtb-b \
  --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id pcx-xxxx

# List peering connections
aws ec2 describe-vpc-peering-connections

# Delete a peering connection
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id pcx-xxxx
```

---

## Transit Gateway

```bash
# Create a Transit Gateway
aws ec2 create-transit-gateway --description "Central hub TGW" \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=central-tgw}]'

# Attach a VPC to the TGW
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-xxxx --vpc-id vpc-xxxx --subnet-ids subnet-a subnet-b

# List TGW attachments
aws ec2 describe-transit-gateway-attachments

# Add a route in a VPC route table pointing to the TGW
aws ec2 create-route --route-table-id rtb-xxxx \
  --destination-cidr-block 10.2.0.0/16 --transit-gateway-id tgw-xxxx

# Create a TGW route table and associate an attachment
aws ec2 create-transit-gateway-route-table --transit-gateway-id tgw-xxxx
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-route-table-id tgw-rtb-xxxx --transit-gateway-attachment-id tgw-attach-xxxx

# Delete a VPC attachment
aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id tgw-attach-xxxx
```

---

## VPC Endpoints

```bash
# Create a Gateway Endpoint (S3) — free, adds a route table target
aws ec2 create-vpc-endpoint --vpc-id vpc-xxxx \
  --service-name com.amazonaws.ap-south-1.s3 \
  --vpc-endpoint-type Gateway --route-table-ids rtb-xxxx

# Create an Interface Endpoint (e.g., Secrets Manager) — provisions an ENI
aws ec2 create-vpc-endpoint --vpc-id vpc-xxxx \
  --service-name com.amazonaws.ap-south-1.secretsmanager \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-app-a subnet-app-b \
  --security-group-ids sg-endpoint

# List endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=vpc-xxxx"

# Delete an endpoint
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-xxxx
```

---

## Site-to-Site VPN

```bash
# Create a Customer Gateway (represents your on-prem device)
aws ec2 create-customer-gateway --type ipsec.1 \
  --public-ip 203.0.113.10 --bgp-asn 65000

# Create a Virtual Private Gateway and attach to VPC
aws ec2 create-vpn-gateway --type ipsec.1
aws ec2 attach-vpn-gateway --vpn-gateway-id vgw-xxxx --vpc-id vpc-xxxx

# Create the Site-to-Site VPN connection
aws ec2 create-vpn-connection --type ipsec.1 \
  --customer-gateway-id cgw-xxxx --vpn-gateway-id vgw-xxxx \
  --options StaticRoutesOnly=false

# Enable route propagation so the VPC learns on-prem routes automatically
aws ec2 enable-vgw-route-propagation --route-table-id rtb-xxxx --gateway-id vgw-xxxx

# Check tunnel status
aws ec2 describe-vpn-connections --vpn-connection-ids vpn-xxxx
```

---

## Client VPN

```bash
# Create a Client VPN endpoint (requires ACM server + client certificates pre-provisioned)
aws ec2 create-client-vpn-endpoint \
  --client-cidr-block 172.16.0.0/22 \
  --server-certificate-arn arn:aws:acm:ap-south-1:123456789012:certificate/xxxx \
  --authentication-options Type=certificate-authentication,MutualAuthentication={ClientRootCertificateChainArn=arn:aws:acm:...} \
  --connection-log-options Enabled=false

# Associate a target subnet
aws ec2 associate-client-vpn-target-network \
  --client-vpn-endpoint-id cvpn-endpoint-xxxx --subnet-id subnet-app-a

# Authorize a network (which CIDR the client can reach)
aws ec2 authorize-client-vpn-ingress \
  --client-vpn-endpoint-id cvpn-endpoint-xxxx \
  --target-network-cidr 10.0.0.0/16 --authorize-all-groups
```

---

## VPC Flow Logs

```bash
# Enable Flow Logs to CloudWatch Logs (VPC-level)
aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-xxxx \
  --traffic-type ALL --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flow-logs --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flow-logs-role

# Enable Flow Logs to S3 (subnet-level)
aws ec2 create-flow-logs --resource-type Subnet --resource-ids subnet-xxxx \
  --traffic-type REJECT --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flow-logs-bucket/vpc/

# List Flow Logs
aws ec2 describe-flow-logs

# Delete a Flow Log
aws ec2 delete-flow-logs --flow-log-ids fl-xxxx
```

---

## DHCP Option Sets

```bash
# Create a custom DHCP option set
aws ec2 create-dhcp-options --dhcp-configurations \
  "Key=domain-name-servers,Values=10.0.0.2" \
  "Key=domain-name,Values=example.internal"

# Associate with a VPC
aws ec2 associate-dhcp-options --dhcp-options-id dopt-xxxx --vpc-id vpc-xxxx

# List option sets
aws ec2 describe-dhcp-options
```

---

## Diagnostics / Reachability

```bash
# Run a Reachability Analyzer path check between two ENIs
aws ec2 create-network-insights-path \
  --source eni-source-xxxx --destination eni-dest-xxxx --protocol tcp --destination-port 5432

aws ec2 start-network-insights-analysis --network-insights-path-id nip-xxxx

aws ec2 describe-network-insights-analyses --network-insights-analysis-ids nia-xxxx

# Describe route table associated with a specific subnet (quick sanity check)
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-xxxx"

# Check Security Group rules attached to an instance's ENI
aws ec2 describe-network-interfaces --network-interface-ids eni-xxxx \
  --query 'NetworkInterfaces[0].Groups'
```

---

## Cleanup / Teardown Order

Deleting VPC resources out of order causes `DependencyViolation` errors. Follow this order:

1. Terminate EC2 instances / delete ENIs in the VPC.
2. Delete NAT Gateways (wait for `deleted` state — takes a few minutes).
3. Release Elastic IPs used by NAT Gateways.
4. Delete VPC Endpoints.
5. Delete VPN connections, then Virtual Private Gateways (detach first), then Customer Gateways.
6. Delete Transit Gateway VPC attachments (before deleting the TGW itself, if applicable).
7. Delete non-main route table associations, then the route tables themselves.
8. Detach and delete the Internet Gateway.
9. Delete subnets.
10. Delete Security Groups (except the default, which cannot be deleted) and custom NACLs.
11. Delete the VPC.

```bash
# Example teardown snippet
aws ec2 delete-nat-gateway --nat-gateway-id nat-xxxx
aws ec2 release-address --allocation-id eipalloc-xxxx
aws ec2 detach-internet-gateway --vpc-id vpc-xxxx --internet-gateway-id igw-xxxx
aws ec2 delete-internet-gateway --internet-gateway-id igw-xxxx
aws ec2 delete-subnet --subnet-id subnet-xxxx
aws ec2 delete-vpc --vpc-id vpc-xxxx
```
