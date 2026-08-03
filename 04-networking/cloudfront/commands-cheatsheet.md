# Amazon CloudFront — AWS CLI Command Cheatsheet

> Every command you need, grouped by task, with real flags and copy-pasteable JSON.
> Assumes **AWS CLI v2**. CloudFront is a global service — most commands don't need `--region`,
> but anything CloudFront-*adjacent* (ACM certs, Lambda@Edge, WAF for CloudFront) **must** use
> `--region us-east-1`.

---

## Table of Contents

- [0. Setup & Conventions](#0-setup--conventions)
- [1. Distributions — Create, Read, Update, Delete](#1-distributions--create-read-update-delete)
- [2. The Safe Update Pattern (ETag handling)](#2-the-safe-update-pattern-etag-handling)
- [3. Origins & Origin Access Control](#3-origins--origin-access-control)
- [4. VPC Origins](#4-vpc-origins)
- [5. Cache Policies](#5-cache-policies)
- [6. Origin Request Policies](#6-origin-request-policies)
- [7. Response Headers Policies](#7-response-headers-policies)
- [8. Invalidations](#8-invalidations)
- [9. CloudFront Functions](#9-cloudfront-functions)
- [10. CloudFront KeyValueStore](#10-cloudfront-keyvaluestore)
- [11. Lambda@Edge](#11-lambdaedge)
- [12. Signed URLs, Public Keys & Key Groups](#12-signed-urls-public-keys--key-groups)
- [13. Certificates (ACM) & Custom Domains](#13-certificates-acm--custom-domains)
- [14. Route 53 DNS](#14-route-53-dns)
- [15. AWS WAF for CloudFront](#15-aws-waf-for-cloudfront)
- [16. Logging & Monitoring](#16-logging--monitoring)
- [17. Origin Shield, Price Class & Compression](#17-origin-shield-price-class--compression)
- [18. Continuous Deployment](#18-continuous-deployment)
- [19. Field-Level Encryption](#19-field-level-encryption)
- [20. Anycast Static IPs & Distribution Tenants](#20-anycast-static-ips--distribution-tenants)
- [21. S3 Commands You'll Use Alongside CloudFront](#21-s3-commands-youll-use-alongside-cloudfront)
- [22. Tagging](#22-tagging)
- [23. Quotas & Service Limits](#23-quotas--service-limits)
- [24. Testing & Debugging with curl / dig / openssl](#24-testing--debugging-with-curl--dig--openssl)
- [25. One-Liners & Recipes](#25-one-liners--recipes)
- [26. Terraform Quick Reference](#26-terraform-quick-reference)

---

## 0. Setup & Conventions

```bash
# Verify CLI version (must be v2)
aws --version

# Verify identity
aws sts get-caller-identity

# Install jq — you will need it constantly
sudo apt install jq -y          # Ubuntu/Debian
brew install jq                 # macOS
```

### Environment variables used throughout this document

```bash
export AWS_REGION="ap-south-1"                     # your working region
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET="my-cf-demo-$ACCOUNT_ID"
export DIST_ID="E1234ABCDEFGH"                     # replace after creation
export DOMAIN="cdn.example.com"
```

### Two universal helpers

```bash
# Get a distribution's domain name
cfdomain() { aws cloudfront get-distribution --id "$1" \
  --query 'Distribution.DomainName' --output text; }

# Get a distribution's current ETag
cfetag() { aws cloudfront get-distribution-config --id "$1" \
  --query 'ETag' --output text; }
```

### Output formatting flags worth memorizing

```bash
--output json|table|text|yaml
--query '<JMESPath>'          # server-independent filtering, always available
--no-cli-pager                # stop the CLI from opening less
--no-paginate                 # single API page only
--dry-run                     # not supported on CloudFront APIs, but is on EC2
```

---

## 1. Distributions — Create, Read, Update, Delete

### List and inspect

```bash
# All distributions, compact table
aws cloudfront list-distributions \
  --query 'DistributionList.Items[].{Id:Id,Domain:DomainName,Status:Status,Enabled:Enabled,Aliases:Aliases.Items,Origin:Origins.Items[0].DomainName}' \
  --output table

# Just the IDs
aws cloudfront list-distributions --query 'DistributionList.Items[].Id' --output text

# Find a distribution by its alternate domain name
aws cloudfront list-distributions \
  --query "DistributionList.Items[?contains(Aliases.Items||\`[]\`, 'cdn.example.com')].Id" \
  --output text

# Find a distribution by origin domain
aws cloudfront list-distributions \
  --query "DistributionList.Items[?Origins.Items[?contains(DomainName,'my-bucket')]].{Id:Id,Domain:DomainName}" \
  --output table

# Full detail for one distribution
aws cloudfront get-distribution --id $DIST_ID

# Just the config (this is what you edit)
aws cloudfront get-distribution-config --id $DIST_ID

# Status only — poll this after an update
aws cloudfront get-distribution --id $DIST_ID --query 'Distribution.Status' --output text

# Block until fully deployed
aws cloudfront wait distribution-deployed --id $DIST_ID

# List distributions by web ACL
aws cloudfront list-distributions-by-web-acl-id --web-acl-id <acl-id>

# List distributions using a specific cache policy / origin request policy / key group
aws cloudfront list-distributions-by-cache-policy-id       --cache-policy-id <id>
aws cloudfront list-distributions-by-origin-request-policy-id --origin-request-policy-id <id>
aws cloudfront list-distributions-by-response-headers-policy-id --response-headers-policy-id <id>
aws cloudfront list-distributions-by-key-group             --key-group-id <id>
aws cloudfront list-distributions-by-realtime-log-config   --realtime-log-config-name <name>
```

### Create — the quickest possible distribution

```bash
# Minimal: S3 origin, all defaults, HTTPS redirect
aws cloudfront create-distribution \
  --origin-domain-name "$BUCKET.s3.$AWS_REGION.amazonaws.com" \
  --default-root-object index.html
```

That shorthand is fine for a first test but gives you a public-bucket configuration.
For anything real, use a config file.

### Create from a JSON config file (the way you should do it)

`distribution-config.json`:

```json
{
  "CallerReference": "cf-demo-2026-08-03-001",
  "Comment": "Static site — S3 + OAC",
  "Enabled": true,
  "DefaultRootObject": "index.html",
  "PriceClass": "PriceClass_All",
  "HttpVersion": "http2and3",
  "IsIPV6Enabled": true,
  "Aliases": { "Quantity": 0 },
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "s3-origin",
        "DomainName": "REPLACE-BUCKET.s3.ap-south-1.amazonaws.com",
        "OriginPath": "",
        "OriginAccessControlId": "REPLACE-OAC-ID",
        "S3OriginConfig": { "OriginAccessIdentity": "" },
        "ConnectionAttempts": 3,
        "ConnectionTimeout": 10,
        "OriginShield": { "Enabled": false },
        "CustomHeaders": { "Quantity": 0 }
      }
    ]
  },
  "OriginGroups": { "Quantity": 0 },
  "DefaultCacheBehavior": {
    "TargetOriginId": "s3-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"],
      "CachedMethods": { "Quantity": 2, "Items": ["GET", "HEAD"] }
    },
    "Compress": true,
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "OriginRequestPolicyId": "88a5eaf4-2fd4-4709-b370-b4c650ea3fcf",
    "ResponseHeadersPolicyId": "67f7725c-6f97-4210-82d7-5512b31e9d03",
    "SmoothStreaming": false,
    "FieldLevelEncryptionId": "",
    "FunctionAssociations": { "Quantity": 0 },
    "LambdaFunctionAssociations": { "Quantity": 0 },
    "TrustedKeyGroups": { "Enabled": false, "Quantity": 0 }
  },
  "CacheBehaviors": { "Quantity": 0 },
  "CustomErrorResponses": {
    "Quantity": 2,
    "Items": [
      { "ErrorCode": 403, "ResponsePagePath": "/index.html",
        "ResponseCode": "200", "ErrorCachingMinTTL": 10 },
      { "ErrorCode": 404, "ResponsePagePath": "/index.html",
        "ResponseCode": "200", "ErrorCachingMinTTL": 10 }
    ]
  },
  "ViewerCertificate": {
    "CloudFrontDefaultCertificate": true,
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "Restrictions": {
    "GeoRestriction": { "RestrictionType": "none", "Quantity": 0 }
  },
  "Logging": { "Enabled": false, "IncludeCookies": false, "Bucket": "", "Prefix": "" },
  "WebACLId": ""
}
```

> **Managed policy IDs shown above** are the commonly cited values for `CachingOptimized`,
> `CORS-S3Origin`, and `SecurityHeadersPolicy`. **Verify them for your account** with
> `aws cloudfront list-cache-policies --type managed` before relying on them in automation.

```bash
aws cloudfront create-distribution \
  --distribution-config file://distribution-config.json
```

### Create with tags in one call

```bash
cat > dist-with-tags.json <<'EOF'
{
  "DistributionConfig": { ...paste the config above... },
  "Tags": {
    "Items": [
      { "Key": "Environment", "Value": "production" },
      { "Key": "Owner",       "Value": "platform-team" },
      { "Key": "CostCenter",  "Value": "CC-1042" }
    ]
  }
}
EOF

aws cloudfront create-distribution-with-tags \
  --distribution-config-with-tags file://dist-with-tags.json
```

### Delete a distribution (the three-step dance)

```bash
# STEP 1 — disable it
ETAG=$(aws cloudfront get-distribution-config --id $DIST_ID --query ETag --output text)
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig' > dist.json
jq '.Enabled = false' dist.json > dist-disabled.json
aws cloudfront update-distribution \
  --id $DIST_ID --if-match $ETAG --distribution-config file://dist-disabled.json

# STEP 2 — wait for Deployed (can take several minutes)
aws cloudfront wait distribution-deployed --id $DIST_ID

# STEP 3 — delete with the NEW ETag
NEW_ETAG=$(aws cloudfront get-distribution-config --id $DIST_ID --query ETag --output text)
aws cloudfront delete-distribution --id $DIST_ID --if-match $NEW_ETAG
```

---

## 2. The Safe Update Pattern (ETag handling)

Every CloudFront update needs `--if-match <ETag>`. Getting this wrong produces
`PreconditionFailed` or `InvalidArgument: Unknown field DistributionConfig`.

**The reusable pattern — copy this into your notes:**

```bash
cf_update() {
  local DIST_ID="$1" JQ_FILTER="$2"

  # 1. Fetch config + ETag together
  aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/cf-full.json
  local ETAG=$(jq -r '.ETag' /tmp/cf-full.json)

  # 2. Extract ONLY the DistributionConfig object — this is the #1 gotcha
  jq '.DistributionConfig' /tmp/cf-full.json > /tmp/cf-config.json

  # 3. Apply the change
  jq "$JQ_FILTER" /tmp/cf-config.json > /tmp/cf-new.json

  # 4. Push it back
  aws cloudfront update-distribution \
    --id "$DIST_ID" \
    --if-match "$ETAG" \
    --distribution-config file:///tmp/cf-new.json
}

# Examples
cf_update $DIST_ID '.Comment = "Updated via CLI"'
cf_update $DIST_ID '.DefaultCacheBehavior.Compress = true'
cf_update $DIST_ID '.PriceClass = "PriceClass_200"'
cf_update $DIST_ID '.Enabled = false'
cf_update $DIST_ID '.DefaultCacheBehavior.ViewerProtocolPolicy = "https-only"'
cf_update $DIST_ID '.HttpVersion = "http2and3"'
cf_update $DIST_ID '.Origins.Items[0].OriginPath = "/prod"'
cf_update $DIST_ID '.Origins.Items[0].CustomOriginConfig.OriginReadTimeout = 60'
cf_update $DIST_ID '.WebACLId = "arn:aws:wafv2:us-east-1:1111:global/webacl/my-acl/abc"'
```

**The three classic errors:**

| Error | Cause |
|---|---|
| `PreconditionFailed` | Your ETag is stale — someone (or you, in another shell) changed it. Re-fetch. |
| `Unknown field DistributionConfig` | You passed the whole `get-distribution-config` output instead of just `.DistributionConfig`. |
| `InconsistentQuantities` | A `Quantity` field doesn't match the length of its `Items` array. Always update both. |

**Fixing Quantity/Items together:**

```bash
# Add an alias correctly
jq '.Aliases.Items += ["cdn.example.com"] | .Aliases.Quantity = (.Aliases.Items | length)' \
  /tmp/cf-config.json > /tmp/cf-new.json
```

---

## 3. Origins & Origin Access Control

### Create an OAC

```bash
aws cloudfront create-origin-access-control --origin-access-control-config '{
  "Name": "s3-oac-my-site",
  "Description": "OAC for the static site bucket",
  "SigningProtocol": "sigv4",
  "SigningBehavior": "always",
  "OriginAccessControlOriginType": "s3"
}'
# OriginAccessControlOriginType: s3 | mediastore | lambda | mediapackagev2
# SigningBehavior:               always | never | no-override
```

```bash
# List / get / update / delete
aws cloudfront list-origin-access-controls \
  --query 'OriginAccessControlList.Items[].{Id:Id,Name:Name,Type:OriginAccessControlOriginType}' \
  --output table

aws cloudfront get-origin-access-control --id E2ABCDEFGHIJKL

OAC_ETAG=$(aws cloudfront get-origin-access-control --id E2ABC... --query ETag --output text)
aws cloudfront delete-origin-access-control --id E2ABC... --if-match $OAC_ETAG
```

### Apply the S3 bucket policy for OAC

```bash
cat > oac-bucket-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipalReadOnly",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::$ACCOUNT_ID:distribution/$DIST_ID"
      }
    }
  }]
}
EOF

aws s3api put-bucket-policy --bucket $BUCKET --policy file://oac-bucket-policy.json
aws s3api get-bucket-policy --bucket $BUCKET --query Policy --output text | jq .
```

**If the bucket uses SSE-KMS, also add to the KMS key policy:**

```json
{
  "Sid": "AllowCloudFrontDecrypt",
  "Effect": "Allow",
  "Principal": { "Service": "cloudfront.amazonaws.com" },
  "Action": ["kms:Decrypt", "kms:GenerateDataKey*"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::111122223333:distribution/E1234ABCDEFGH"
    }
  }
}
```

### Legacy: Origin Access Identity (OAI)

```bash
aws cloudfront create-cloud-front-origin-access-identity \
  --cloud-front-origin-access-identity-config \
  CallerReference=oai-$(date +%s),Comment="legacy OAI"

aws cloudfront list-cloud-front-origin-access-identities
aws cloudfront get-cloud-front-origin-access-identity --id E1ABCDEFGHIJKL
aws cloudfront delete-cloud-front-origin-access-identity --id E1ABC... --if-match $ETAG
```

### Add a custom (non-S3) origin via jq

```bash
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)
jq '.DistributionConfig' /tmp/full.json > /tmp/cfg.json

jq '.Origins.Items += [{
      "Id": "alb-api",
      "DomainName": "my-alb-123456.ap-south-1.elb.amazonaws.com",
      "OriginPath": "",
      "CustomHeaders": {
        "Quantity": 1,
        "Items": [{ "HeaderName": "X-Origin-Verify", "HeaderValue": "s3cr3t-value" }]
      },
      "CustomOriginConfig": {
        "HTTPPort": 80,
        "HTTPSPort": 443,
        "OriginProtocolPolicy": "https-only",
        "OriginSslProtocols": { "Quantity": 1, "Items": ["TLSv1.2"] },
        "OriginReadTimeout": 60,
        "OriginKeepaliveTimeout": 30
      },
      "ConnectionAttempts": 3,
      "ConnectionTimeout": 10,
      "OriginShield": { "Enabled": false }
    }]
    | .Origins.Quantity = (.Origins.Items | length)' /tmp/cfg.json > /tmp/new.json

aws cloudfront update-distribution --id $DIST_ID --if-match $ETAG \
  --distribution-config file:///tmp/new.json
```

### Add a cache behavior for that origin

```bash
jq '.CacheBehaviors.Items = ((.CacheBehaviors.Items // []) + [{
      "PathPattern": "/api/*",
      "TargetOriginId": "alb-api",
      "ViewerProtocolPolicy": "https-only",
      "AllowedMethods": {
        "Quantity": 7,
        "Items": ["GET","HEAD","OPTIONS","PUT","POST","PATCH","DELETE"],
        "CachedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] }
      },
      "Compress": true,
      "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
      "OriginRequestPolicyId": "b689b0a8-53d0-40ab-baf2-68738e2966ac",
      "SmoothStreaming": false,
      "FieldLevelEncryptionId": "",
      "FunctionAssociations": { "Quantity": 0 },
      "LambdaFunctionAssociations": { "Quantity": 0 },
      "TrustedKeyGroups": { "Enabled": false, "Quantity": 0 }
    }])
    | .CacheBehaviors.Quantity = (.CacheBehaviors.Items | length)' \
  /tmp/cfg.json > /tmp/new.json
```

### Create an origin group (failover)

```bash
jq '.OriginGroups = {
      "Quantity": 1,
      "Items": [{
        "Id": "api-failover-group",
        "FailoverCriteria": {
          "StatusCodes": { "Quantity": 4, "Items": [500, 502, 503, 504] }
        },
        "Members": {
          "Quantity": 2,
          "Items": [
            { "OriginId": "alb-primary" },
            { "OriginId": "alb-secondary" }
          ]
        }
      }]
    }
    | .DefaultCacheBehavior.TargetOriginId = "api-failover-group"' \
  /tmp/cfg.json > /tmp/new.json
```

---

## 4. VPC Origins

Serve from private ALB / NLB / EC2 with no public exposure.

```bash
# Create a VPC origin
aws cloudfront create-vpc-origin --vpc-origin-endpoint-config '{
  "Name": "internal-alb-origin",
  "Arn": "arn:aws:elasticloadbalancing:ap-south-1:111122223333:loadbalancer/app/internal-alb/abc123",
  "HTTPPort": 80,
  "HTTPSPort": 443,
  "OriginProtocolPolicy": "http-only",
  "OriginSslProtocols": { "Quantity": 1, "Items": ["TLSv1.2"] }
}'

# List / inspect
aws cloudfront list-vpc-origins \
  --query 'VpcOriginList.Items[].{Id:Id,Name:Name,Status:Status,Arn:Arn}' --output table
aws cloudfront get-vpc-origin --id vo-0123456789abcdef

# Update
VO_ETAG=$(aws cloudfront get-vpc-origin --id vo-01234 --query ETag --output text)
aws cloudfront update-vpc-origin --id vo-01234 --if-match $VO_ETAG \
  --vpc-origin-endpoint-config file://vpc-origin.json

# Delete
aws cloudfront delete-vpc-origin --id vo-01234 --if-match $VO_ETAG
```

**Reference it from the distribution:**

```bash
jq '.Origins.Items += [{
      "Id": "vpc-alb",
      "DomainName": "internal-alb-123.ap-south-1.elb.amazonaws.com",
      "VpcOriginConfig": { "VpcOriginId": "vo-0123456789abcdef" },
      "CustomHeaders": { "Quantity": 0 },
      "ConnectionAttempts": 3,
      "ConnectionTimeout": 10,
      "OriginShield": { "Enabled": false }
    }]
    | .Origins.Quantity = (.Origins.Items | length)' /tmp/cfg.json > /tmp/new.json
```

**Security group on the ALB** must allow inbound from the VPC origin's managed ENIs — the console
wizard handles this; if you build it manually, allow the VPC origin security group.

---

## 5. Cache Policies

```bash
# List AWS-managed policies (do this before you build your own)
aws cloudfront list-cache-policies --type managed \
  --query 'CachePolicyList.Items[].{Name:CachePolicy.CachePolicyConfig.Name,Id:CachePolicy.Id}' \
  --output table

# List your custom policies
aws cloudfront list-cache-policies --type custom \
  --query 'CachePolicyList.Items[].{Name:CachePolicy.CachePolicyConfig.Name,Id:CachePolicy.Id}' \
  --output table

# Get one
aws cloudfront get-cache-policy --id 658327ea-f89d-4fab-a63d-7e88639e58f6
aws cloudfront get-cache-policy-config --id <id>
```

### Create a custom cache policy

```bash
cat > cache-policy.json <<'EOF'
{
  "Name": "AppCachePolicy-QueryAndLang",
  "Comment": "Cache on id+page query strings and Accept-Language",
  "DefaultTTL": 86400,
  "MaxTTL": 31536000,
  "MinTTL": 0,
  "ParametersInCacheKeyAndForwardedToOrigin": {
    "EnableAcceptEncodingGzip": true,
    "EnableAcceptEncodingBrotli": true,
    "HeadersConfig": {
      "HeaderBehavior": "whitelist",
      "Headers": { "Quantity": 1, "Items": ["Accept-Language"] }
    },
    "CookiesConfig": { "CookieBehavior": "none" },
    "QueryStringsConfig": {
      "QueryStringBehavior": "whitelist",
      "QueryStrings": { "Quantity": 2, "Items": ["id", "page"] }
    }
  }
}
EOF

aws cloudfront create-cache-policy --cache-policy-config file://cache-policy.json
```

**Behavior enums:**

```
HeaderBehavior:      none | whitelist
CookieBehavior:      none | whitelist | allExcept | all
QueryStringBehavior: none | whitelist | allExcept | all
```

### Update / delete

```bash
CP_ETAG=$(aws cloudfront get-cache-policy --id <id> --query ETag --output text)
aws cloudfront update-cache-policy --id <id> --if-match $CP_ETAG \
  --cache-policy-config file://cache-policy.json

aws cloudfront delete-cache-policy --id <id> --if-match $CP_ETAG
# Fails if any distribution still references it — detach first.
```

### Attach a cache policy to a behavior

```bash
cf_update $DIST_ID '.DefaultCacheBehavior.CachePolicyId = "NEW-POLICY-ID"'

# Removing legacy settings when migrating from ForwardedValues:
jq 'del(.DefaultCacheBehavior.ForwardedValues)
    | del(.DefaultCacheBehavior.MinTTL)
    | del(.DefaultCacheBehavior.DefaultTTL)
    | del(.DefaultCacheBehavior.MaxTTL)
    | .DefaultCacheBehavior.CachePolicyId = "NEW-POLICY-ID"' \
  /tmp/cfg.json > /tmp/new.json
```

---

## 6. Origin Request Policies

```bash
aws cloudfront list-origin-request-policies --type managed \
  --query 'OriginRequestPolicyList.Items[].{Name:OriginRequestPolicy.OriginRequestPolicyConfig.Name,Id:OriginRequestPolicy.Id}' \
  --output table

aws cloudfront get-origin-request-policy --id <id>
```

```bash
cat > orp.json <<'EOF'
{
  "Name": "ApiOriginRequestPolicy",
  "Comment": "Forward everything except Host, plus viewer geo headers",
  "HeadersConfig": {
    "HeaderBehavior": "allViewerAndWhitelistCloudFront",
    "Headers": {
      "Quantity": 3,
      "Items": ["CloudFront-Viewer-Country", "CloudFront-Is-Mobile-Viewer", "CloudFront-Viewer-City"]
    }
  },
  "CookiesConfig": { "CookieBehavior": "all" },
  "QueryStringsConfig": { "QueryStringBehavior": "all" }
}
EOF

aws cloudfront create-origin-request-policy --origin-request-policy-config file://orp.json
```

**Behavior enums:**

```
HeaderBehavior:      none | whitelist | allViewer | allViewerAndWhitelistCloudFront | allExcept
CookieBehavior:      none | whitelist | all | allExcept
QueryStringBehavior: none | whitelist | all | allExcept
```

```bash
ORP_ETAG=$(aws cloudfront get-origin-request-policy --id <id> --query ETag --output text)
aws cloudfront update-origin-request-policy --id <id> --if-match $ORP_ETAG \
  --origin-request-policy-config file://orp.json
aws cloudfront delete-origin-request-policy --id <id> --if-match $ORP_ETAG
```

---

## 7. Response Headers Policies

```bash
aws cloudfront list-response-headers-policies --type managed \
  --query 'ResponseHeadersPolicyList.Items[].{Name:ResponseHeadersPolicy.ResponseHeadersPolicyConfig.Name,Id:ResponseHeadersPolicy.Id}' \
  --output table
```

```bash
cat > rhp.json <<'EOF'
{
  "Name": "SecureSiteHeaders",
  "Comment": "HSTS + CSP + CORS + strip server headers",
  "SecurityHeadersConfig": {
    "StrictTransportSecurity": {
      "AccessControlMaxAgeSec": 63072000,
      "IncludeSubdomains": true,
      "Preload": true,
      "Override": true
    },
    "ContentTypeOptions": { "Override": true },
    "FrameOptions": { "FrameOption": "DENY", "Override": true },
    "ReferrerPolicy": { "ReferrerPolicy": "strict-origin-when-cross-origin", "Override": true },
    "XSSProtection": { "Protection": true, "ModeBlock": true, "Override": true },
    "ContentSecurityPolicy": {
      "ContentSecurityPolicy": "default-src 'self'; img-src 'self' data:; script-src 'self'",
      "Override": true
    }
  },
  "CorsConfig": {
    "AccessControlAllowOrigins": { "Quantity": 1, "Items": ["https://app.example.com"] },
    "AccessControlAllowHeaders": { "Quantity": 1, "Items": ["*"] },
    "AccessControlAllowMethods": { "Quantity": 3, "Items": ["GET", "HEAD", "OPTIONS"] },
    "AccessControlAllowCredentials": false,
    "AccessControlExposeHeaders": { "Quantity": 1, "Items": ["ETag"] },
    "AccessControlMaxAgeSec": 600,
    "OriginOverride": true
  },
  "CustomHeadersConfig": {
    "Quantity": 1,
    "Items": [{ "Header": "X-Delivered-By", "Value": "CloudFront", "Override": true }]
  },
  "RemoveHeadersConfig": {
    "Quantity": 2,
    "Items": [{ "Header": "Server" }, { "Header": "X-Powered-By" }]
  },
  "ServerTimingHeadersConfig": { "Enabled": true, "SamplingRate": 10.0 }
}
EOF

aws cloudfront create-response-headers-policy --response-headers-policy-config file://rhp.json

# Attach it
cf_update $DIST_ID '.DefaultCacheBehavior.ResponseHeadersPolicyId = "NEW-RHP-ID"'
```

**FrameOption values:** `DENY` | `SAMEORIGIN`
**ReferrerPolicy values:** `no-referrer`, `no-referrer-when-downgrade`, `origin`,
`origin-when-cross-origin`, `same-origin`, `strict-origin`, `strict-origin-when-cross-origin`,
`unsafe-url`

---

## 8. Invalidations

```bash
# Single path
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/index.html"

# Multiple paths
aws cloudfront create-invalidation --distribution-id $DIST_ID \
  --paths "/index.html" "/css/*" "/js/*" "/api/config.json"

# Everything (costs 1 path; use sparingly)
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/*"

# From a JSON batch file
cat > inv.json <<'EOF'
{
  "Paths": { "Quantity": 3, "Items": ["/index.html", "/manifest.json", "/sw.js"] },
  "CallerReference": "deploy-2026-08-03-1430"
}
EOF
aws cloudfront create-invalidation --distribution-id $DIST_ID --invalidation-batch file://inv.json

# Track it
aws cloudfront list-invalidations --distribution-id $DIST_ID \
  --query 'InvalidationList.Items[].{Id:Id,Status:Status,Created:CreateTime}' --output table

aws cloudfront get-invalidation --distribution-id $DIST_ID --id I2ABCDEFGHIJ

# Block until complete
aws cloudfront wait invalidation-completed --distribution-id $DIST_ID --id I2ABCDEFGHIJ
```

### Deploy-and-invalidate one-liner

```bash
aws s3 sync ./dist "s3://$BUCKET" --delete \
  --cache-control "public, max-age=31536000, immutable" \
  --exclude "index.html" --exclude "*.json" &&
aws s3 cp ./dist/index.html "s3://$BUCKET/index.html" \
  --cache-control "no-cache" --content-type "text/html" &&
aws cloudfront create-invalidation --distribution-id "$DIST_ID" --paths "/index.html" \
  --query 'Invalidation.Id' --output text
```

### Wait for an invalidation to finish in a pipeline

```bash
INV_ID=$(aws cloudfront create-invalidation --distribution-id $DIST_ID \
  --paths "/index.html" --query 'Invalidation.Id' --output text)
aws cloudfront wait invalidation-completed --distribution-id $DIST_ID --id "$INV_ID"
echo "Invalidation $INV_ID complete"
```

**Escaping note:** paths with special characters must be URL-encoded, and paths with `*` should be
quoted so the shell doesn't glob them.

```bash
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/images/my%20file.jpg"
```

---

## 9. CloudFront Functions

```bash
# Write the function
cat > uri-rewrite.js <<'EOF'
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    if (uri.endsWith('/')) {
        request.uri += 'index.html';
    } else if (!uri.includes('.')) {
        request.uri += '/index.html';
    }
    return request;
}
EOF

# Create (lands in DEVELOPMENT stage)
aws cloudfront create-function \
  --name uri-rewrite \
  --function-config '{"Comment":"Append index.html to directory URIs","Runtime":"cloudfront-js-2.0"}' \
  --function-code fileb://uri-rewrite.js
```

### Test before you publish

```bash
cat > test-event.json <<'EOF'
{
  "version": "1.0",
  "context": { "eventType": "viewer-request" },
  "viewer": { "ip": "1.2.3.4" },
  "request": {
    "method": "GET",
    "uri": "/about/",
    "headers": { "host": { "value": "cdn.example.com" } },
    "querystring": {},
    "cookies": {}
  }
}
EOF

FN_ETAG=$(aws cloudfront describe-function --name uri-rewrite --query ETag --output text)

aws cloudfront test-function \
  --name uri-rewrite \
  --if-match "$FN_ETAG" \
  --stage DEVELOPMENT \
  --event-object fileb://test-event.json \
  --query 'TestResult.{Output:FunctionOutput,Errors:FunctionErrorMessage,Time:ComputeUtilization}' \
  --output json
```

`ComputeUtilization` is a percentage of the 1 ms budget. Keep it well under 100.

### Publish, associate, iterate

```bash
# Publish DEVELOPMENT → LIVE
FN_ETAG=$(aws cloudfront describe-function --name uri-rewrite --query ETag --output text)
aws cloudfront publish-function --name uri-rewrite --if-match "$FN_ETAG"

# Get the ARN
FN_ARN=$(aws cloudfront describe-function --name uri-rewrite \
  --query 'FunctionSummary.FunctionMetadata.FunctionARN' --output text)

# Associate with the default cache behavior
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)
jq --arg arn "$FN_ARN" '.DistributionConfig
   | .DefaultCacheBehavior.FunctionAssociations = {
       "Quantity": 1,
       "Items": [{ "FunctionARN": $arn, "EventType": "viewer-request" }]
     }' /tmp/full.json > /tmp/new.json
aws cloudfront update-distribution --id $DIST_ID --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json
```

```bash
# Update the code (goes back to DEVELOPMENT; must re-publish)
FN_ETAG=$(aws cloudfront describe-function --name uri-rewrite --query ETag --output text)
aws cloudfront update-function \
  --name uri-rewrite --if-match "$FN_ETAG" \
  --function-config '{"Comment":"v2","Runtime":"cloudfront-js-2.0"}' \
  --function-code fileb://uri-rewrite.js

# List / inspect / fetch code
aws cloudfront list-functions --stage LIVE \
  --query 'FunctionList.Items[].{Name:Name,Stage:FunctionMetadata.Stage,Runtime:FunctionConfig.Runtime}' \
  --output table
aws cloudfront describe-function --name uri-rewrite --stage LIVE
aws cloudfront get-function --name uri-rewrite --stage LIVE outfile.js

# Delete (must be disassociated from all distributions first)
aws cloudfront delete-function --name uri-rewrite --if-match "$FN_ETAG"
```

### Read CloudFront Functions logs

`console.log()` output goes to CloudWatch Logs in **us-east-1** under
`/aws/cloudfront/function/<function-name>`.

```bash
aws logs tail /aws/cloudfront/function/uri-rewrite --follow --region us-east-1
aws logs filter-log-events --region us-east-1 \
  --log-group-name /aws/cloudfront/function/uri-rewrite \
  --start-time $(( ($(date +%s) - 3600) * 1000 ))
```

**Event object shapes:**

```jsonc
// viewer-request
{ "version":"1.0","context":{"distributionDomainName":"","distributionId":"","eventType":"viewer-request","requestId":""},
  "viewer":{"ip":""},
  "request":{"method":"GET","uri":"/","querystring":{},"headers":{},"cookies":{}} }

// viewer-response — adds:
{ "response":{"statusCode":200,"statusDescription":"OK","headers":{},"cookies":{}} }
```

---

## 10. CloudFront KeyValueStore

```bash
# Create a store
aws cloudfront-keyvaluestore --help          # data-plane commands live here
aws cloudfront create-key-value-store \
  --name redirects-kvs --comment "URL redirect map"

# Get its ARN and current ETag
KVS_ARN=$(aws cloudfront describe-key-value-store --name redirects-kvs \
  --query 'KeyValueStore.ARN' --output text)
KVS_ETAG=$(aws cloudfront-keyvaluestore describe-key-value-store --kvs-arn "$KVS_ARN" \
  --query ETag --output text)

# Put a key
aws cloudfront-keyvaluestore put-key \
  --kvs-arn "$KVS_ARN" --if-match "$KVS_ETAG" \
  --key "/old-page" --value "https://example.com/new-page"

# Batch update
aws cloudfront-keyvaluestore update-keys \
  --kvs-arn "$KVS_ARN" --if-match "$KVS_ETAG" \
  --puts '[{"Key":"/a","Value":"/z"},{"Key":"/b","Value":"/y"}]' \
  --deletes '[{"Key":"/obsolete"}]'

# Read
aws cloudfront-keyvaluestore get-key --kvs-arn "$KVS_ARN" --key "/old-page"
aws cloudfront-keyvaluestore list-keys --kvs-arn "$KVS_ARN"
aws cloudfront-keyvaluestore delete-key --kvs-arn "$KVS_ARN" --if-match "$KVS_ETAG" --key "/a"

# Management-plane
aws cloudfront list-key-value-stores
aws cloudfront update-key-value-store --name redirects-kvs --comment "updated" --if-match <etag>
aws cloudfront delete-key-value-store --name redirects-kvs --if-match <etag>
```

**Associate the store with a function** (in the function config):

```bash
aws cloudfront update-function \
  --name redirect-fn --if-match "$FN_ETAG" \
  --function-config "{\"Comment\":\"redirects\",\"Runtime\":\"cloudfront-js-2.0\",\"KeyValueStoreAssociations\":{\"Quantity\":1,\"Items\":[{\"KeyValueStoreARN\":\"$KVS_ARN\"}]}}" \
  --function-code fileb://redirect-fn.js
```

---

## 11. Lambda@Edge

> **Everything here must be in `us-east-1`.**

```bash
# 1. Trust policy — BOTH principals are required
cat > trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": ["lambda.amazonaws.com", "edgelambda.amazonaws.com"] },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role --role-name lambda-edge-role \
  --assume-role-policy-document file://trust.json

aws iam attach-role-policy --role-name lambda-edge-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```bash
# 2. Package and create — us-east-1 only
cat > index.mjs <<'EOF'
export const handler = async (event) => {
  const request = event.Records[0].cf.request;
  const country = request.headers['cloudfront-viewer-country']?.[0]?.value ?? 'XX';
  request.headers['x-viewer-country'] = [{ key: 'X-Viewer-Country', value: country }];
  return request;
};
EOF
zip function.zip index.mjs

aws lambda create-function --region us-east-1 \
  --function-name cf-edge-country \
  --runtime nodejs22.x \
  --role arn:aws:iam::$ACCOUNT_ID:role/lambda-edge-role \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --timeout 5 --memory-size 128
```

```bash
# 3. PUBLISH A VERSION — you cannot associate $LATEST
LE_ARN=$(aws lambda publish-version --region us-east-1 \
  --function-name cf-edge-country --query 'FunctionArn' --output text)
echo "$LE_ARN"        # ends in :1, :2, ...
```

```bash
# 4. Associate with a cache behavior
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)
jq --arg arn "$LE_ARN" '.DistributionConfig
   | .DefaultCacheBehavior.LambdaFunctionAssociations = {
       "Quantity": 1,
       "Items": [{ "LambdaFunctionARN": $arn, "EventType": "origin-request", "IncludeBody": false }]
     }' /tmp/full.json > /tmp/new.json
aws cloudfront update-distribution --id $DIST_ID --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json
```

**EventType values:** `viewer-request` | `origin-request` | `origin-response` | `viewer-response`

```bash
# 5. Update code and publish a new version
aws lambda update-function-code --region us-east-1 \
  --function-name cf-edge-country --zip-file fileb://function.zip
NEW_ARN=$(aws lambda publish-version --region us-east-1 \
  --function-name cf-edge-country --query FunctionArn --output text)
# then re-associate NEW_ARN on the distribution
```

### Reading Lambda@Edge logs (they're scattered across regions)

```bash
for R in us-east-1 us-west-2 eu-west-1 ap-south-1 ap-southeast-1 ap-northeast-1 sa-east-1; do
  echo "=== $R ==="
  aws logs describe-log-groups --region $R \
    --log-group-name-prefix "/aws/lambda/us-east-1.cf-edge-country" \
    --query 'logGroups[].logGroupName' --output text
done

aws logs tail /aws/lambda/us-east-1.cf-edge-country --region ap-south-1 --follow
```

### Deleting Lambda@Edge (the slow part)

```bash
# 1. Remove the association from every distribution
cf_update $DIST_ID '.DefaultCacheBehavior.LambdaFunctionAssociations = {"Quantity":0,"Items":[]}'
aws cloudfront wait distribution-deployed --id $DIST_ID

# 2. Wait for replicas to be deleted — this can take SEVERAL HOURS
#    Retry until it succeeds:
aws lambda delete-function --region us-east-1 --function-name cf-edge-country
# "Lambda was unable to delete ... because it is a replicated function" → wait longer
```

---

## 12. Signed URLs, Public Keys & Key Groups

```bash
# 1. Generate an RSA key pair
openssl genrsa -out private_key.pem 2048
openssl rsa -pubout -in private_key.pem -out public_key.pem

# 2. Upload the PUBLIC key
PUBKEY=$(cat public_key.pem)
aws cloudfront create-public-key --public-key-config "{
  \"CallerReference\": \"pk-$(date +%s)\",
  \"Name\": \"signing-key-2026\",
  \"EncodedKey\": \"$PUBKEY\",
  \"Comment\": \"Signed URL key\"
}"
# → note the Id (this is your Key-Pair-Id, e.g. K2JCJMDEHXQW5F)

# 3. Create a key group containing it
aws cloudfront create-key-group --key-group-config '{
  "Name": "premium-content-keys",
  "Items": ["K2JCJMDEHXQW5F"],
  "Comment": "Keys trusted for /premium/*"
}'
```

```bash
# 4. Attach the key group to a cache behavior (restrict viewer access)
jq --arg kg "KEY-GROUP-ID" '.DistributionConfig
   | .DefaultCacheBehavior.TrustedKeyGroups = {
       "Enabled": true, "Quantity": 1, "Items": [$kg]
     }' /tmp/full.json > /tmp/new.json
```

```bash
# Management
aws cloudfront list-public-keys \
  --query 'PublicKeyList.Items[].{Id:Id,Name:Name}' --output table
aws cloudfront get-public-key --id K2JCJMDEHXQW5F
aws cloudfront list-key-groups \
  --query 'KeyGroupList.Items[].{Id:KeyGroup.Id,Name:KeyGroup.KeyGroupConfig.Name}' --output table
aws cloudfront get-key-group --id <key-group-id>

KG_ETAG=$(aws cloudfront get-key-group --id <id> --query ETag --output text)
aws cloudfront update-key-group --id <id> --if-match $KG_ETAG \
  --key-group-config '{"Name":"premium-content-keys","Items":["NEW-KEY","OLD-KEY"]}'
aws cloudfront delete-key-group --id <id> --if-match $KG_ETAG

PK_ETAG=$(aws cloudfront get-public-key --id <id> --query ETag --output text)
aws cloudfront delete-public-key --id <id> --if-match $PK_ETAG
```

### Sign a URL with the AWS CLI

```bash
aws cloudfront sign \
  --url "https://cdn.example.com/private/video.mp4" \
  --key-pair-id "K2JCJMDEHXQW5F" \
  --private-key file://private_key.pem \
  --date-less-than "2026-08-04T00:00:00Z"
```

### Custom policy signing (start time + IP restriction)

```bash
cat > policy.json <<'EOF'
{
  "Statement": [{
    "Resource": "https://cdn.example.com/premium/*",
    "Condition": {
      "DateLessThan":    { "AWS:EpochTime": 1785000000 },
      "DateGreaterThan": { "AWS:EpochTime": 1784900000 },
      "IpAddress":       { "AWS:SourceIp": "203.0.113.0/24" }
    }
  }]
}
EOF

# Base64-URL-encode the policy (CloudFront's variant: + → -, = → _, / → ~)
POLICY_B64=$(cat policy.json | tr -d ' \n' | openssl base64 -A | tr '+=/' '-_~')

# Sign it
SIG=$(cat policy.json | tr -d ' \n' \
  | openssl sha1 -sign private_key.pem \
  | openssl base64 -A | tr '+=/' '-_~')

echo "https://cdn.example.com/premium/video.mp4?Policy=${POLICY_B64}&Signature=${SIG}&Key-Pair-Id=K2JCJMDEHXQW5F"
```

### Rotate a signing key with zero downtime

```bash
# 1. Create the new public key
# 2. Add BOTH keys to the key group
aws cloudfront update-key-group --id <kg-id> --if-match $KG_ETAG \
  --key-group-config '{"Name":"premium-content-keys","Items":["NEW-KEY-ID","OLD-KEY-ID"]}'
# 3. Switch your signing service to the new private key
# 4. After all old signed URLs expire, remove the old key
aws cloudfront update-key-group --id <kg-id> --if-match $KG_ETAG2 \
  --key-group-config '{"Name":"premium-content-keys","Items":["NEW-KEY-ID"]}'
```

---

## 13. Certificates (ACM) & Custom Domains

> **`--region us-east-1` on every ACM command for CloudFront. No exceptions.**

```bash
# Request with DNS validation
CERT_ARN=$(aws acm request-certificate --region us-east-1 \
  --domain-name "example.com" \
  --subject-alternative-names "*.example.com" \
  --validation-method DNS \
  --query CertificateArn --output text)
echo "$CERT_ARN"

# Get the validation CNAME records
aws acm describe-certificate --region us-east-1 --certificate-arn "$CERT_ARN" \
  --query 'Certificate.DomainValidationOptions[].{Domain:DomainName,Name:ResourceRecord.Name,Value:ResourceRecord.Value}' \
  --output table

# Poll status
aws acm describe-certificate --region us-east-1 --certificate-arn "$CERT_ARN" \
  --query 'Certificate.Status' --output text        # PENDING_VALIDATION → ISSUED

# Block until issued
aws acm wait certificate-validated --region us-east-1 --certificate-arn "$CERT_ARN"

# List all CloudFront-eligible certs
aws acm list-certificates --region us-east-1 \
  --query 'CertificateSummaryList[].{Domain:DomainName,Arn:CertificateArn,Status:Status}' \
  --output table
```

### Attach the cert + aliases to the distribution

```bash
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)

jq --arg cert "$CERT_ARN" '.DistributionConfig
   | .Aliases = { "Quantity": 2, "Items": ["example.com", "www.example.com"] }
   | .ViewerCertificate = {
       "ACMCertificateArn": $cert,
       "SSLSupportMethod": "sni-only",
       "MinimumProtocolVersion": "TLSv1.2_2021",
       "Certificate": $cert,
       "CertificateSource": "acm"
     }' /tmp/full.json > /tmp/new.json

aws cloudfront update-distribution --id $DIST_ID --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json
```

**`SSLSupportMethod`:** `sni-only` (free, use this) | `vip` (dedicated IP, expensive) | `static-ip`

### CNAME conflict tools

```bash
# Who is already using this alias?
aws cloudfront list-conflicting-aliases --distribution-id $DIST_ID --alias "cdn.example.com"

# Move an alias between your own distributions with no downtime
aws cloudfront associate-alias --target-distribution-id E_TARGET --alias "cdn.example.com"
```

---

## 14. Route 53 DNS

```bash
HZ_ID=$(aws route53 list-hosted-zones-by-name --dns-name example.com \
  --query 'HostedZones[0].Id' --output text | sed 's|/hostedzone/||')

CF_DOMAIN=$(aws cloudfront get-distribution --id $DIST_ID \
  --query 'Distribution.DomainName' --output text)

cat > dns.json <<EOF
{
  "Comment": "Point cdn at CloudFront",
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "cdn.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "$CF_DOMAIN",
          "EvaluateTargetHealth": false
        }
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "cdn.example.com",
        "Type": "AAAA",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "$CF_DOMAIN",
          "EvaluateTargetHealth": false
        }
      }
    }
  ]
}
EOF

aws route53 change-resource-record-sets --hosted-zone-id "$HZ_ID" --change-batch file://dns.json
```

> **`Z2FDTNDATAQYW2`** is the fixed Route 53 hosted zone ID for *all* CloudFront distributions
> globally. It is not your zone ID and never changes.

```bash
# Verify propagation
aws route53 get-change --id /change/C1234567890ABC
dig +short cdn.example.com
dig +short AAAA cdn.example.com
```

**Non-Route-53 DNS:** create a plain `CNAME cdn.example.com → dxxxx.cloudfront.net`.
For an apex/root domain you need ALIAS/ANAME/CNAME-flattening support from your provider.

---

## 15. AWS WAF for CloudFront

> **Scope must be `CLOUDFRONT`, and the region must be `us-east-1`.**

```bash
# Create a web ACL with managed rules + rate limiting
cat > waf-rules.json <<'EOF'
[
  {
    "Name": "AWSManagedCommonRuleSet",
    "Priority": 0,
    "OverrideAction": { "Count": {} },
    "Statement": {
      "ManagedRuleGroupStatement": {
        "VendorName": "AWS", "Name": "AWSManagedRulesCommonRuleSet"
      }
    },
    "VisibilityConfig": {
      "SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true,
      "MetricName": "CommonRuleSet"
    }
  },
  {
    "Name": "AWSManagedKnownBadInputs",
    "Priority": 1,
    "OverrideAction": { "Count": {} },
    "Statement": {
      "ManagedRuleGroupStatement": {
        "VendorName": "AWS", "Name": "AWSManagedRulesKnownBadInputsRuleSet"
      }
    },
    "VisibilityConfig": {
      "SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true,
      "MetricName": "KnownBadInputs"
    }
  },
  {
    "Name": "RateLimitPerIP",
    "Priority": 2,
    "Action": { "Block": {} },
    "Statement": {
      "RateBasedStatement": { "Limit": 2000, "AggregateKeyType": "IP" }
    },
    "VisibilityConfig": {
      "SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true,
      "MetricName": "RateLimit"
    }
  }
]
EOF

aws wafv2 create-web-acl --region us-east-1 \
  --name cloudfront-protection \
  --scope CLOUDFRONT \
  --default-action Allow={} \
  --rules file://waf-rules.json \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=cfProtection
```

```bash
# Get the ARN and attach it
WAF_ARN=$(aws wafv2 list-web-acls --region us-east-1 --scope CLOUDFRONT \
  --query "WebACLs[?Name=='cloudfront-protection'].ARN" --output text)

cf_update $DIST_ID ".WebACLId = \"$WAF_ARN\""
```

```bash
# Inspect sampled blocked requests
aws wafv2 get-sampled-requests --region us-east-1 \
  --web-acl-arn "$WAF_ARN" --rule-metric-name RateLimit --scope CLOUDFRONT \
  --time-window StartTime=$(date -u -d '1 hour ago' +%s),EndTime=$(date -u +%s) \
  --max-items 20

# Switch a rule from Count to Block after review
aws wafv2 get-web-acl --region us-east-1 --scope CLOUDFRONT \
  --name cloudfront-protection --id <acl-id> > waf.json
# edit "OverrideAction": {"Count":{}} → {"None":{}}
aws wafv2 update-web-acl --region us-east-1 --scope CLOUDFRONT \
  --name cloudfront-protection --id <acl-id> --lock-token <token> \
  --default-action Allow={} --rules file://waf-rules-updated.json \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=cfProtection
```

---

## 16. Logging & Monitoring

### Standard logging (v2) — to S3

```bash
# Create the log delivery source
aws logs put-delivery-source --region us-east-1 \
  --name cf-logs-source-$DIST_ID \
  --resource-arn "arn:aws:cloudfront::$ACCOUNT_ID:distribution/$DIST_ID" \
  --log-type ACCESS_LOGS

# Create the destination
aws logs put-delivery-destination --region us-east-1 \
  --name cf-logs-s3-dest \
  --delivery-destination-configuration "destinationResourceArn=arn:aws:s3:::my-cf-logs" \
  --output-format plain      # plain | w3c | raw | json | parquet

# Link them
aws logs create-delivery --region us-east-1 \
  --delivery-source-name cf-logs-source-$DIST_ID \
  --delivery-destination-arn "arn:aws:logs:us-east-1:$ACCOUNT_ID:delivery-destination:cf-logs-s3-dest"

# Inspect
aws logs describe-deliveries --region us-east-1
aws logs describe-delivery-sources --region us-east-1
aws logs describe-delivery-destinations --region us-east-1
```

### Standard logging (legacy v1) — set directly on the distribution

```bash
jq '.Logging = {
      "Enabled": true,
      "IncludeCookies": false,
      "Bucket": "my-cf-logs.s3.amazonaws.com",
      "Prefix": "cloudfront/"
    }' /tmp/cfg.json > /tmp/new.json
```

The log bucket must have ACLs enabled (`BucketOwnerPreferred`) for v1 logging:

```bash
aws s3api put-bucket-ownership-controls --bucket my-cf-logs \
  --ownership-controls 'Rules=[{ObjectOwnership=BucketOwnerPreferred}]'
```

### Real-time logs → Kinesis Data Streams

```bash
aws kinesis create-stream --stream-name cf-realtime-logs --shard-count 1
STREAM_ARN=$(aws kinesis describe-stream --stream-name cf-realtime-logs \
  --query 'StreamDescription.StreamARN' --output text)

# IAM role CloudFront assumes to write to Kinesis
cat > kinesis-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"cloudfront.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name cf-realtime-logs-role \
  --assume-role-policy-document file://kinesis-trust.json

aws cloudfront create-realtime-log-config \
  --name cf-realtime \
  --sampling-rate 10 \
  --end-points "StreamType=Kinesis,KinesisStreamConfig={RoleARN=arn:aws:iam::$ACCOUNT_ID:role/cf-realtime-logs-role,StreamARN=$STREAM_ARN}" \
  --fields timestamp c-ip sc-status cs-method cs-uri-stem cs-uri-query \
           x-edge-result-type x-edge-response-result-type time-taken \
           cs-user-agent c-country sc-content-len time-to-first-byte

aws cloudfront list-realtime-log-configs
aws cloudfront get-realtime-log-config --name cf-realtime
aws cloudfront update-realtime-log-config --name cf-realtime --sampling-rate 5
aws cloudfront delete-realtime-log-config --name cf-realtime
```

Attach it to a cache behavior:

```bash
RTL_ARN=$(aws cloudfront get-realtime-log-config --name cf-realtime \
  --query 'RealtimeLogConfig.ARN' --output text)
cf_update $DIST_ID ".DefaultCacheBehavior.RealtimeLogConfigArn = \"$RTL_ARN\""
```

### CloudWatch metrics

```bash
# Enable additional (paid) metrics: CacheHitRate, OriginLatency, per-status error rates
aws cloudfront create-monitoring-subscription --distribution-id $DIST_ID \
  --monitoring-subscription 'RealtimeMetricsSubscriptionConfig={RealtimeMetricsSubscriptionStatus=Enabled}'

aws cloudfront get-monitoring-subscription --distribution-id $DIST_ID
aws cloudfront delete-monitoring-subscription --distribution-id $DIST_ID
```

```bash
# Query metrics — ALWAYS region us-east-1, namespace AWS/CloudFront
aws cloudwatch get-metric-statistics --region us-east-1 \
  --namespace AWS/CloudFront --metric-name Requests \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 3600 --statistics Sum

# Cache hit rate
aws cloudwatch get-metric-statistics --region us-east-1 \
  --namespace AWS/CloudFront --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 3600 --statistics Average

# 5xx error rate
aws cloudwatch get-metric-statistics --region us-east-1 \
  --namespace AWS/CloudFront --metric-name 5xxErrorRate \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --start-time $(date -u -d '3 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average
```

### Alarms

```bash
aws cloudwatch put-metric-alarm --region us-east-1 \
  --alarm-name "CF-5xx-High-$DIST_ID" \
  --alarm-description "CloudFront 5xx error rate above 1%" \
  --namespace AWS/CloudFront --metric-name 5xxErrorRate \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --statistic Average --period 300 --evaluation-periods 2 \
  --threshold 1 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:$ACCOUNT_ID:ops-alerts

aws cloudwatch put-metric-alarm --region us-east-1 \
  --alarm-name "CF-CacheHitRate-Low-$DIST_ID" \
  --namespace AWS/CloudFront --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --statistic Average --period 3600 --evaluation-periods 3 \
  --threshold 80 --comparison-operator LessThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:$ACCOUNT_ID:ops-alerts
```

### CloudTrail — who changed the distribution?

```bash
aws cloudtrail lookup-events --region us-east-1 \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudfront.amazonaws.com \
  --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query 'Events[].{Time:EventTime,User:Username,Event:EventName}' --output table

aws cloudtrail lookup-events --region us-east-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateDistribution \
  --max-results 10
```

---

## 17. Origin Shield, Price Class & Compression

```bash
# Enable Origin Shield in the region CLOSEST TO YOUR ORIGIN
jq '.Origins.Items[0].OriginShield = {
      "Enabled": true,
      "OriginShieldRegion": "ap-south-1"
    }' /tmp/cfg.json > /tmp/new.json

# Price class
cf_update $DIST_ID '.PriceClass = "PriceClass_200"'
# PriceClass_All | PriceClass_200 | PriceClass_100

# Compression on every behavior
jq '.DefaultCacheBehavior.Compress = true
    | .CacheBehaviors.Items = [ .CacheBehaviors.Items[]? | .Compress = true ]' \
  /tmp/cfg.json > /tmp/new.json

# HTTP/3 + IPv6
cf_update $DIST_ID '.HttpVersion = "http2and3" | .IsIPV6Enabled = true'
# HttpVersion: http1.1 | http2 | http3 | http2and3

# Geo restriction — block a list of countries
jq '.Restrictions.GeoRestriction = {
      "RestrictionType": "blacklist",
      "Quantity": 2,
      "Items": ["CU", "KP"]
    }' /tmp/cfg.json > /tmp/new.json
# RestrictionType: none | whitelist | blacklist
```

---

## 18. Continuous Deployment

```bash
# 1. Create a staging distribution cloned from the primary
aws cloudfront copy-distribution \
  --primary-distribution-id $DIST_ID \
  --staging \
  --caller-reference "staging-$(date +%s)"
# → returns STAGING_DIST_ID

# 2. Change whatever you're testing on the staging distribution
cf_update $STAGING_DIST_ID '.DefaultCacheBehavior.CachePolicyId = "NEW-POLICY-ID"'

# 3a. Weight-based traffic split (max 15%)
aws cloudfront create-continuous-deployment-policy \
  --continuous-deployment-policy-config "{
    \"StagingDistributionDnsNames\": { \"Quantity\": 1, \"Items\": [\"$STAGING_DOMAIN\"] },
    \"Enabled\": true,
    \"TrafficConfig\": {
      \"Type\": \"SingleWeight\",
      \"SingleWeightConfig\": {
        \"Weight\": 0.05,
        \"SessionStickinessConfig\": { \"IdleTTL\": 300, \"MaximumTTL\": 600 }
      }
    }
  }"

# 3b. OR header-based targeting (for your own QA traffic)
aws cloudfront create-continuous-deployment-policy \
  --continuous-deployment-policy-config "{
    \"StagingDistributionDnsNames\": { \"Quantity\": 1, \"Items\": [\"$STAGING_DOMAIN\"] },
    \"Enabled\": true,
    \"TrafficConfig\": {
      \"Type\": \"SingleHeader\",
      \"SingleHeaderConfig\": { \"Header\": \"aws-cf-cd-canary\", \"Value\": \"true\" }
    }
  }"

# 4. Attach the policy to the PRIMARY distribution
cf_update $DIST_ID '.ContinuousDeploymentPolicyId = "CDP-ID"'

# Test the staging path explicitly
curl -H "aws-cf-cd-canary: true" -I https://cdn.example.com/

# 5. Promote staging config onto the primary
aws cloudfront update-distribution-with-staging-config \
  --id $DIST_ID --staging-distribution-id $STAGING_DIST_ID --if-match $ETAG

# Management
aws cloudfront list-continuous-deployment-policies
aws cloudfront get-continuous-deployment-policy --id CDP-ID
aws cloudfront update-continuous-deployment-policy --id CDP-ID --if-match $ETAG \
  --continuous-deployment-policy-config file://cdp.json
aws cloudfront delete-continuous-deployment-policy --id CDP-ID --if-match $ETAG
```

---

## 19. Field-Level Encryption

```bash
# 1. Upload a public key (same process as signed URLs)
aws cloudfront create-public-key --public-key-config "{
  \"CallerReference\": \"fle-$(date +%s)\",
  \"Name\": \"fle-key\",
  \"EncodedKey\": \"$(cat fle_public.pem)\"
}"

# 2. Create a profile mapping fields → key
aws cloudfront create-field-level-encryption-profile \
  --field-level-encryption-profile-config '{
    "Name": "payment-profile",
    "CallerReference": "fle-profile-1",
    "EncryptionEntities": {
      "Quantity": 1,
      "Items": [{
        "PublicKeyId": "K1ABCDEFGHIJKL",
        "ProviderId": "payment-service",
        "FieldPatterns": { "Quantity": 2, "Items": ["card_number", "cvv"] }
      }]
    }
  }'

# 3. Create a configuration mapping content types → profile
aws cloudfront create-field-level-encryption-config \
  --field-level-encryption-config '{
    "CallerReference": "fle-config-1",
    "Comment": "Encrypt card fields",
    "ContentTypeProfileConfig": {
      "ForwardWhenContentTypeIsUnknown": false,
      "ContentTypeProfiles": {
        "Quantity": 1,
        "Items": [{
          "Format": "URLEncoded",
          "ProfileId": "FLE-PROFILE-ID",
          "ContentType": "application/x-www-form-urlencoded"
        }]
      }
    }
  }'

# 4. Attach to a cache behavior
cf_update $DIST_ID '.DefaultCacheBehavior.FieldLevelEncryptionId = "FLE-CONFIG-ID"'

# Management
aws cloudfront list-field-level-encryption-configs
aws cloudfront list-field-level-encryption-profiles
aws cloudfront get-field-level-encryption --id <id>
aws cloudfront delete-field-level-encryption-config --id <id> --if-match $ETAG
aws cloudfront delete-field-level-encryption-profile --id <id> --if-match $ETAG
```

---

## 20. Anycast Static IPs & Distribution Tenants

```bash
# Anycast static IP lists (ENTERPRISE PRICING — do not create casually)
aws cloudfront create-anycast-ip-list --name partner-allowlist --ip-count 21
aws cloudfront list-anycast-ip-lists
aws cloudfront get-anycast-ip-list --id <id>
aws cloudfront delete-anycast-ip-list --id <id> --if-match $ETAG

# Attach to a distribution
cf_update $DIST_ID '.AnycastIpListId = "<anycast-list-id>"'
```

```bash
# Multi-tenant: distribution templates and tenants (SaaS Manager)
aws cloudfront create-distribution-tenant \
  --distribution-id $TEMPLATE_DIST_ID \
  --name "tenant-acme" \
  --domains '[{"Domain":"cdn.acme.com"}]' \
  --parameters '[{"Name":"originPath","Value":"/tenants/acme"}]' \
  --enabled

aws cloudfront list-distribution-tenants
aws cloudfront get-distribution-tenant --identifier <tenant-id>
aws cloudfront update-distribution-tenant --id <tenant-id> --if-match $ETAG --enabled
aws cloudfront delete-distribution-tenant --id <tenant-id> --if-match $ETAG

# Connection groups & domain verification
aws cloudfront list-connection-groups
aws cloudfront verify-dns-configuration --domain cdn.acme.com --identifier <tenant-id>
```

---

## 21. S3 Commands You'll Use Alongside CloudFront

```bash
# Create a private bucket (Block Public Access is on by default — leave it)
aws s3api create-bucket --bucket "$BUCKET" --region "$AWS_REGION" \
  --create-bucket-configuration LocationConstraint="$AWS_REGION"

aws s3api put-public-access-block --bucket "$BUCKET" \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

aws s3api put-bucket-versioning --bucket "$BUCKET" \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption --bucket "$BUCKET" \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

### Upload with the RIGHT Cache-Control and Content-Type

```bash
# Long-lived hashed assets
aws s3 sync ./dist/assets "s3://$BUCKET/assets" \
  --cache-control "public, max-age=31536000, immutable" \
  --metadata-directive REPLACE

# Short-lived entry point
aws s3 cp ./dist/index.html "s3://$BUCKET/index.html" \
  --cache-control "no-cache" \
  --content-type "text/html; charset=utf-8"

# Per-type content types (fixes the "compression not working" bug)
aws s3 cp ./dist "s3://$BUCKET" --recursive \
  --exclude "*" --include "*.js"  --content-type "application/javascript"
aws s3 cp ./dist "s3://$BUCKET" --recursive \
  --exclude "*" --include "*.css" --content-type "text/css"
aws s3 cp ./dist "s3://$BUCKET" --recursive \
  --exclude "*" --include "*.svg" --content-type "image/svg+xml"
aws s3 cp ./dist "s3://$BUCKET" --recursive \
  --exclude "*" --include "*.json" --content-type "application/json"

# Verify what actually got set
aws s3api head-object --bucket "$BUCKET" --key "assets/app.a3f9c1.js" \
  --query '{CT:ContentType,CC:CacheControl,Size:ContentLength}'
```

### CORS on the bucket (needed for fonts and XHR to S3-origin assets)

```bash
aws s3api put-bucket-cors --bucket "$BUCKET" --cors-configuration '{
  "CORSRules": [{
    "AllowedOrigins": ["https://app.example.com"],
    "AllowedMethods": ["GET", "HEAD"],
    "AllowedHeaders": ["*"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }]
}'
```

---

## 22. Tagging

```bash
DIST_ARN="arn:aws:cloudfront::$ACCOUNT_ID:distribution/$DIST_ID"

aws cloudfront tag-resource --resource "$DIST_ARN" --tags 'Items=[
  {Key=Environment,Value=production},
  {Key=Owner,Value=platform-team},
  {Key=CostCenter,Value=CC-1042},
  {Key=ManagedBy,Value=terraform}
]'

aws cloudfront list-tags-for-resource --resource "$DIST_ARN"
aws cloudfront untag-resource --resource "$DIST_ARN" --tag-keys 'Items=[CostCenter]'
```

---

## 23. Quotas & Service Limits

```bash
# List all CloudFront quotas and current values
aws service-quotas list-service-quotas --service-code cloudfront \
  --query 'Quotas[].{Name:QuotaName,Value:Value,Adjustable:Adjustable}' --output table

# Look up a specific quota
aws service-quotas list-service-quotas --service-code cloudfront \
  --query "Quotas[?contains(QuotaName,'Cache behaviors')]" --output table

# Request an increase
aws service-quotas request-service-quota-increase \
  --service-code cloudfront --quota-code L-24B04930 --desired-value 400

aws service-quotas list-requested-service-quota-change-history --service-code cloudfront
```

---

## 24. Testing & Debugging with curl / dig / openssl

### The essential curl checks

```bash
# Full headers
curl -sSI https://cdn.example.com/

# Cache status + which PoP + request id (the three you always want)
curl -sSI https://cdn.example.com/ | grep -iE 'x-cache|x-amz-cf-pop|x-amz-cf-id|age'

# Hit ratio spot-check: run twice, second should be a Hit
for i in 1 2; do curl -sSI https://cdn.example.com/ | grep -i x-cache; done

# Verify HTTP → HTTPS redirect
curl -sSI http://cdn.example.com/ | head -3

# Compression
curl -sSI -H 'Accept-Encoding: gzip, br' https://cdn.example.com/app.js \
  | grep -iE 'content-encoding|content-type|content-length'

# Full timing breakdown
curl -sS -o /dev/null -w '
  dns:       %{time_namelookup}s
  connect:   %{time_connect}s
  tls:       %{time_appconnect}s
  ttfb:      %{time_starttransfer}s
  total:     %{time_total}s
  size:      %{size_download} bytes
  http:      %{http_code}
' https://cdn.example.com/

# Force HTTP/2 and HTTP/3
curl -sSI --http2 https://cdn.example.com/
curl -sSI --http3 https://cdn.example.com/          # requires curl built with HTTP/3

# Test a specific cache behavior
curl -sSI https://cdn.example.com/api/health
curl -sSI https://cdn.example.com/static/logo.png

# Test with an Origin header (CORS preflight)
curl -sSI -X OPTIONS https://cdn.example.com/api/data \
  -H 'Origin: https://app.example.com' \
  -H 'Access-Control-Request-Method: POST'

# Range request
curl -sSI -H 'Range: bytes=0-1023' https://cdn.example.com/video.mp4

# Signed URL test
curl -sS -o /dev/null -w '%{http_code}\n' "$SIGNED_URL"

# Verify the origin is NOT directly reachable
curl -sSI "https://$BUCKET.s3.$AWS_REGION.amazonaws.com/index.html"   # expect 403

# Test a specific edge by resolving to a known IP (advanced)
curl -sSI --resolve cdn.example.com:443:13.x.x.x https://cdn.example.com/

# Bypass CloudFront cache for a one-off check
curl -sSI -H 'Cache-Control: no-cache' https://cdn.example.com/
```

### DNS

```bash
dig +short cdn.example.com
dig +short AAAA cdn.example.com
dig cdn.example.com CNAME +short
dig @8.8.8.8 cdn.example.com          # check from a public resolver
nslookup cdn.example.com
```

### TLS inspection

```bash
# Full handshake detail, certificate chain, negotiated protocol
openssl s_client -connect cdn.example.com:443 -servername cdn.example.com </dev/null

# Certificate dates and SANs
echo | openssl s_client -connect cdn.example.com:443 -servername cdn.example.com 2>/dev/null \
  | openssl x509 -noout -dates -subject -ext subjectAltName

# Verify TLS 1.2 works and TLS 1.0 is rejected
openssl s_client -connect cdn.example.com:443 -tls1_2 </dev/null 2>&1 | grep -i protocol
openssl s_client -connect cdn.example.com:443 -tls1   </dev/null 2>&1 | grep -i 'alert\|error'

# Check the ORIGIN's certificate (the usual cause of 502s)
echo | openssl s_client -connect origin.example.com:443 -servername origin.example.com 2>/dev/null \
  | openssl x509 -noout -dates -issuer -subject
```

### CloudFront IP ranges

```bash
# Fetch the current CloudFront edge IP ranges
curl -s https://ip-ranges.amazonaws.com/ip-ranges.json \
  | jq -r '.prefixes[] | select(.service=="CLOUDFRONT") | .ip_prefix' | head -20

# Origin-facing ranges only (what your firewall should allow)
curl -s https://ip-ranges.amazonaws.com/ip-ranges.json \
  | jq -r '.prefixes[] | select(.service=="CLOUDFRONT_ORIGIN_FACING") | .ip_prefix'

# Better: use the managed prefix list in security groups
aws ec2 describe-managed-prefix-lists --region $AWS_REGION \
  --filters "Name=prefix-list-name,Values=com.amazonaws.global.cloudfront.origin-facing" \
  --query 'PrefixLists[0].PrefixListId' --output text
```

---

## 25. One-Liners & Recipes

```bash
# Every distribution with its cache hit rate over 24h
for D in $(aws cloudfront list-distributions --query 'DistributionList.Items[].Id' --output text); do
  R=$(aws cloudwatch get-metric-statistics --region us-east-1 \
    --namespace AWS/CloudFront --metric-name CacheHitRate \
    --dimensions Name=DistributionId,Value=$D Name=Region,Value=Global \
    --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 86400 --statistics Average \
    --query 'Datapoints[0].Average' --output text 2>/dev/null)
  echo "$D  ${R:-n/a}"
done
```

```bash
# Find all distributions with NO WAF attached
aws cloudfront list-distributions \
  --query "DistributionList.Items[?WebACLId==''].{Id:Id,Domain:DomainName,Aliases:Aliases.Items}" \
  --output table
```

```bash
# Find all distributions still using the legacy OAI
aws cloudfront list-distributions --query \
  "DistributionList.Items[?Origins.Items[?S3OriginConfig.OriginAccessIdentity!='']].Id" --output text
```

```bash
# Find distributions with a weak minimum TLS version
aws cloudfront list-distributions --query \
  "DistributionList.Items[?ViewerCertificate.MinimumProtocolVersion!='TLSv1.2_2021'].{Id:Id,TLS:ViewerCertificate.MinimumProtocolVersion}" \
  --output table
```

```bash
# Find distributions where compression is off
aws cloudfront list-distributions --query \
  "DistributionList.Items[?DefaultCacheBehavior.Compress==\`false\`].{Id:Id,Domain:DomainName}" \
  --output table
```

```bash
# Back up every distribution config to disk
mkdir -p cf-backups
for D in $(aws cloudfront list-distributions --query 'DistributionList.Items[].Id' --output text); do
  aws cloudfront get-distribution-config --id "$D" > "cf-backups/${D}.json"
  echo "saved $D"
done
```

```bash
# Total invalidation paths used this month
aws cloudfront list-invalidations --distribution-id $DIST_ID \
  --query 'InvalidationList.Items[].Id' --output text | tr '\t' '\n' | while read I; do
    aws cloudfront get-invalidation --distribution-id $DIST_ID --id "$I" \
      --query 'Invalidation.InvalidationBatch.Paths.Quantity' --output text
  done | awk '{s+=$1} END {print "paths this listing:", s}'
```

```bash
# Full teardown of a demo distribution
teardown() {
  local D=$1
  aws cloudfront get-distribution-config --id "$D" > /tmp/t.json
  local E=$(jq -r '.ETag' /tmp/t.json)
  jq '.DistributionConfig | .Enabled=false' /tmp/t.json > /tmp/t2.json
  aws cloudfront update-distribution --id "$D" --if-match "$E" \
    --distribution-config file:///tmp/t2.json >/dev/null
  echo "disabled — waiting for deploy..."
  aws cloudfront wait distribution-deployed --id "$D"
  local E2=$(aws cloudfront get-distribution-config --id "$D" --query ETag --output text)
  aws cloudfront delete-distribution --id "$D" --if-match "$E2"
  echo "deleted $D"
}
```

---

## 26. Terraform Quick Reference

```hcl
resource "aws_cloudfront_origin_access_control" "s3" {
  name                              = "s3-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_distribution" "site" {
  enabled             = true
  is_ipv6_enabled     = true
  http_version        = "http2and3"
  default_root_object = "index.html"
  price_class         = "PriceClass_All"
  aliases             = ["cdn.example.com"]
  comment             = "Static site"
  web_acl_id          = aws_wafv2_web_acl.cf.arn

  origin {
    domain_name              = aws_s3_bucket.site.bucket_regional_domain_name
    origin_id                = "s3-site"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }

  default_cache_behavior {
    target_origin_id           = "s3-site"
    viewer_protocol_policy     = "redirect-to-https"
    allowed_methods            = ["GET", "HEAD", "OPTIONS"]
    cached_methods             = ["GET", "HEAD"]
    compress                   = true
    cache_policy_id            = data.aws_cloudfront_cache_policy.optimized.id
    response_headers_policy_id = data.aws_cloudfront_response_headers_policy.security.id

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.rewrite.arn
    }
  }

  ordered_cache_behavior {
    path_pattern           = "/api/*"
    target_origin_id       = "alb-api"
    viewer_protocol_policy = "https-only"
    allowed_methods        = ["GET","HEAD","OPTIONS","PUT","POST","PATCH","DELETE"]
    cached_methods         = ["GET","HEAD"]
    cache_policy_id           = data.aws_cloudfront_cache_policy.disabled.id
    origin_request_policy_id  = data.aws_cloudfront_origin_request_policy.all_no_host.id
  }

  custom_error_response {
    error_code            = 403
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 10
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cert.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }

  tags = { Environment = "prod", Owner = "platform" }
}

data "aws_cloudfront_cache_policy" "optimized" { name = "Managed-CachingOptimized" }
data "aws_cloudfront_cache_policy" "disabled"  { name = "Managed-CachingDisabled" }
data "aws_cloudfront_origin_request_policy" "all_no_host" {
  name = "Managed-AllViewerExceptHostHeader"
}
data "aws_cloudfront_response_headers_policy" "security" {
  name = "Managed-SecurityHeadersPolicy"
}

resource "aws_cloudfront_function" "rewrite" {
  name    = "uri-rewrite"
  runtime = "cloudfront-js-2.0"
  publish = true
  code    = file("${path.module}/uri-rewrite.js")
}
```

**The provider alias you'll need for us-east-1 resources:**

```hcl
provider "aws" { region = "ap-south-1" }
provider "aws" { alias = "us_east_1", region = "us-east-1" }

resource "aws_acm_certificate" "cert" {
  provider          = aws.us_east_1
  domain_name       = "cdn.example.com"
  validation_method = "DNS"
}
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────┐
│  THINGS THAT MUST BE IN us-east-1                                       │
│    ACM certificates for CloudFront                                      │
│    Lambda@Edge functions                                                │
│    WAF web ACLs with scope CLOUDFRONT                                   │
│    CloudWatch metrics for AWS/CloudFront                                │
│    CloudFront Functions logs                                            │
├─────────────────────────────────────────────────────────────────────────┤
│  THINGS THAT NEED --if-match <ETag>                                     │
│    update-distribution, delete-distribution                             │
│    update/delete of every policy, OAC, key group, public key, function  │
├─────────────────────────────────────────────────────────────────────────┤
│  FIXED VALUES WORTH MEMORIZING                                          │
│    Route 53 hosted zone for all CloudFront distributions: Z2FDTNDATAQYW2│
│    Managed prefix list: com.amazonaws.global.cloudfront.origin-facing   │
├─────────────────────────────────────────────────────────────────────────┤
│  THE THREE DEBUG HEADERS                                                │
│    X-Cache  •  X-Amz-Cf-Pop  •  X-Amz-Cf-Id                             │
└─────────────────────────────────────────────────────────────────────────┘
```
