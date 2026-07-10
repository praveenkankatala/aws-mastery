# ⌨️ Commands Cheatsheet — DNS & Route 53

Quick-reference commands used throughout the labs. Covers **AWS CLI**, **Terraform**,
**CloudFormation**, and plain **DNS diagnostic tools**.

> Prerequisite: AWS CLI v2 installed and configured (`aws configure`) with a profile that has
> Route 53 / EC2 / S3 permissions.

---

## 📑 Contents
- [Domain Registration](#domain-registration)
- [Hosted Zones](#hosted-zones)
- [Resource Records](#resource-records)
- [Alias Records (EC2 / S3 / CloudFront)](#alias-records)
- [Health Checks](#health-checks)
- [Route 53 Resolver (Hybrid DNS)](#route-53-resolver-hybrid-dns)
- [DNSSEC](#dnssec)
- [DNS Firewall](#dns-firewall)
- [Traffic Policies](#traffic-policies)
- [DNS Diagnostic Commands](#dns-diagnostic-commands)
- [Terraform Snippets](#terraform-snippets)
- [CloudFormation Snippet](#cloudformation-snippet)

---

## Domain Registration

```bash
# Check domain availability
aws route53domains check-domain-availability --domain-name yourname.com

# List domains you already own in Route 53
aws route53domains list-domains

# Get full detail on a registered domain (contacts, status, nameservers)
aws route53domains get-domain-detail --domain-name yourname.com

# Enable transfer lock (clientTransferProhibited)
aws route53domains enable-domain-transfer-lock --domain-name yourname.com

# Get the Auth/EPP code needed to transfer a domain OUT to another registrar
aws route53domains retrieve-domain-auth-code --domain-name yourname.com
```

## Hosted Zones

```bash
# Create a public hosted zone
aws route53 create-hosted-zone \
  --name yourname.com \
  --caller-reference "$(date +%s)"

# Create a private hosted zone (associated with a VPC)
aws route53 create-hosted-zone \
  --name internal.yourname.com \
  --caller-reference "$(date +%s)" \
  --vpc VPCRegion=us-east-1,VPCId=vpc-0123456789abcdef0 \
  --hosted-zone-config Comment="Private zone",PrivateZone=true

# List all hosted zones
aws route53 list-hosted-zones

# Get the 4 NS records + SOA for a hosted zone (copy these to your external registrar)
aws route53 get-hosted-zone --id /hostedzone/Z1234567890ABC

# Associate an additional VPC with a private hosted zone
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z1234567890ABC \
  --vpc VPCRegion=us-west-2,VPCId=vpc-0987654321fedcba0

# Delete a hosted zone (must remove all non-default records first)
aws route53 delete-hosted-zone --id /hostedzone/Z1234567890ABC
```

## Resource Records

Route 53 changes always go through a JSON **change batch** file applied with
`change-resource-record-sets`.

```bash
# List existing records in a zone
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890ABC
```

**`create-a-record.json`** — simple A record:
```json
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.yourname.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{ "Value": "203.0.113.25" }]
      }
    }
  ]
}
```

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch file://create-a-record.json
```

**`weighted-record.json`** — weighted routing (two versions of the same name):
```json
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.yourname.com",
        "Type": "A",
        "SetIdentifier": "ServerA",
        "Weight": 70,
        "TTL": 60,
        "ResourceRecords": [{ "Value": "203.0.113.25" }]
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.yourname.com",
        "Type": "A",
        "SetIdentifier": "ServerB",
        "Weight": 30,
        "TTL": 60,
        "ResourceRecords": [{ "Value": "203.0.113.50" }]
      }
    }
  ]
}
```

## Alias Records

**Point root domain to an Application Load Balancer:**
```json
{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "yourname.com",
      "Type": "A",
      "AliasTarget": {
        "HostedZoneId": "Z35SXDOTRQ7X7K",
        "DNSName": "dualstack.my-load-balancer-1234567890.us-east-1.elb.amazonaws.com",
        "EvaluateTargetHealth": true
      }
    }
  }]
}
```

**Point subdomain to an S3 static website endpoint:**
```json
{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "web.yourname.com",
      "Type": "A",
      "AliasTarget": {
        "HostedZoneId": "Z3AQBSTGFYJSTF",
        "DNSName": "s3-website-us-east-1.amazonaws.com",
        "EvaluateTargetHealth": false
      }
    }
  }]
}
```

> 🔎 `AliasTarget.HostedZoneId` is a **fixed AWS-published ID per service and region**
> (different for ALB vs S3 vs CloudFront) — look it up in the AWS docs for your target
> service/region before running this.

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch file://alias-s3-record.json
```

## Health Checks

```bash
# Create an HTTP health check with string matching
aws route53 create-health-check \
  --caller-reference "$(date +%s)" \
  --health-check-config \
      Type=HTTP,ResourcePath="/health",FullyQualifiedDomainName="app.yourname.com",Port=80,SearchString="Database Connected",RequestInterval=30,FailureThreshold=3

# List health checks
aws route53 list-health-checks

# Get current status of a health check
aws route53 get-health-check-status --health-check-id abcd1234-5678-90ab-cdef-1234567890ab

# Create a calculated (aggregate) health check over child checks
aws route53 create-health-check \
  --caller-reference "$(date +%s)" \
  --health-check-config \
      Type=CALCULATED,ChildHealthChecks=abcd1234-...,efgh5678-...,HealthThreshold=2
```

## Route 53 Resolver (Hybrid DNS)

```bash
# Create an inbound endpoint (on-prem -> AWS)
aws route53resolver create-resolver-endpoint \
  --creator-request-id "$(date +%s)" \
  --direction INBOUND \
  --security-group-ids sg-0123456789abcdef0 \
  --ip-addresses SubnetId=subnet-0111,Ip=10.0.1.10 SubnetId=subnet-0112,Ip=10.0.2.10

# Create an outbound endpoint (AWS -> on-prem)
aws route53resolver create-resolver-endpoint \
  --creator-request-id "$(date +%s)" \
  --direction OUTBOUND \
  --security-group-ids sg-0123456789abcdef0 \
  --ip-addresses SubnetId=subnet-0111,Ip=10.0.1.11 SubnetId=subnet-0112,Ip=10.0.2.11

# Create a conditional forwarding rule to on-prem DNS
aws route53resolver create-resolver-rule \
  --creator-request-id "$(date +%s)" \
  --rule-type FORWARD \
  --domain-name "corp.internal" \
  --target-ips Ip=192.168.1.10,Port=53 Ip=192.168.1.11,Port=53 \
  --resolver-endpoint-id rslvr-out-abc123

# Associate the rule with a VPC
aws route53resolver associate-resolver-rule \
  --resolver-rule-id rslvr-rr-abc123 \
  --vpc-id vpc-0123456789abcdef0
```

## DNSSEC

```bash
# Enable DNSSEC signing on a hosted zone
aws route53 enable-hosted-zone-dnssec --hosted-zone-id Z1234567890ABC

# Create a KSK (Key Signing Key) — required before enabling DNSSEC
aws route53 create-key-signing-key \
  --caller-reference "$(date +%s)" \
  --hosted-zone-id Z1234567890ABC \
  --key-management-service-arn arn:aws:kms:us-east-1:123456789012:key/abcd-1234 \
  --name ksk-yourname \
  --status ACTIVE

# View DNSSEC status
aws route53 get-dnssec --hosted-zone-id Z1234567890ABC
```

## DNS Firewall

```bash
# Create a domain list (block list)
aws route53resolver create-firewall-domain-list \
  --creator-request-id "$(date +%s)" \
  --name blocked-domains

# Add domains to the block list
aws route53resolver update-firewall-domains \
  --firewall-domain-list-id rslvr-fdl-abc123 \
  --operation ADD \
  --domains "malware-example.com" "cryptominer-example.net"

# Create a rule group and rule that blocks the list
aws route53resolver create-firewall-rule-group \
  --creator-request-id "$(date +%s)" \
  --name my-rule-group

aws route53resolver create-firewall-rule \
  --creator-request-id "$(date +%s)" \
  --firewall-rule-group-id rslvr-frg-abc123 \
  --firewall-domain-list-id rslvr-fdl-abc123 \
  --priority 100 \
  --action BLOCK \
  --block-response NXDOMAIN \
  --name block-malware

# Associate the rule group with a VPC
aws route53resolver associate-firewall-rule-group \
  --creator-request-id "$(date +%s)" \
  --firewall-rule-group-id rslvr-frg-abc123 \
  --vpc-id vpc-0123456789abcdef0 \
  --priority 101 \
  --name attach-to-vpc
```

## Traffic Policies

```bash
# Create a versioned traffic policy from a JSON document
aws route53 create-traffic-policy \
  --name multi-tier-policy \
  --document file://traffic-policy.json

# Instantiate the policy against a hosted zone as live records
aws route53 create-traffic-policy-instance \
  --hosted-zone-id Z1234567890ABC \
  --name app.yourname.com \
  --ttl 60 \
  --traffic-policy-id abcd1234-5678 \
  --traffic-policy-version 1
```

## DNS Diagnostic Commands

These work on any DNS setup (not just Route 53) — see `troubleshooting.md` for how to read
their output.

```bash
# Query an A record directly against your Route 53 name servers
dig app.yourname.com A @ns-123.awsdns-45.com

# Trace the full resolution path from the root down
dig +trace yourname.com

# Show only the answer section
dig +short yourname.com

# Query a specific record type
dig yourname.com MX
dig yourname.com TXT
dig yourname.com NS

# Classic lookup tool (works cross-platform)
nslookup app.yourname.com

# Look up WHOIS registration data
whois yourname.com

# Test propagation from many global locations (browser tool)
# https://dnschecker.org
```

## Terraform Snippets

```hcl
resource "aws_route53_zone" "main" {
  name = "yourname.com"
}

resource "aws_route53_record" "app" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.yourname.com"
  type    = "A"
  ttl     = 300
  records = ["203.0.113.25"]
}

resource "aws_route53_record" "root_alias_to_alb" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "yourname.com"
  type    = "A"

  alias {
    name                   = aws_lb.app.dns_name
    zone_id                = aws_lb.app.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_health_check" "app" {
  fqdn              = "app.yourname.com"
  port              = 80
  type              = "HTTP"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}
```

## CloudFormation Snippet

```yaml
Resources:
  HostedZone:
    Type: AWS::Route53::HostedZone
    Properties:
      Name: yourname.com

  AppRecord:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneId: !Ref HostedZone
      Name: app.yourname.com
      Type: A
      TTL: "300"
      ResourceRecords:
        - 203.0.113.25

  HealthCheck:
    Type: AWS::Route53::HealthCheck
    Properties:
      HealthCheckConfig:
        Type: HTTP
        FullyQualifiedDomainName: app.yourname.com
        Port: 80
        ResourcePath: /health
        RequestInterval: 30
        FailureThreshold: 3
```
