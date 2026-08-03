# Amazon CloudFront — Hands-On Labs

> Fifteen labs that take you from an empty AWS account to a production-grade, private-origin,
> edge-compute-enabled, monitored CloudFront deployment. Every lab has an objective, prerequisites,
> full commands, a verification step, and a "what you just learned" summary.
>
> **Do them in order.** Later labs build on earlier ones.
> **Lab 15 is the teardown.** Run it when you're done or you will pay for idle resources.

---

## Lab Index

| # | Lab | Time | Cost |
|---|---|---|---|
| 0 | [Environment Setup](#lab-0--environment-setup) | 10 min | free |
| 1 | [First Distribution: S3 + OAC](#lab-1--first-distribution-s3--oac) | 25 min | free tier |
| 2 | [Custom Domain, ACM & Route 53](#lab-2--custom-domain-acm--route-53) | 30 min | domain cost only |
| 3 | [Cache Policies, TTLs & Proving the Cache Works](#lab-3--cache-policies-ttls--proving-the-cache-works) | 30 min | free tier |
| 4 | [Multiple Behaviors & a Second Origin](#lab-4--multiple-behaviors--a-second-origin) | 30 min | ~$0.03/hr for ALB |
| 5 | [Invalidations vs Versioned Assets](#lab-5--invalidations-vs-versioned-assets) | 20 min | free tier |
| 6 | [Origin Groups & Failover](#lab-6--origin-groups--failover) | 25 min | free tier |
| 7 | [CloudFront Functions + KeyValueStore](#lab-7--cloudfront-functions--keyvaluestore) | 35 min | free tier |
| 8 | [Lambda@Edge](#lab-8--lambdaedge) | 40 min | pennies |
| 9 | [Signed URLs & Signed Cookies](#lab-9--signed-urls--signed-cookies) | 35 min | free tier |
| 10 | [Security: WAF, Geo, Response Headers](#lab-10--security-waf-geo-restriction--response-headers) | 35 min | ~$5/mo WAF |
| 11 | [Logging: Standard Logs + Athena + Real-Time](#lab-11--logging-standard-logs-athena--real-time) | 40 min | pennies |
| 12 | [Origin Shield, Compression & Price Class](#lab-12--origin-shield-compression--price-class) | 20 min | pennies |
| 13 | [Continuous Deployment (Staging Distribution)](#lab-13--continuous-deployment-staging-distribution) | 30 min | free tier |
| 14 | [Everything as Code with Terraform](#lab-14--everything-as-code-with-terraform) | 45 min | free tier |
| 15 | [Complete Teardown](#lab-15--complete-teardown) | 15 min | saves money |

---

## Lab 0 — Environment Setup

**Objective:** Get a working shell with the right tools and variables so every later lab is
copy-paste.

### Steps

```bash
# 1. Verify the CLI
aws --version                 # need aws-cli/2.x
aws sts get-caller-identity   # should print your account, user/role ARN

# 2. Install jq if you don't have it
jq --version || sudo apt install -y jq

# 3. Set up a working directory and environment file
mkdir -p ~/cloudfront-labs && cd ~/cloudfront-labs

cat > env.sh <<'EOF'
export AWS_REGION="ap-south-1"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET="cf-labs-${ACCOUNT_ID}-$(echo $RANDOM | md5sum | head -c6)"
export LOG_BUCKET="cf-labs-logs-${ACCOUNT_ID}"
EOF

source env.sh
echo "Region: $AWS_REGION | Account: $ACCOUNT_ID | Bucket: $BUCKET"

# 4. Save BUCKET permanently so later labs reuse the same one
sed -i "s|^export BUCKET=.*|export BUCKET=\"$BUCKET\"|" env.sh
cat env.sh
```

### Reusable helper functions

```bash
cat >> env.sh <<'EOF'

# Fetch config + ETag, apply a jq filter, push it back
cf_update() {
  local D="$1" F="$2"
  aws cloudfront get-distribution-config --id "$D" > /tmp/cf-full.json
  local E=$(jq -r '.ETag' /tmp/cf-full.json)
  jq '.DistributionConfig' /tmp/cf-full.json > /tmp/cf-cfg.json
  jq "$F" /tmp/cf-cfg.json > /tmp/cf-new.json
  aws cloudfront update-distribution --id "$D" --if-match "$E" \
    --distribution-config file:///tmp/cf-new.json --query 'Distribution.Status' --output text
}

# Show the three debug headers
cfcheck() { curl -sSI "$1" | grep -iE 'HTTP/|x-cache|age|x-amz-cf-pop|content-encoding|content-type'; }
EOF

source env.sh
```

### Verify

```bash
type cf_update && type cfcheck && echo "helpers loaded ✔"
```

**What you learned:** CloudFront work is 20% CloudFront and 80% shell plumbing. Setting up
`cf_update` now saves you a hundred ETag mistakes later.

---

## Lab 1 — First Distribution: S3 + OAC

**Objective:** Serve a static site from a **completely private** S3 bucket through CloudFront using
Origin Access Control. This is the single most common CloudFront pattern in the world.

**Architecture:**

```
   Browser ──► CloudFront (dxxxx.cloudfront.net) ──[OAC / SigV4]──► S3 (PRIVATE)
                                                                      │
   Browser ──X── direct bucket URL ───────────────────────────────► 403 ✔
```

### Step 1 — Create the private bucket

```bash
source ~/cloudfront-labs/env.sh

aws s3api create-bucket --bucket "$BUCKET" --region "$AWS_REGION" \
  --create-bucket-configuration LocationConstraint="$AWS_REGION"

# Block Public Access is on by default — confirm it
aws s3api get-public-access-block --bucket "$BUCKET"

aws s3api put-bucket-versioning --bucket "$BUCKET" \
  --versioning-configuration Status=Enabled
```

### Step 2 — Build a tiny site

```bash
mkdir -p site/assets

cat > site/index.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>CloudFront Lab</title>
  <link rel="stylesheet" href="/assets/style.css">
</head>
<body>
  <main>
    <h1>Served through CloudFront</h1>
    <p id="ts">Build: BUILD_TIME</p>
    <p><a href="/about/">About page (tests directory routing)</a></p>
    <p><a href="/assets/data.json">JSON asset</a></p>
  </main>
  <script src="/assets/app.js"></script>
</body>
</html>
EOF

sed -i "s/BUILD_TIME/$(date -u +%Y-%m-%dT%H:%M:%SZ)/" site/index.html

cat > site/assets/style.css <<'EOF'
:root { color-scheme: light dark; }
body { font-family: system-ui, -apple-system, sans-serif; margin: 0;
       display: grid; place-items: center; min-height: 100vh; }
main { max-width: 42rem; padding: 2rem; line-height: 1.7; }
h1 { font-weight: 650; letter-spacing: -0.02em; }
a { color: #0b62d0; }
EOF

cat > site/assets/app.js <<'EOF'
console.log('CloudFront lab loaded at', new Date().toISOString());
document.addEventListener('DOMContentLoaded', () => {
  fetch('/assets/data.json').then(r => r.json()).then(d => console.log('data:', d));
});
EOF

echo '{"lab":1,"message":"hello from S3 via CloudFront"}' > site/assets/data.json

mkdir -p site/about
cat > site/about/index.html <<'EOF'
<!DOCTYPE html><html><head><title>About</title></head>
<body><h1>About</h1><p>If you can see this, URI rewriting works.</p></body></html>
EOF
```

### Step 3 — Upload with correct Content-Type and Cache-Control

```bash
# Long-lived assets
aws s3 sync ./site/assets "s3://$BUCKET/assets" \
  --cache-control "public, max-age=31536000, immutable"

# HTML — short TTL
aws s3 cp ./site/index.html "s3://$BUCKET/index.html" \
  --cache-control "no-cache" --content-type "text/html; charset=utf-8"
aws s3 cp ./site/about/index.html "s3://$BUCKET/about/index.html" \
  --cache-control "no-cache" --content-type "text/html; charset=utf-8"

# Verify the metadata actually landed
aws s3api head-object --bucket "$BUCKET" --key "assets/style.css" \
  --query '{ContentType:ContentType,CacheControl:CacheControl}'
```

### Step 4 — Create the Origin Access Control

```bash
OAC_ID=$(aws cloudfront create-origin-access-control --origin-access-control-config "{
  \"Name\": \"oac-$BUCKET\",
  \"Description\": \"OAC for CloudFront lab bucket\",
  \"SigningProtocol\": \"sigv4\",
  \"SigningBehavior\": \"always\",
  \"OriginAccessControlOriginType\": \"s3\"
}" --query 'OriginAccessControl.Id' --output text)

echo "OAC_ID=$OAC_ID" >> ~/cloudfront-labs/env.sh
echo "OAC: $OAC_ID"
```

### Step 5 — Create the distribution

```bash
source ~/cloudfront-labs/env.sh

cat > /tmp/dist.json <<EOF
{
  "CallerReference": "cf-lab-$(date +%s)",
  "Comment": "Lab 1 — S3 + OAC static site",
  "Enabled": true,
  "DefaultRootObject": "index.html",
  "PriceClass": "PriceClass_All",
  "HttpVersion": "http2and3",
  "IsIPV6Enabled": true,
  "Aliases": { "Quantity": 0 },
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "s3-origin",
      "DomainName": "$BUCKET.s3.$AWS_REGION.amazonaws.com",
      "OriginPath": "",
      "OriginAccessControlId": "$OAC_ID",
      "S3OriginConfig": { "OriginAccessIdentity": "" },
      "CustomHeaders": { "Quantity": 0 },
      "ConnectionAttempts": 3,
      "ConnectionTimeout": 10,
      "OriginShield": { "Enabled": false }
    }]
  },
  "OriginGroups": { "Quantity": 0 },
  "DefaultCacheBehavior": {
    "TargetOriginId": "s3-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 2, "Items": ["GET","HEAD"],
      "CachedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] }
    },
    "Compress": true,
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "SmoothStreaming": false,
    "FieldLevelEncryptionId": "",
    "FunctionAssociations": { "Quantity": 0 },
    "LambdaFunctionAssociations": { "Quantity": 0 },
    "TrustedKeyGroups": { "Enabled": false, "Quantity": 0 }
  },
  "CacheBehaviors": { "Quantity": 0 },
  "CustomErrorResponses": { "Quantity": 0 },
  "ViewerCertificate": {
    "CloudFrontDefaultCertificate": true,
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "Restrictions": { "GeoRestriction": { "RestrictionType": "none", "Quantity": 0 } },
  "Logging": { "Enabled": false, "IncludeCookies": false, "Bucket": "", "Prefix": "" },
  "WebACLId": ""
}
EOF
```

> If `create-distribution` rejects the `CachePolicyId`, look up the real ID for your account:
> `aws cloudfront list-cache-policies --type managed --query "CachePolicyList.Items[?CachePolicy.CachePolicyConfig.Name=='Managed-CachingOptimized'].CachePolicy.Id" --output text`

```bash
DIST_JSON=$(aws cloudfront create-distribution --distribution-config file:///tmp/dist.json)
DIST_ID=$(echo "$DIST_JSON"  | jq -r '.Distribution.Id')
CF_DOMAIN=$(echo "$DIST_JSON"| jq -r '.Distribution.DomainName')

echo "export DIST_ID=\"$DIST_ID\""     >> ~/cloudfront-labs/env.sh
echo "export CF_DOMAIN=\"$CF_DOMAIN\"" >> ~/cloudfront-labs/env.sh
source ~/cloudfront-labs/env.sh

echo "Distribution: $DIST_ID"
echo "Domain:       https://$CF_DOMAIN"
```

### Step 6 — Apply the bucket policy (the step people forget)

```bash
cat > /tmp/bucket-policy.json <<EOF
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

aws s3api put-bucket-policy --bucket "$BUCKET" --policy file:///tmp/bucket-policy.json
aws s3api get-bucket-policy --bucket "$BUCKET" --query Policy --output text | jq .
```

### Step 7 — Wait and verify

```bash
echo "Waiting for deployment (usually 2–5 minutes)..."
aws cloudfront wait distribution-deployed --id "$DIST_ID"
echo "Deployed."

# 1. The site loads
curl -sS "https://$CF_DOMAIN/" | head -20

# 2. Headers — first request should be a Miss
cfcheck "https://$CF_DOMAIN/"

# 3. Second request should be a Hit
cfcheck "https://$CF_DOMAIN/assets/style.css"
cfcheck "https://$CF_DOMAIN/assets/style.css"

# 4. HTTP redirects to HTTPS
curl -sSI "http://$CF_DOMAIN/" | head -3

# 5. The bucket is NOT publicly reachable — THIS IS THE POINT OF THE LAB
curl -sS -o /dev/null -w '%{http_code}\n' \
  "https://$BUCKET.s3.$AWS_REGION.amazonaws.com/index.html"     # expect 403

# 6. Compression works on the CSS
curl -sSI -H 'Accept-Encoding: br,gzip' "https://$CF_DOMAIN/assets/style.css" \
  | grep -i content-encoding
```

### Expected output

```
HTTP/2 200
content-type: text/html; charset=utf-8
x-cache: Miss from cloudfront          ← first request
x-amz-cf-pop: BOM51-P2

HTTP/2 200
x-cache: Hit from cloudfront           ← second request
age: 7
```

### Known issue you'll hit right now

Open `https://$CF_DOMAIN/about/` in a browser. You get **403 or 404**, not the About page.

**Why:** S3 REST origins have no concept of a directory index. `/about/` maps to an S3 key literally
named `about/`, which doesn't exist. `DefaultRootObject` only works for `/`, not for subdirectories.

**Fix:** Lab 7 solves this with a CloudFront Function. (You could also use an S3 *website* endpoint,
but then the bucket must be public — which defeats the whole lab.)

### What you learned

- OAC + bucket policy with `AWS:SourceArn` is the correct way to serve private S3 content.
- `X-Cache` and `Age` are how you prove caching is happening.
- Content-Type and Cache-Control must be set at upload time, not later.
- `DefaultRootObject` is root-only. Directory routing needs edge logic.

---

## Lab 2 — Custom Domain, ACM & Route 53

**Objective:** Put your own domain in front of the distribution with a free, auto-renewing
certificate.

**Prerequisites:** A domain you control. If it's in Route 53, everything is automatic. If not, you
can still do the ACM part and add the DNS records manually.

### Step 1 — Request the certificate IN us-east-1

```bash
export DOMAIN="cdn.example.com"          # ← change this
export ROOT_DOMAIN="example.com"         # ← change this

CERT_ARN=$(aws acm request-certificate --region us-east-1 \
  --domain-name "$DOMAIN" \
  --validation-method DNS \
  --query CertificateArn --output text)

echo "export CERT_ARN=\"$CERT_ARN\"" >> ~/cloudfront-labs/env.sh
echo "$CERT_ARN"
```

> **If you skip `--region us-east-1`, the certificate will not appear when you attach it. This is
> the single most common CloudFront mistake.**

### Step 2 — Validate via DNS

```bash
# Get the validation record
sleep 10
aws acm describe-certificate --region us-east-1 --certificate-arn "$CERT_ARN" \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord' --output json
```

**If your domain is in Route 53, create it automatically:**

```bash
HZ_ID=$(aws route53 list-hosted-zones-by-name --dns-name "$ROOT_DOMAIN" \
  --query 'HostedZones[0].Id' --output text | sed 's|/hostedzone/||')

VAL=$(aws acm describe-certificate --region us-east-1 --certificate-arn "$CERT_ARN" \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord' --output json)
VNAME=$(echo "$VAL" | jq -r '.Name')
VVALUE=$(echo "$VAL" | jq -r '.Value')

cat > /tmp/validate.json <<EOF
{ "Changes": [{ "Action": "UPSERT", "ResourceRecordSet": {
    "Name": "$VNAME", "Type": "CNAME", "TTL": 300,
    "ResourceRecords": [{ "Value": "$VVALUE" }] }}]}
EOF

aws route53 change-resource-record-sets --hosted-zone-id "$HZ_ID" \
  --change-batch file:///tmp/validate.json
```

**If your DNS is elsewhere:** add that CNAME record in your provider's control panel manually.

```bash
# Wait for issuance
aws acm wait certificate-validated --region us-east-1 --certificate-arn "$CERT_ARN"
aws acm describe-certificate --region us-east-1 --certificate-arn "$CERT_ARN" \
  --query 'Certificate.Status' --output text     # ISSUED
```

### Step 3 — Attach the certificate and alias to the distribution

```bash
source ~/cloudfront-labs/env.sh

aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)

jq --arg cert "$CERT_ARN" --arg d "$DOMAIN" '.DistributionConfig
   | .Aliases = { "Quantity": 1, "Items": [$d] }
   | .ViewerCertificate = {
       "ACMCertificateArn": $cert,
       "SSLSupportMethod": "sni-only",
       "MinimumProtocolVersion": "TLSv1.2_2021",
       "Certificate": $cert,
       "CertificateSource": "acm"
     }' /tmp/full.json > /tmp/new.json

aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json --query 'Distribution.Status' --output text

aws cloudfront wait distribution-deployed --id "$DIST_ID"
```

### Step 4 — Point DNS at CloudFront

```bash
cat > /tmp/dns.json <<EOF
{ "Comment": "Point $DOMAIN at CloudFront",
  "Changes": [
    { "Action": "UPSERT", "ResourceRecordSet": {
        "Name": "$DOMAIN", "Type": "A",
        "AliasTarget": { "HostedZoneId": "Z2FDTNDATAQYW2",
                         "DNSName": "$CF_DOMAIN", "EvaluateTargetHealth": false }}},
    { "Action": "UPSERT", "ResourceRecordSet": {
        "Name": "$DOMAIN", "Type": "AAAA",
        "AliasTarget": { "HostedZoneId": "Z2FDTNDATAQYW2",
                         "DNSName": "$CF_DOMAIN", "EvaluateTargetHealth": false }}}
  ]}
EOF

aws route53 change-resource-record-sets --hosted-zone-id "$HZ_ID" --change-batch file:///tmp/dns.json
```

**Non-Route-53 DNS:** create `CNAME cdn.example.com → $CF_DOMAIN`.

### Verify

```bash
dig +short "$DOMAIN"
dig +short AAAA "$DOMAIN"

curl -sSI "https://$DOMAIN/" | head -5

# Certificate details
echo | openssl s_client -connect "$DOMAIN:443" -servername "$DOMAIN" 2>/dev/null \
  | openssl x509 -noout -subject -dates -issuer

# TLS 1.0 should be rejected
openssl s_client -connect "$DOMAIN:443" -tls1 </dev/null 2>&1 | grep -iE 'alert|failure' | head -2
```

### Common failures

| Symptom | Cause |
|---|---|
| Certificate doesn't appear in the console dropdown | It's not in us-east-1 |
| `InvalidViewerCertificate` | Cert is not ISSUED, or doesn't cover the alias |
| `CNAMEAlreadyExists` | Another distribution (possibly in another account) already claims it — run `list-conflicting-aliases` |
| 403 "The request could not be satisfied" on the new domain | DNS points at CloudFront but the alias isn't on the distribution yet |

### What you learned

- ACM for CloudFront = us-east-1, always.
- `Z2FDTNDATAQYW2` is the universal CloudFront alias target zone ID.
- SNI is free and correct; dedicated IP is a legacy money pit.

---

## Lab 3 — Cache Policies, TTLs & Proving the Cache Works

**Objective:** Build a custom cache policy, watch the cache key change behaviour, and see exactly
how origin `Cache-Control` interacts with CloudFront's Min/Default/Max TTL.

### Step 1 — Demonstrate the query-string problem

```bash
source ~/cloudfront-labs/env.sh

# Create two "products" that differ only by query string
cat > /tmp/product.html <<'EOF'
<!DOCTYPE html><html><body><h1>Product page</h1>
<p>Whatever product you asked for, you are getting THIS cached copy.</p></body></html>
EOF
aws s3 cp /tmp/product.html "s3://$BUCKET/product.html" \
  --content-type "text/html" --cache-control "public, max-age=300"

# CachingOptimized excludes query strings — both requests share a cache key
curl -sSI "https://$CF_DOMAIN/product.html?id=1" | grep -i x-cache
curl -sSI "https://$CF_DOMAIN/product.html?id=2" | grep -i x-cache
```

You'll see the second request is a **Hit** even though the query string is different. In a real app
this is the bug where user B sees user A's product. Let's fix it.

### Step 2 — Create a cache policy that includes the query string

```bash
cat > /tmp/cache-policy.json <<'EOF'
{
  "Name": "LabProductCachePolicy",
  "Comment": "Cache on id + page query strings, and Accept-Language",
  "DefaultTTL": 300,
  "MinTTL": 0,
  "MaxTTL": 86400,
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

CP_ID=$(aws cloudfront create-cache-policy \
  --cache-policy-config file:///tmp/cache-policy.json \
  --query 'CachePolicy.Id' --output text)
echo "export CP_ID=\"$CP_ID\"" >> ~/cloudfront-labs/env.sh
echo "Cache policy: $CP_ID"
```

### Step 3 — Add a behavior for /product.html using the new policy

```bash
source ~/cloudfront-labs/env.sh
aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)

jq --arg cp "$CP_ID" '.DistributionConfig
  | .CacheBehaviors = {
      "Quantity": 1,
      "Items": [{
        "PathPattern": "/product.html",
        "TargetOriginId": "s3-origin",
        "ViewerProtocolPolicy": "redirect-to-https",
        "AllowedMethods": { "Quantity": 2, "Items": ["GET","HEAD"],
                            "CachedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] } },
        "Compress": true,
        "CachePolicyId": $cp,
        "SmoothStreaming": false,
        "FieldLevelEncryptionId": "",
        "FunctionAssociations": { "Quantity": 0 },
        "LambdaFunctionAssociations": { "Quantity": 0 },
        "TrustedKeyGroups": { "Enabled": false, "Quantity": 0 }
      }]
    }' /tmp/full.json > /tmp/new.json

aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json --query 'Distribution.Status' --output text
aws cloudfront wait distribution-deployed --id "$DIST_ID"
```

### Step 4 — Prove the cache key changed

```bash
echo "--- ?id=1 (twice) ---"
curl -sSI "https://$CF_DOMAIN/product.html?id=1" | grep -i x-cache
curl -sSI "https://$CF_DOMAIN/product.html?id=1" | grep -i x-cache

echo "--- ?id=2 (should be a MISS the first time) ---"
curl -sSI "https://$CF_DOMAIN/product.html?id=2" | grep -i x-cache
curl -sSI "https://$CF_DOMAIN/product.html?id=2" | grep -i x-cache

echo "--- ?utm_source=twitter (NOT in the key → should HIT the id-less entry) ---"
curl -sSI "https://$CF_DOMAIN/product.html?utm_source=twitter" | grep -i x-cache
```

**Expected:** `id=1` and `id=2` each get their own cache entry. `utm_source` is ignored entirely,
which is exactly what you want — tracking parameters should never fragment your cache.

### Step 5 — Watch TTL in action

```bash
# Upload with a 60-second max-age
echo "<h1>TTL test $(date -u)</h1>" > /tmp/ttl.html
aws s3 cp /tmp/ttl.html "s3://$BUCKET/ttl.html" \
  --content-type "text/html" --cache-control "public, max-age=60"

echo "--- watching Age climb ---"
for i in $(seq 1 8); do
  printf "t=%2ds  " $((i*10))
  curl -sSI "https://$CF_DOMAIN/ttl.html" | grep -iE '^age|^x-cache' | tr '\n' ' '
  echo
  sleep 10
done
```

You'll see `Age` climb to ~60 and then reset to 0 with a fresh `Miss` — CloudFront revalidated.

### Step 6 — Prove MinTTL overrides no-cache (the dangerous behaviour)

```bash
# Upload an object the origin explicitly says not to cache
echo "<h1>Should not be cached: $(date -u)</h1>" > /tmp/nocache.html
aws s3 cp /tmp/nocache.html "s3://$BUCKET/nocache.html" \
  --content-type "text/html" --cache-control "no-cache"

curl -sSI "https://$CF_DOMAIN/nocache.html" | grep -iE 'x-cache|cache-control'
curl -sSI "https://$CF_DOMAIN/nocache.html" | grep -iE 'x-cache|cache-control'
```

With `MinTTL: 0` (as in our policy) this correctly revalidates. If you set `MinTTL: 300`, CloudFront
would serve a stale copy for five minutes despite the origin's `no-cache`. **This is why MinTTL
should stay at 0 unless you have a specific reason.**

### The TTL decision table (memorize this)

| Origin sends | MinTTL | DefaultTTL | MaxTTL | CloudFront caches for |
|---|---|---|---|---|
| nothing | 0 | 86400 | 31536000 | **86400 s** (DefaultTTL) |
| `max-age=60` | 0 | 86400 | 31536000 | **60 s** |
| `max-age=60` | 300 | 86400 | 31536000 | **300 s** (clamped up by MinTTL) |
| `max-age=99999999` | 0 | 86400 | 86400 | **86400 s** (clamped down by MaxTTL) |
| `s-maxage=120, max-age=60` | 0 | 86400 | 31536000 | **120 s** (s-maxage wins for CDNs) |
| `no-cache` | 0 | 86400 | 31536000 | **stored but revalidated every time** |
| `no-store` | 0 | 86400 | 31536000 | **not stored at all** |
| `private` | 0 | 86400 | 31536000 | **not stored** (CloudFront is a shared cache) |

### What you learned

- The default cache key is path-only. Anything else you need must be added explicitly.
- Query-string allow-lists beat deny-lists: new tracking params are ignored automatically.
- `s-maxage` beats `max-age` for CloudFront.
- A high `MinTTL` silently overrides your origin's caching intent.

---

## Lab 4 — Multiple Behaviors & a Second Origin

**Objective:** Serve a static site and a dynamic API from **one domain** using path-based routing —
eliminating CORS and giving you one WAF, one certificate, one log stream.

**Architecture:**

```
                         ┌── /api/*     → API Gateway  (CachingDisabled)
   cdn.example.com ─────►├── /assets/*  → S3           (CachingOptimized, 1y)
                         └── *          → S3           (CachingOptimized)
```

We'll use a Lambda Function URL behind API Gateway-style routing; it's the cheapest dynamic origin
for a lab. (Substitute an ALB if you have one.)

### Step 1 — Create a tiny API with a Lambda Function URL

```bash
source ~/cloudfront-labs/env.sh

cat > /tmp/api.mjs <<'EOF'
export const handler = async (event) => {
  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/json', 'Cache-Control': 'no-store' },
    body: JSON.stringify({
      ok: true,
      path: event.rawPath,
      time: new Date().toISOString(),
      country: event.headers?.['cloudfront-viewer-country'] ?? 'unknown',
      ua: event.headers?.['user-agent']?.slice(0, 40)
    })
  };
};
EOF
(cd /tmp && zip -q api.zip api.mjs)

# Execution role
cat > /tmp/lambda-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name cf-lab-api-role \
  --assume-role-policy-document file:///tmp/lambda-trust.json >/dev/null 2>&1 || true
aws iam attach-role-policy --role-name cf-lab-api-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
sleep 10

aws lambda create-function --region "$AWS_REGION" \
  --function-name cf-lab-api \
  --runtime nodejs22.x \
  --role "arn:aws:iam::$ACCOUNT_ID:role/cf-lab-api-role" \
  --handler api.handler \
  --zip-file fileb:///tmp/api.zip \
  --timeout 10 --memory-size 128 >/dev/null

# Function URL with IAM auth (so only CloudFront's signed requests get in)
API_URL=$(aws lambda create-function-url-config --region "$AWS_REGION" \
  --function-name cf-lab-api --auth-type AWS_IAM \
  --query 'FunctionUrl' --output text)

API_HOST=$(echo "$API_URL" | sed -E 's|https://([^/]+)/?|\1|')
echo "export API_HOST=\"$API_HOST\"" >> ~/cloudfront-labs/env.sh
echo "API host: $API_HOST"
```

### Step 2 — OAC for the Lambda Function URL

```bash
LAMBDA_OAC=$(aws cloudfront create-origin-access-control --origin-access-control-config '{
  "Name": "oac-lambda-api",
  "SigningProtocol": "sigv4",
  "SigningBehavior": "always",
  "OriginAccessControlOriginType": "lambda"
}' --query 'OriginAccessControl.Id' --output text)
echo "export LAMBDA_OAC=\"$LAMBDA_OAC\"" >> ~/cloudfront-labs/env.sh

# Allow CloudFront to invoke the Function URL
source ~/cloudfront-labs/env.sh
aws lambda add-permission --region "$AWS_REGION" \
  --function-name cf-lab-api \
  --statement-id AllowCloudFrontServicePrincipal \
  --action lambda:InvokeFunctionUrl \
  --principal cloudfront.amazonaws.com \
  --source-arn "arn:aws:cloudfront::$ACCOUNT_ID:distribution/$DIST_ID" \
  --function-url-auth-type AWS_IAM
```

### Step 3 — Add the origin and the /api/* behavior

```bash
source ~/cloudfront-labs/env.sh
aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)

# Look up the managed policy IDs rather than hardcoding them
CACHING_DISABLED=$(aws cloudfront list-cache-policies --type managed \
  --query "CachePolicyList.Items[?CachePolicy.CachePolicyConfig.Name=='Managed-CachingDisabled'].CachePolicy.Id" \
  --output text)
ALL_NO_HOST=$(aws cloudfront list-origin-request-policies --type managed \
  --query "OriginRequestPolicyList.Items[?OriginRequestPolicy.OriginRequestPolicyConfig.Name=='Managed-AllViewerExceptHostHeader'].OriginRequestPolicy.Id" \
  --output text)
echo "CachingDisabled=$CACHING_DISABLED  AllViewerExceptHost=$ALL_NO_HOST"

jq --arg host "$API_HOST" --arg oac "$LAMBDA_OAC" \
   --arg cp "$CACHING_DISABLED" --arg orp "$ALL_NO_HOST" '.DistributionConfig
  | .Origins.Items += [{
      "Id": "lambda-api",
      "DomainName": $host,
      "OriginPath": "",
      "OriginAccessControlId": $oac,
      "CustomHeaders": { "Quantity": 0 },
      "CustomOriginConfig": {
        "HTTPPort": 80, "HTTPSPort": 443,
        "OriginProtocolPolicy": "https-only",
        "OriginSslProtocols": { "Quantity": 1, "Items": ["TLSv1.2"] },
        "OriginReadTimeout": 30,
        "OriginKeepaliveTimeout": 5
      },
      "ConnectionAttempts": 3, "ConnectionTimeout": 10,
      "OriginShield": { "Enabled": false }
    }]
  | .Origins.Quantity = (.Origins.Items | length)
  | .CacheBehaviors.Items = ((.CacheBehaviors.Items // []) + [{
      "PathPattern": "/api/*",
      "TargetOriginId": "lambda-api",
      "ViewerProtocolPolicy": "https-only",
      "AllowedMethods": {
        "Quantity": 7, "Items": ["GET","HEAD","OPTIONS","PUT","POST","PATCH","DELETE"],
        "CachedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] }
      },
      "Compress": true,
      "CachePolicyId": $cp,
      "OriginRequestPolicyId": $orp,
      "SmoothStreaming": false,
      "FieldLevelEncryptionId": "",
      "FunctionAssociations": { "Quantity": 0 },
      "LambdaFunctionAssociations": { "Quantity": 0 },
      "TrustedKeyGroups": { "Enabled": false, "Quantity": 0 }
    }])
  | .CacheBehaviors.Quantity = (.CacheBehaviors.Items | length)' \
  /tmp/full.json > /tmp/new.json

aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json --query 'Distribution.Status' --output text
aws cloudfront wait distribution-deployed --id "$DIST_ID"
```

### Verify

```bash
# Static path — cached
cfcheck "https://$CF_DOMAIN/assets/style.css"

# API path — never cached, always fresh
curl -sS "https://$CF_DOMAIN/api/hello" | jq .
curl -sSI "https://$CF_DOMAIN/api/hello" | grep -i x-cache     # Miss every time

# Two API calls should return different timestamps
curl -sS "https://$CF_DOMAIN/api/one" | jq -r .time
sleep 2
curl -sS "https://$CF_DOMAIN/api/two" | jq -r .time

# The Lambda URL is NOT directly callable
curl -sS -o /dev/null -w '%{http_code}\n' "https://$API_HOST/api/hello"   # expect 403
```

### Behavior ordering — the trap

```bash
# See the order CloudFront will evaluate
aws cloudfront get-distribution-config --id "$DIST_ID" \
  --query 'DistributionConfig.CacheBehaviors.Items[].{Order:PathPattern,Origin:TargetOriginId}' \
  --output table
```

If you ever add a `*` pattern to `CacheBehaviors` (not the default behavior), it will shadow
everything after it. Order most-specific → least-specific, and remember the default `*` behavior is
always evaluated last regardless.

### What you learned

- One domain can front many origins; path patterns are your router.
- Use `AllViewerExceptHostHeader` for API-style origins — the `Host` header would otherwise break
  routing or signing.
- OAC works for Lambda Function URLs, not just S3.
- `CachingDisabled` is a real, correct choice — CloudFront still adds value via TLS and connection
  reuse.

---

## Lab 5 — Invalidations vs Versioned Assets

**Objective:** Feel the difference between the slow/costly way and the fast/free way of shipping
updates.

### Step 1 — The invalidation way

```bash
source ~/cloudfront-labs/env.sh

# Change the CSS
echo "body { background: #101418; color: #e8eaed; }" >> site/assets/style.css
aws s3 cp site/assets/style.css "s3://$BUCKET/assets/style.css" \
  --content-type "text/css" --cache-control "public, max-age=31536000, immutable"

# Old version still served — the edge doesn't know
curl -sS "https://$CF_DOMAIN/assets/style.css" | tail -2

# Now invalidate and time it
START=$(date +%s)
INV_ID=$(aws cloudfront create-invalidation --distribution-id "$DIST_ID" \
  --paths "/assets/style.css" --query 'Invalidation.Id' --output text)
aws cloudfront wait invalidation-completed --distribution-id "$DIST_ID" --id "$INV_ID"
echo "Invalidation took $(( $(date +%s) - START )) seconds"

curl -sS "https://$CF_DOMAIN/assets/style.css" | tail -2      # new content
```

### Step 2 — The versioned way

```bash
# Content-hash the filename
HASH=$(md5sum site/assets/style.css | cut -c1-8)
cp site/assets/style.css "site/assets/style.$HASH.css"

aws s3 cp "site/assets/style.$HASH.css" "s3://$BUCKET/assets/style.$HASH.css" \
  --content-type "text/css" --cache-control "public, max-age=31536000, immutable"

# Point index.html at it
sed -i "s|/assets/style[^\"]*\.css|/assets/style.$HASH.css|" site/index.html
aws s3 cp site/index.html "s3://$BUCKET/index.html" \
  --content-type "text/html; charset=utf-8" --cache-control "no-cache"

# ONE invalidation path, for the entry point only
aws cloudfront create-invalidation --distribution-id "$DIST_ID" --paths "/index.html" \
  --query 'Invalidation.Id' --output text

# The new CSS is instantly available — it's a new cache key
curl -sSI "https://$CF_DOMAIN/assets/style.$HASH.css" | grep -iE 'HTTP/|x-cache|cache-control'
```

### Step 3 — Build the production deploy script

```bash
cat > ~/cloudfront-labs/deploy.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
source ~/cloudfront-labs/env.sh

SRC="${1:-./site}"

echo "==> Uploading immutable assets"
aws s3 sync "$SRC/assets" "s3://$BUCKET/assets" \
  --cache-control "public, max-age=31536000, immutable" \
  --delete

echo "==> Fixing content types"
aws s3 cp "s3://$BUCKET/assets" "s3://$BUCKET/assets" --recursive \
  --exclude "*" --include "*.css" --content-type "text/css" \
  --metadata-directive REPLACE --cache-control "public, max-age=31536000, immutable"
aws s3 cp "s3://$BUCKET/assets" "s3://$BUCKET/assets" --recursive \
  --exclude "*" --include "*.js" --content-type "application/javascript" \
  --metadata-directive REPLACE --cache-control "public, max-age=31536000, immutable"

echo "==> Uploading HTML entry points (short TTL)"
find "$SRC" -name '*.html' | while read -r f; do
  KEY="${f#$SRC/}"
  aws s3 cp "$f" "s3://$BUCKET/$KEY" \
    --content-type "text/html; charset=utf-8" --cache-control "no-cache"
done

echo "==> Invalidating entry points only"
INV=$(aws cloudfront create-invalidation --distribution-id "$DIST_ID" \
  --paths "/" "/index.html" "/*/index.html" --query 'Invalidation.Id' --output text)
aws cloudfront wait invalidation-completed --distribution-id "$DIST_ID" --id "$INV"

echo "==> Deployed. https://$CF_DOMAIN/"
EOF

chmod +x ~/cloudfront-labs/deploy.sh
~/cloudfront-labs/deploy.sh ./site
```

### Cost comparison

```
Daily deploys with /* invalidation:
   30 deploys/month × 1 path = 30 paths     → free, but every deploy cold-starts the cache
Daily deploys invalidating 40 asset paths:
   30 × 40 = 1,200 paths → 200 billable      → small cost, plus cache churn
Versioned assets + /index.html only:
   30 × 1 = 30 paths → free, cache stays warm for every unchanged asset ✔
```

### What you learned

- Invalidation is a repair tool, not a deployment mechanism.
- Content hashing makes cache busting free, instant, and atomic.
- `/*` invalidation is worse than it looks: it cold-starts your entire cache, so your origin gets a
  traffic spike right after every deploy.

---

## Lab 6 — Origin Groups & Failover

**Objective:** Configure automatic origin failover and then *prove* it works by deliberately
breaking the primary origin.

**Architecture:**

```
                     ┌── 200 ────────────────► serve from PRIMARY (S3 bucket A)
   CloudFront ──────►│
                     └── 403/404/5xx/timeout ► retry against SECONDARY (S3 bucket B)
```

### Step 1 — Create a secondary bucket with fallback content

```bash
source ~/cloudfront-labs/env.sh

export BUCKET2="${BUCKET}-dr"
echo "export BUCKET2=\"$BUCKET2\"" >> ~/cloudfront-labs/env.sh

aws s3api create-bucket --bucket "$BUCKET2" --region "$AWS_REGION" \
  --create-bucket-configuration LocationConstraint="$AWS_REGION"

cat > /tmp/failover.html <<'EOF'
<!DOCTYPE html><html><body style="font-family:system-ui;padding:3rem">
<h1>Failover origin</h1>
<p>You are seeing the SECONDARY origin. The primary returned an error.</p>
</body></html>
EOF

aws s3 cp /tmp/failover.html "s3://$BUCKET2/failover-test.html" \
  --content-type "text/html" --cache-control "no-cache"
```

### Step 2 — Create an OAC and bucket policy for the secondary

```bash
OAC2_ID=$(aws cloudfront create-origin-access-control --origin-access-control-config "{
  \"Name\": \"oac-$BUCKET2\",
  \"SigningProtocol\": \"sigv4\",
  \"SigningBehavior\": \"always\",
  \"OriginAccessControlOriginType\": \"s3\"
}" --query 'OriginAccessControl.Id' --output text)
echo "export OAC2_ID=\"$OAC2_ID\"" >> ~/cloudfront-labs/env.sh

cat > /tmp/bp2.json <<EOF
{ "Version": "2012-10-17", "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET2/*",
    "Condition": { "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::$ACCOUNT_ID:distribution/$DIST_ID" }}}]}
EOF
aws s3api put-bucket-policy --bucket "$BUCKET2" --policy file:///tmp/bp2.json
```

### Step 3 — Add the secondary origin and build the origin group

```bash
source ~/cloudfront-labs/env.sh
aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)

jq --arg b2 "$BUCKET2.s3.$AWS_REGION.amazonaws.com" --arg oac2 "$OAC2_ID" '.DistributionConfig
  | .Origins.Items += [{
      "Id": "s3-origin-dr",
      "DomainName": $b2,
      "OriginPath": "",
      "OriginAccessControlId": $oac2,
      "S3OriginConfig": { "OriginAccessIdentity": "" },
      "CustomHeaders": { "Quantity": 0 },
      "ConnectionAttempts": 3, "ConnectionTimeout": 10,
      "OriginShield": { "Enabled": false }
    }]
  | .Origins.Quantity = (.Origins.Items | length)
  | .OriginGroups = {
      "Quantity": 1,
      "Items": [{
        "Id": "s3-failover-group",
        "FailoverCriteria": {
          "StatusCodes": { "Quantity": 6, "Items": [403, 404, 500, 502, 503, 504] }
        },
        "Members": {
          "Quantity": 2,
          "Items": [{ "OriginId": "s3-origin" }, { "OriginId": "s3-origin-dr" }]
        }
      }]
    }
  | .DefaultCacheBehavior.TargetOriginId = "s3-failover-group"' \
  /tmp/full.json > /tmp/new.json

aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json --query 'Distribution.Status' --output text
aws cloudfront wait distribution-deployed --id "$DIST_ID"
```

### Step 4 — Prove failover works

```bash
# This key exists ONLY in the secondary bucket.
# The primary returns 403 (private bucket, missing key) → CloudFront fails over.
curl -sS "https://$CF_DOMAIN/failover-test.html"
```

You should see the **Failover origin** page. The primary was asked first, returned 403, and
CloudFront transparently retried the secondary.

```bash
# A key that exists in the primary is served normally
curl -sSI "https://$CF_DOMAIN/index.html" | head -3
```

### Step 5 — Understand the limits

```bash
# POST is NOT retried — failover only applies to GET/HEAD/OPTIONS
curl -sS -X POST -o /dev/null -w '%{http_code}\n' "https://$CF_DOMAIN/failover-test.html"
```

**Rules to internalize:**

| Rule | Detail |
|---|---|
| Methods | Only `GET`, `HEAD`, `OPTIONS` fail over. Non-idempotent methods never do. |
| Trigger | Selected status codes, connection timeouts, and connection failures. |
| Health checks | **None.** Failover is reactive, per request. No background probing. |
| Caching | The failed response isn't cached; the successful one is. |
| Latency | A failover request pays for both attempts. Keep `ConnectionAttempts`/timeouts sane. |
| Behaviors | The origin group is set as the behavior's target origin, not as an origin itself. |

### What you learned

- Origin groups give you cheap, config-only resilience.
- Add `403` and `404` to the failover criteria only when you genuinely want fallback content —
  otherwise every missing file doubles your origin requests.
- For true active-active with health checking, combine this with Route 53 health checks.

---

## Lab 7 — CloudFront Functions + KeyValueStore

**Objective:** Fix the `/about/` 404 from Lab 1, add canonical redirects, strip tracking parameters,
and drive a redirect map from KeyValueStore without redeploying code.

### Step 1 — The URI rewrite function

```bash
source ~/cloudfront-labs/env.sh
cd ~/cloudfront-labs

cat > uri-rewrite.js <<'EOF'
function handler(event) {
    var request = event.request;
    var uri = request.uri;

    // /about/  →  /about/index.html
    if (uri.endsWith('/')) {
        request.uri = uri + 'index.html';
    }
    // /about   →  /about/index.html   (no file extension present)
    else if (!uri.includes('.')) {
        request.uri = uri + '/index.html';
    }

    // Drop tracking params so they never fragment the cache
    var qs = request.querystring;
    var junk = ['utm_source','utm_medium','utm_campaign','utm_term','utm_content',
                'fbclid','gclid','msclkid','mc_cid','mc_eid'];
    for (var i = 0; i < junk.length; i++) {
        if (qs[junk[i]]) { delete qs[junk[i]]; }
    }

    return request;
}
EOF

aws cloudfront create-function \
  --name lab-uri-rewrite \
  --function-config '{"Comment":"Directory index + tracking param strip","Runtime":"cloudfront-js-2.0"}' \
  --function-code fileb://uri-rewrite.js \
  --query 'FunctionSummary.FunctionMetadata.FunctionARN' --output text
```

### Step 2 — Test it before publishing (this is the killer feature)

```bash
cat > /tmp/ev-dir.json <<'EOF'
{ "version":"1.0",
  "context":{"eventType":"viewer-request"},
  "viewer":{"ip":"1.2.3.4"},
  "request":{"method":"GET","uri":"/about/","headers":{"host":{"value":"example.com"}},
             "querystring":{"utm_source":{"value":"twitter"},"id":{"value":"7"}},
             "cookies":{}} }
EOF

FN_ETAG=$(aws cloudfront describe-function --name lab-uri-rewrite --query ETag --output text)

aws cloudfront test-function \
  --name lab-uri-rewrite --if-match "$FN_ETAG" --stage DEVELOPMENT \
  --event-object fileb:///tmp/ev-dir.json \
  --query 'TestResult.{Compute:ComputeUtilization,Errors:FunctionErrorMessage,Out:FunctionOutput}' \
  --output json | jq -r '.Out | fromjson | .request | {uri, querystring}'
```

**Expected:** `uri` becomes `/about/index.html`, `utm_source` is gone, `id` survives.
`ComputeUtilization` should be well under 100 (it's a % of the 1 ms budget).

Test the extension-less case too:

```bash
sed 's|"/about/"|"/about"|' /tmp/ev-dir.json > /tmp/ev-noext.json
aws cloudfront test-function --name lab-uri-rewrite --if-match "$FN_ETAG" \
  --stage DEVELOPMENT --event-object fileb:///tmp/ev-noext.json \
  --query 'TestResult.FunctionOutput' --output text | jq -r '.request.uri'
```

### Step 3 — Publish and associate

```bash
FN_ETAG=$(aws cloudfront describe-function --name lab-uri-rewrite --query ETag --output text)
aws cloudfront publish-function --name lab-uri-rewrite --if-match "$FN_ETAG" >/dev/null

FN_ARN=$(aws cloudfront describe-function --name lab-uri-rewrite --stage LIVE \
  --query 'FunctionSummary.FunctionMetadata.FunctionARN' --output text)
echo "export FN_ARN=\"$FN_ARN\"" >> ~/cloudfront-labs/env.sh

aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)
jq --arg arn "$FN_ARN" '.DistributionConfig
  | .DefaultCacheBehavior.FunctionAssociations = {
      "Quantity": 1,
      "Items": [{ "FunctionARN": $arn, "EventType": "viewer-request" }]
    }' /tmp/full.json > /tmp/new.json

aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json --query 'Distribution.Status' --output text
aws cloudfront wait distribution-deployed --id "$DIST_ID"
```

### Verify

```bash
# The Lab 1 bug is now fixed
curl -sS "https://$CF_DOMAIN/about/"
curl -sS "https://$CF_DOMAIN/about"

# Tracking params are ignored — this should be a HIT on the clean cache entry
curl -sSI "https://$CF_DOMAIN/?utm_source=newsletter" | grep -i x-cache

# Function logs (us-east-1, always)
aws logs tail /aws/cloudfront/function/lab-uri-rewrite --region us-east-1 --since 10m
```

### Step 4 — KeyValueStore-driven redirects

```bash
cd ~/cloudfront-labs

KVS_ARN=$(aws cloudfront create-key-value-store --name lab-redirects \
  --comment "Redirect map" --query 'KeyValueStore.ARN' --output text)
echo "export KVS_ARN=\"$KVS_ARN\"" >> ~/cloudfront-labs/env.sh

# Wait for it to become READY
until [ "$(aws cloudfront describe-key-value-store --name lab-redirects \
  --query 'KeyValueStore.Status' --output text)" = "READY" ]; do sleep 5; echo -n .; done; echo

# Seed some redirects
KVS_ETAG=$(aws cloudfront-keyvaluestore describe-key-value-store --kvs-arn "$KVS_ARN" \
  --query ETag --output text)
aws cloudfront-keyvaluestore update-keys --kvs-arn "$KVS_ARN" --if-match "$KVS_ETAG" \
  --puts '[{"Key":"/old-home","Value":"/"},
           {"Key":"/legacy-docs","Value":"/about/"},
           {"Key":"/promo","Value":"/product.html?id=1"}]'

aws cloudfront-keyvaluestore list-keys --kvs-arn "$KVS_ARN"
```

```bash
cat > redirect-fn.js <<'EOF'
import cf from 'cloudfront';
const kvs = cf.kvs();

async function handler(event) {
    const request = event.request;
    try {
        const target = await kvs.get(request.uri);
        if (target) {
            return {
                statusCode: 301,
                statusDescription: 'Moved Permanently',
                headers: { 'location': { value: target },
                           'cache-control': { value: 'max-age=3600' } }
            };
        }
    } catch (e) {
        // key miss — fall through to normal handling
    }

    // Keep the directory-index behaviour from the previous function
    if (request.uri.endsWith('/')) { request.uri += 'index.html'; }
    else if (!request.uri.includes('.')) { request.uri += '/index.html'; }
    return request;
}
EOF

aws cloudfront create-function \
  --name lab-redirects \
  --function-config "{\"Comment\":\"KVS redirects\",\"Runtime\":\"cloudfront-js-2.0\",\"KeyValueStoreAssociations\":{\"Quantity\":1,\"Items\":[{\"KeyValueStoreARN\":\"$KVS_ARN\"}]}}" \
  --function-code fileb://redirect-fn.js >/dev/null

RFN_ETAG=$(aws cloudfront describe-function --name lab-redirects --query ETag --output text)
aws cloudfront publish-function --name lab-redirects --if-match "$RFN_ETAG" >/dev/null

RFN_ARN=$(aws cloudfront describe-function --name lab-redirects --stage LIVE \
  --query 'FunctionSummary.FunctionMetadata.FunctionARN' --output text)

aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)
jq --arg arn "$RFN_ARN" '.DistributionConfig
  | .DefaultCacheBehavior.FunctionAssociations = {
      "Quantity": 1, "Items": [{ "FunctionARN": $arn, "EventType": "viewer-request" }] }' \
  /tmp/full.json > /tmp/new.json
aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json >/dev/null
aws cloudfront wait distribution-deployed --id "$DIST_ID"
```

### Verify the KVS redirects

```bash
curl -sSI "https://$CF_DOMAIN/old-home"    | grep -iE 'HTTP/|location'
curl -sSI "https://$CF_DOMAIN/legacy-docs" | grep -iE 'HTTP/|location'

# Now add a NEW redirect WITHOUT touching the function code
KVS_ETAG=$(aws cloudfront-keyvaluestore describe-key-value-store --kvs-arn "$KVS_ARN" \
  --query ETag --output text)
aws cloudfront-keyvaluestore put-key --kvs-arn "$KVS_ARN" --if-match "$KVS_ETAG" \
  --key "/brand-new" --value "/about/"

sleep 20
curl -sSI "https://$CF_DOMAIN/brand-new" | grep -iE 'HTTP/|location'
```

**That's the point of KeyValueStore:** data changes propagate globally in seconds with no function
redeploy and no distribution update.

### What you learned

- `test-function` lets you validate edge logic before it touches production traffic.
- CloudFront Functions run on **every** request — keep `ComputeUtilization` low.
- Only one function per event type per behavior; combine logic into one function.
- KVS separates configuration from code, which is exactly what you want for redirect maps and
  feature flags.

---

## Lab 8 — Lambda@Edge

**Objective:** Run real compute on a cache miss. We'll add security headers and country-based
routing hints at the **origin-response** trigger, then look at the operational reality of
Lambda@Edge (regional logs, versioning, slow deletes).

> **Everything in this lab happens in `us-east-1`.**

### Step 1 — The execution role (both principals required)

```bash
source ~/cloudfront-labs/env.sh

cat > /tmp/edge-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": ["lambda.amazonaws.com", "edgelambda.amazonaws.com"] },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role --role-name cf-lab-edge-role \
  --assume-role-policy-document file:///tmp/edge-trust.json >/dev/null 2>&1 || true
aws iam attach-role-policy --role-name cf-lab-edge-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
sleep 10
```

> Forgetting `edgelambda.amazonaws.com` gives you
> `The function execution role must be assumable with these principals` at association time.

### Step 2 — Write and deploy the function

```bash
cat > /tmp/edge.mjs <<'EOF'
export const handler = async (event) => {
  const response = event.Records[0].cf.response;
  const request  = event.Records[0].cf.request;

  const h = response.headers;
  const set = (key, name, value) => { h[key] = [{ key: name, value }]; };

  set('strict-transport-security', 'Strict-Transport-Security',
      'max-age=63072000; includeSubDomains; preload');
  set('x-content-type-options', 'X-Content-Type-Options', 'nosniff');
  set('x-frame-options', 'X-Frame-Options', 'DENY');
  set('referrer-policy', 'Referrer-Policy', 'strict-origin-when-cross-origin');

  // Echo back some viewer context so you can see it working
  const country = request.headers['cloudfront-viewer-country']?.[0]?.value ?? 'XX';
  set('x-edge-country', 'X-Edge-Country', country);
  set('x-edge-processed', 'X-Edge-Processed', new Date().toISOString());

  delete h['server'];
  return response;
};
EOF

(cd /tmp && rm -f edge.zip && zip -q edge.zip edge.mjs)

aws lambda create-function --region us-east-1 \
  --function-name cf-lab-edge \
  --runtime nodejs22.x \
  --role "arn:aws:iam::$ACCOUNT_ID:role/cf-lab-edge-role" \
  --handler edge.handler \
  --zip-file fileb:///tmp/edge.zip \
  --timeout 5 --memory-size 128 >/dev/null

# PUBLISH A VERSION — you cannot associate $LATEST
LE_ARN=$(aws lambda publish-version --region us-east-1 \
  --function-name cf-lab-edge --query FunctionArn --output text)
echo "export LE_ARN=\"$LE_ARN\"" >> ~/cloudfront-labs/env.sh
echo "$LE_ARN"     # note the trailing :1
```

### Step 3 — Associate at origin-response

```bash
source ~/cloudfront-labs/env.sh
aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)

jq --arg arn "$LE_ARN" '.DistributionConfig
  | .DefaultCacheBehavior.LambdaFunctionAssociations = {
      "Quantity": 1,
      "Items": [{ "LambdaFunctionARN": $arn, "EventType": "origin-response", "IncludeBody": false }]
    }' /tmp/full.json > /tmp/new.json

aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json --query 'Distribution.Status' --output text
aws cloudfront wait distribution-deployed --id "$DIST_ID"
```

### Verify — and notice the crucial detail

```bash
# Force a cache MISS (unique query string) so origin-response actually fires
curl -sSI "https://$CF_DOMAIN/index.html?bust=$RANDOM" \
  | grep -iE 'x-cache|strict-transport|x-edge-country|x-edge-processed'

# Now a cached HIT — origin-response did NOT run, but the headers are still there
# because they were baked into the CACHED response
curl -sSI "https://$CF_DOMAIN/index.html" | grep -iE 'x-cache|x-edge-processed'
```

**The lesson:** at `origin-response`, your modifications are stored in the cache. Subsequent hits
serve those headers without invoking Lambda — cheap, but it also means `X-Edge-Processed` shows the
time of the *original* miss, not the current request. If you need per-request logic, use
`viewer-response` instead.

### Step 4 — Find the logs (they're scattered)

```bash
for R in us-east-1 eu-west-1 ap-south-1 ap-southeast-1 ap-northeast-1 us-west-2 sa-east-1; do
  G=$(aws logs describe-log-groups --region "$R" \
      --log-group-name-prefix "/aws/lambda/us-east-1.cf-lab-edge" \
      --query 'logGroups[].logGroupName' --output text 2>/dev/null)
  [ -n "$G" ] && echo "$R → $G"
done
```

Logs land in the region nearest the Regional Edge Cache that executed the function. If you're in
India, check `ap-south-1` — not `us-east-1`.

### Step 5 — Update the code (versioning workflow)

```bash
sed -i "s/X-Edge-Processed/X-Edge-Version-2/" /tmp/edge.mjs
(cd /tmp && rm -f edge.zip && zip -q edge.zip edge.mjs)

aws lambda update-function-code --region us-east-1 \
  --function-name cf-lab-edge --zip-file fileb:///tmp/edge.zip >/dev/null

NEW_ARN=$(aws lambda publish-version --region us-east-1 \
  --function-name cf-lab-edge --query FunctionArn --output text)
echo "New version: $NEW_ARN"      # :2

# Re-associate the NEW version — CloudFront pins to the exact ARN
aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)
jq --arg arn "$NEW_ARN" '.DistributionConfig
  | .DefaultCacheBehavior.LambdaFunctionAssociations.Items[0].LambdaFunctionARN = $arn' \
  /tmp/full.json > /tmp/new.json
aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json >/dev/null
```

### Step 6 — CloudFront Functions vs Lambda@Edge, measured

```bash
# Same header-setting job, both ways. Compare TTFB.
for i in 1 2 3; do
  curl -sS -o /dev/null -w 'ttfb=%{time_starttransfer}s total=%{time_total}s\n' \
    "https://$CF_DOMAIN/index.html?bust=$RANDOM"
done
```

**Rule of thumb:** if a CloudFront Function or a response headers policy can do the job, it will be
faster and cheaper than Lambda@Edge. Reach for Lambda@Edge only when you need network access, the
body, origin-side triggers, or more than 10 KB of code.

### What you learned

- Lambda@Edge requires us-east-1, a published version, and a dual-principal trust policy.
- Origin triggers only fire on cache misses; the result gets cached.
- Logs are regional and easy to lose.
- Deleting is slow (Lab 15 covers this).

---

## Lab 9 — Signed URLs & Signed Cookies

**Objective:** Lock a path behind cryptographic, time-limited access.

### Step 1 — Private content and a key pair

```bash
source ~/cloudfront-labs/env.sh
mkdir -p ~/cloudfront-labs/keys && cd ~/cloudfront-labs/keys

echo "<h1>PREMIUM CONTENT</h1><p>Only signed requests reach this.</p>" > /tmp/premium.html
aws s3 cp /tmp/premium.html "s3://$BUCKET/premium/index.html" \
  --content-type "text/html" --cache-control "no-cache"
echo '{"secret":"members only"}' > /tmp/premium.json
aws s3 cp /tmp/premium.json "s3://$BUCKET/premium/data.json" \
  --content-type "application/json"

openssl genrsa -out private_key.pem 2048
openssl rsa -pubout -in private_key.pem -out public_key.pem
chmod 600 private_key.pem
```

### Step 2 — Upload the public key and create a key group

```bash
PUBKEY=$(cat public_key.pem)

KEY_ID=$(aws cloudfront create-public-key --public-key-config "{
  \"CallerReference\": \"lab-pk-$(date +%s)\",
  \"Name\": \"lab-signing-key\",
  \"EncodedKey\": \"$PUBKEY\",
  \"Comment\": \"Signed URL lab\"
}" --query 'PublicKey.Id' --output text)

KG_ID=$(aws cloudfront create-key-group --key-group-config "{
  \"Name\": \"lab-premium-keys\",
  \"Items\": [\"$KEY_ID\"],
  \"Comment\": \"Trusted for /premium/*\"
}" --query 'KeyGroup.Id' --output text)

echo "export KEY_ID=\"$KEY_ID\"" >> ~/cloudfront-labs/env.sh
echo "export KG_ID=\"$KG_ID\""   >> ~/cloudfront-labs/env.sh
echo "Key-Pair-Id: $KEY_ID   KeyGroup: $KG_ID"
```

### Step 3 — Add a restricted cache behavior for /premium/*

```bash
source ~/cloudfront-labs/env.sh
CACHING_DISABLED=$(aws cloudfront list-cache-policies --type managed \
  --query "CachePolicyList.Items[?CachePolicy.CachePolicyConfig.Name=='Managed-CachingDisabled'].CachePolicy.Id" \
  --output text)

aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)

jq --arg kg "$KG_ID" --arg cp "$CACHING_DISABLED" '.DistributionConfig
  | .CacheBehaviors.Items = ((.CacheBehaviors.Items // []) + [{
      "PathPattern": "/premium/*",
      "TargetOriginId": "s3-origin",
      "ViewerProtocolPolicy": "https-only",
      "AllowedMethods": { "Quantity": 2, "Items": ["GET","HEAD"],
                          "CachedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] } },
      "Compress": true,
      "CachePolicyId": $cp,
      "SmoothStreaming": false,
      "FieldLevelEncryptionId": "",
      "FunctionAssociations": { "Quantity": 0 },
      "LambdaFunctionAssociations": { "Quantity": 0 },
      "TrustedKeyGroups": { "Enabled": true, "Quantity": 1, "Items": [$kg] }
    }])
  | .CacheBehaviors.Quantity = (.CacheBehaviors.Items | length)' \
  /tmp/full.json > /tmp/new.json

aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json --query 'Distribution.Status' --output text
aws cloudfront wait distribution-deployed --id "$DIST_ID"
```

### Step 4 — Unsigned access is blocked

```bash
curl -sS -o /dev/null -w 'unsigned → %{http_code}\n' "https://$CF_DOMAIN/premium/index.html"
# expect 403
curl -sS "https://$CF_DOMAIN/premium/index.html" | head -20
# CloudFront's error XML: "Missing Key-Pair-Id query parameter or cookie value"
```

### Step 5 — Sign a URL (canned policy)

```bash
cd ~/cloudfront-labs/keys
source ~/cloudfront-labs/env.sh

EXPIRY=$(date -u -d '+1 hour' +%Y-%m-%dT%H:%M:%SZ)

SIGNED=$(aws cloudfront sign \
  --url "https://$CF_DOMAIN/premium/index.html" \
  --key-pair-id "$KEY_ID" \
  --private-key file://private_key.pem \
  --date-less-than "$EXPIRY")

echo "$SIGNED"
curl -sS -o /dev/null -w 'signed → %{http_code}\n' "$SIGNED"
curl -sS "$SIGNED"
```

### Step 6 — Custom policy with a wildcard and an IP restriction

```bash
EPOCH_END=$(date -u -d '+2 hours' +%s)

cat > /tmp/policy.json <<EOF
{"Statement":[{"Resource":"https://$CF_DOMAIN/premium/*",
"Condition":{"DateLessThan":{"AWS:EpochTime":$EPOCH_END}}}]}
EOF

POLICY_B64=$(tr -d ' \n' < /tmp/policy.json | openssl base64 -A | tr '+=/' '-_~')
SIG=$(tr -d ' \n' < /tmp/policy.json \
      | openssl sha1 -sign private_key.pem | openssl base64 -A | tr '+=/' '-_~')

URL1="https://$CF_DOMAIN/premium/index.html?Policy=${POLICY_B64}&Signature=${SIG}&Key-Pair-Id=${KEY_ID}"
URL2="https://$CF_DOMAIN/premium/data.json?Policy=${POLICY_B64}&Signature=${SIG}&Key-Pair-Id=${KEY_ID}"

curl -sS -o /dev/null -w 'wildcard doc  → %{http_code}\n' "$URL1"
curl -sS -o /dev/null -w 'wildcard json → %{http_code}\n' "$URL2"
```

**One signature, many objects** — that's the value of a custom policy with a wildcard resource.

### Step 7 — Signed cookies (better for many-file content like HLS)

```bash
COOKIE_POLICY_B64="$POLICY_B64"
COOKIE_SIG="$SIG"

curl -sS -o /dev/null -w 'cookie-auth → %{http_code}\n' \
  -b "CloudFront-Policy=$COOKIE_POLICY_B64" \
  -b "CloudFront-Signature=$COOKIE_SIG" \
  -b "CloudFront-Key-Pair-Id=$KEY_ID" \
  "https://$CF_DOMAIN/premium/index.html"

# Same cookies work for every object under /premium/*
curl -sS -b "CloudFront-Policy=$COOKIE_POLICY_B64" \
        -b "CloudFront-Signature=$COOKIE_SIG" \
        -b "CloudFront-Key-Pair-Id=$KEY_ID" \
        "https://$CF_DOMAIN/premium/data.json"
```

### Step 8 — Prove expiry works

```bash
PAST=$(date -u -d '-1 minute' +%Y-%m-%dT%H:%M:%SZ)
EXPIRED=$(aws cloudfront sign --url "https://$CF_DOMAIN/premium/index.html" \
  --key-pair-id "$KEY_ID" --private-key file://private_key.pem --date-less-than "$PAST")
curl -sS -o /dev/null -w 'expired → %{http_code}\n' "$EXPIRED"   # expect 403
```

### Step 9 — Key rotation with zero downtime

```bash
# Generate a second key
openssl genrsa -out private_key2.pem 2048
openssl rsa -pubout -in private_key2.pem -out public_key2.pem
PUBKEY2=$(cat public_key2.pem)

KEY_ID2=$(aws cloudfront create-public-key --public-key-config "{
  \"CallerReference\": \"lab-pk2-$(date +%s)\",
  \"Name\": \"lab-signing-key-2\",
  \"EncodedKey\": \"$PUBKEY2\"
}" --query 'PublicKey.Id' --output text)

# Put BOTH keys in the group — old signed URLs keep working
KG_ETAG=$(aws cloudfront get-key-group --id "$KG_ID" --query ETag --output text)
aws cloudfront update-key-group --id "$KG_ID" --if-match "$KG_ETAG" \
  --key-group-config "{\"Name\":\"lab-premium-keys\",\"Items\":[\"$KEY_ID2\",\"$KEY_ID\"]}" >/dev/null

# Switch signing to the new key, wait for old URLs to expire, then remove the old key.
```

### What you learned

| | Signed URL | Signed cookie |
|---|---|---|
| Scope | one object | many objects via wildcard |
| Delivery | query string | `Set-Cookie` from your app |
| Best for | download links | streaming, member libraries |

- Never ship the private key to the client. Sign server-side.
- The public key + key group model supports rotation; the legacy root-account key pairs do not.
- Query-string tokens don't fragment the cache unless you add them to the cache key (don't).

---

## Lab 10 — Security: WAF, Geo Restriction & Response Headers

**Objective:** Build the security perimeter — WAF with managed rules and rate limiting, country
restrictions, and a response headers policy.

> WAF costs roughly $5/month per web ACL plus per-rule and per-request charges. Delete it in Lab 15.

### Step 1 — Response headers policy (free, do this first)

```bash
source ~/cloudfront-labs/env.sh

cat > /tmp/rhp.json <<'EOF'
{
  "Name": "lab-secure-headers",
  "Comment": "HSTS, CSP, nosniff, frame deny, strip server headers",
  "SecurityHeadersConfig": {
    "StrictTransportSecurity": {
      "AccessControlMaxAgeSec": 63072000, "IncludeSubdomains": true,
      "Preload": false, "Override": true
    },
    "ContentTypeOptions": { "Override": true },
    "FrameOptions": { "FrameOption": "DENY", "Override": true },
    "ReferrerPolicy": { "ReferrerPolicy": "strict-origin-when-cross-origin", "Override": true },
    "XSSProtection": { "Protection": true, "ModeBlock": true, "Override": true },
    "ContentSecurityPolicy": {
      "ContentSecurityPolicy": "default-src 'self'; img-src 'self' data:; style-src 'self' 'unsafe-inline'",
      "Override": true
    }
  },
  "CustomHeadersConfig": {
    "Quantity": 1,
    "Items": [{ "Header": "X-Delivered-By", "Value": "CloudFront-Lab", "Override": true }]
  },
  "RemoveHeadersConfig": {
    "Quantity": 2,
    "Items": [{ "Header": "Server" }, { "Header": "X-Powered-By" }]
  },
  "ServerTimingHeadersConfig": { "Enabled": true, "SamplingRate": 100.0 }
}
EOF

RHP_ID=$(aws cloudfront create-response-headers-policy \
  --response-headers-policy-config file:///tmp/rhp.json \
  --query 'ResponseHeadersPolicy.Id' --output text)
echo "export RHP_ID=\"$RHP_ID\"" >> ~/cloudfront-labs/env.sh

aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)
jq --arg r "$RHP_ID" '.DistributionConfig
  | .DefaultCacheBehavior.ResponseHeadersPolicyId = $r' /tmp/full.json > /tmp/new.json
aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json >/dev/null
aws cloudfront wait distribution-deployed --id "$DIST_ID"

curl -sSI "https://$CF_DOMAIN/" | grep -iE 'strict-transport|content-security|x-frame|x-content-type|referrer-policy|x-delivered-by|server-timing'
```

`Server-Timing` is worth a look — it exposes CloudFront's internal timing and cache status.

### Step 2 — Create the WAF web ACL (COUNT mode first)

```bash
cat > /tmp/waf-rules.json <<'EOF'
[
  { "Name": "CommonRuleSet", "Priority": 0,
    "OverrideAction": { "Count": {} },
    "Statement": { "ManagedRuleGroupStatement": {
      "VendorName": "AWS", "Name": "AWSManagedRulesCommonRuleSet" } },
    "VisibilityConfig": { "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true, "MetricName": "CommonRuleSet" } },

  { "Name": "KnownBadInputs", "Priority": 1,
    "OverrideAction": { "Count": {} },
    "Statement": { "ManagedRuleGroupStatement": {
      "VendorName": "AWS", "Name": "AWSManagedRulesKnownBadInputsRuleSet" } },
    "VisibilityConfig": { "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true, "MetricName": "KnownBadInputs" } },

  { "Name": "IpReputation", "Priority": 2,
    "OverrideAction": { "Count": {} },
    "Statement": { "ManagedRuleGroupStatement": {
      "VendorName": "AWS", "Name": "AWSManagedRulesAmazonIpReputationList" } },
    "VisibilityConfig": { "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true, "MetricName": "IpReputation" } },

  { "Name": "RateLimitPerIP", "Priority": 3,
    "Action": { "Block": {} },
    "Statement": { "RateBasedStatement": { "Limit": 2000, "AggregateKeyType": "IP" } },
    "VisibilityConfig": { "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true, "MetricName": "RateLimit" } },

  { "Name": "BlockAdminPathFromOutside", "Priority": 4,
    "Action": { "Block": {} },
    "Statement": { "AndStatement": { "Statements": [
      { "ByteMatchStatement": {
          "SearchString": "/admin",
          "FieldToMatch": { "UriPath": {} },
          "TextTransformations": [{ "Priority": 0, "Type": "LOWERCASE" }],
          "PositionalConstraint": "STARTS_WITH" } },
      { "NotStatement": { "Statement": { "GeoMatchStatement": {
          "CountryCodes": ["IN", "US"] } } } }
    ] } },
    "VisibilityConfig": { "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true, "MetricName": "AdminGeoBlock" } }
]
EOF

WAF_ID=$(aws wafv2 create-web-acl --region us-east-1 \
  --name cf-lab-protection --scope CLOUDFRONT \
  --default-action Allow={} \
  --rules file:///tmp/waf-rules.json \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=cfLabProtection \
  --query 'Summary.Id' --output text)

WAF_ARN=$(aws wafv2 list-web-acls --region us-east-1 --scope CLOUDFRONT \
  --query "WebACLs[?Name=='cf-lab-protection'].ARN" --output text)

echo "export WAF_ID=\"$WAF_ID\""   >> ~/cloudfront-labs/env.sh
echo "export WAF_ARN=\"$WAF_ARN\"" >> ~/cloudfront-labs/env.sh
echo "$WAF_ARN"
```

### Step 3 — Attach it to the distribution

```bash
source ~/cloudfront-labs/env.sh
aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)
jq --arg w "$WAF_ARN" '.DistributionConfig | .WebACLId = $w' /tmp/full.json > /tmp/new.json
aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json >/dev/null
aws cloudfront wait distribution-deployed --id "$DIST_ID"
```

### Step 4 — Trigger a rule and inspect

```bash
# A classic SQLi-looking request — CommonRuleSet is in COUNT mode so it won't be blocked yet
curl -sS -o /dev/null -w '%{http_code}\n' \
  "https://$CF_DOMAIN/index.html?q=1%27%20OR%20%271%27=%271"

sleep 60

# See what WAF sampled
aws wafv2 get-sampled-requests --region us-east-1 \
  --web-acl-arn "$WAF_ARN" --rule-metric-name CommonRuleSet --scope CLOUDFRONT \
  --time-window StartTime=$(( $(date +%s) - 900 )),EndTime=$(date +%s) \
  --max-items 10 \
  --query 'SampledRequests[].{URI:Request.URI,Action:Action,Rule:RuleNameWithinRuleGroup}' \
  --output table
```

**This is the workflow that matters:** run in COUNT, review real traffic, then promote to BLOCK.
Going straight to BLOCK with managed rules is how people take down their own login page.

### Step 5 — Promote a rule to BLOCK

```bash
LOCK=$(aws wafv2 get-web-acl --region us-east-1 --scope CLOUDFRONT \
  --name cf-lab-protection --id "$WAF_ID" --query LockToken --output text)

jq '(.[] | select(.Name=="KnownBadInputs") | .OverrideAction) = {"None":{}}' \
  /tmp/waf-rules.json > /tmp/waf-rules-block.json

aws wafv2 update-web-acl --region us-east-1 --scope CLOUDFRONT \
  --name cf-lab-protection --id "$WAF_ID" --lock-token "$LOCK" \
  --default-action Allow={} \
  --rules file:///tmp/waf-rules-block.json \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=cfLabProtection >/dev/null
```

### Step 6 — Geo restriction

```bash
source ~/cloudfront-labs/env.sh
aws cloudfront get-distribution-config --id "$DIST_ID" > /tmp/full.json
ETAG=$(jq -r '.ETag' /tmp/full.json)

# Block a couple of countries (blacklist mode)
jq '.DistributionConfig | .Restrictions.GeoRestriction = {
      "RestrictionType": "blacklist",
      "Quantity": 2,
      "Items": ["CU","KP"]
    }' /tmp/full.json > /tmp/new.json

aws cloudfront update-distribution --id "$DIST_ID" --if-match "$ETAG" \
  --distribution-config file:///tmp/new.json >/dev/null
aws cloudfront wait distribution-deployed --id "$DIST_ID"

aws cloudfront get-distribution-config --id "$DIST_ID" \
  --query 'DistributionConfig.Restrictions.GeoRestriction' --output json
```

To switch to an allow-list:

```bash
cf_update "$DIST_ID" '.Restrictions.GeoRestriction = {
  "RestrictionType": "whitelist", "Quantity": 3, "Items": ["IN","US","GB"] }'
```

**Remember:** geo restriction is a licensing/compliance tool, not security — a VPN defeats it. Use
WAF geo match rules when you need per-path or combinable logic.

### Step 7 — Security scorecard

```bash
echo "=== TLS ==="
aws cloudfront get-distribution-config --id "$DIST_ID" \
  --query 'DistributionConfig.ViewerCertificate.MinimumProtocolVersion' --output text

echo "=== Viewer protocol ==="
aws cloudfront get-distribution-config --id "$DIST_ID" \
  --query 'DistributionConfig.DefaultCacheBehavior.ViewerProtocolPolicy' --output text

echo "=== WAF ==="
aws cloudfront get-distribution-config --id "$DIST_ID" \
  --query 'DistributionConfig.WebACLId' --output text

echo "=== Bucket is private ==="
curl -sS -o /dev/null -w '%{http_code} (want 403)\n' \
  "https://$BUCKET.s3.$AWS_REGION.amazonaws.com/index.html"

echo "=== Security headers ==="
curl -sSI "https://$CF_DOMAIN/" | grep -icE 'strict-transport|x-content-type|x-frame|content-security'
```

### What you learned

- Response headers policies give you most of a security header audit for free, no code.
- WAF for CloudFront is `--scope CLOUDFRONT --region us-east-1`. Always.
- COUNT → review → BLOCK is not optional advice; it's the only safe rollout.
- Geo restriction and WAF geo rules solve different problems.

---

## Lab 11 — Logging: Standard Logs, Athena & Real-Time

**Objective:** Get visibility. Standard logs into S3, an Athena table over them, the three queries
that matter, plus CloudWatch metrics and alarms.

### Step 1 — Log bucket

```bash
source ~/cloudfront-labs/env.sh

aws s3api create-bucket --bucket "$LOG_BUCKET" --region "$AWS_REGION" \
  --create-bucket-configuration LocationConstraint="$AWS_REGION" 2>/dev/null || true

# Legacy (v1) logging requires bucket ACLs; v2 logging does not
aws s3api put-bucket-ownership-controls --bucket "$LOG_BUCKET" \
  --ownership-controls 'Rules=[{ObjectOwnership=BucketOwnerPreferred}]'

# Lifecycle: expire logs after 90 days so this doesn't grow forever
aws s3api put-bucket-lifecycle-configuration --bucket "$LOG_BUCKET" \
  --lifecycle-configuration '{"Rules":[{"ID":"expire-logs","Status":"Enabled",
    "Filter":{"Prefix":"cloudfront/"},"Expiration":{"Days":90}}]}'
```

### Step 2 — Enable standard logging (v2, via CloudWatch Logs delivery)

```bash
aws logs put-delivery-source --region us-east-1 \
  --name "cf-src-$DIST_ID" \
  --resource-arn "arn:aws:cloudfront::$ACCOUNT_ID:distribution/$DIST_ID" \
  --log-type ACCESS_LOGS

DEST_ARN=$(aws logs put-delivery-destination --region us-east-1 \
  --name "cf-dest-s3" \
  --delivery-destination-configuration "destinationResourceArn=arn:aws:s3:::$LOG_BUCKET" \
  --output-format plain \
  --query 'deliveryDestination.arn' --output text)

aws logs create-delivery --region us-east-1 \
  --delivery-source-name "cf-src-$DIST_ID" \
  --delivery-destination-arn "$DEST_ARN"

aws logs describe-deliveries --region us-east-1 \
  --query 'deliveries[].{Source:deliverySourceName,Dest:deliveryDestinationArn}' --output table
```

**Simpler alternative (legacy v1) if the above is fiddly in your account:**

```bash
cf_update "$DIST_ID" "$(cat <<EOF
.Logging = { "Enabled": true, "IncludeCookies": false,
             "Bucket": "$LOG_BUCKET.s3.amazonaws.com", "Prefix": "cloudfront/" }
EOF
)"
```

### Step 3 — Generate traffic

```bash
for i in $(seq 1 60); do
  curl -sS -o /dev/null "https://$CF_DOMAIN/"
  curl -sS -o /dev/null "https://$CF_DOMAIN/assets/style.css"
  curl -sS -o /dev/null "https://$CF_DOMAIN/does-not-exist-$i"
  curl -sS -o /dev/null "https://$CF_DOMAIN/product.html?id=$((RANDOM % 5))"
done
echo "traffic generated — logs appear within a few minutes"
```

### Step 4 — Athena table

```bash
aws s3 mb "s3://athena-results-$ACCOUNT_ID" --region "$AWS_REGION" 2>/dev/null || true

cat > /tmp/create-table.sql <<EOF
CREATE DATABASE IF NOT EXISTS cflabs;

CREATE EXTERNAL TABLE IF NOT EXISTS cflabs.cf_logs (
  \`date\` DATE, time STRING, x_edge_location STRING, sc_bytes BIGINT,
  c_ip STRING, cs_method STRING, cs_host STRING, cs_uri_stem STRING,
  sc_status INT, cs_referer STRING, cs_user_agent STRING, cs_uri_query STRING,
  cs_cookie STRING, x_edge_result_type STRING, x_edge_request_id STRING,
  x_host_header STRING, cs_protocol STRING, cs_bytes BIGINT, time_taken FLOAT,
  x_forwarded_for STRING, ssl_protocol STRING, ssl_cipher STRING,
  x_edge_response_result_type STRING, cs_protocol_version STRING, fle_status STRING,
  fle_encrypted_fields INT, c_port INT, time_to_first_byte FLOAT,
  x_edge_detailed_result_type STRING, sc_content_type STRING, sc_content_len BIGINT,
  sc_range_start BIGINT, sc_range_end BIGINT
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
LOCATION 's3://$LOG_BUCKET/cloudfront/'
TBLPROPERTIES ('skip.header.line.count'='2');
EOF

aws athena start-query-execution \
  --query-string "$(cat /tmp/create-table.sql)" \
  --result-configuration "OutputLocation=s3://athena-results-$ACCOUNT_ID/"
```

### Step 5 — The three queries you'll actually run

```bash
runq() {
  QID=$(aws athena start-query-execution --query-string "$1" \
    --query-execution-context Database=cflabs \
    --result-configuration "OutputLocation=s3://athena-results-$ACCOUNT_ID/" \
    --query QueryExecutionId --output text)
  while [ "$(aws athena get-query-execution --query-execution-id "$QID" \
    --query 'QueryExecution.Status.State' --output text)" = "RUNNING" ]; do sleep 2; done
  aws athena get-query-results --query-execution-id "$QID" \
    --query 'ResultSet.Rows[].Data[].VarCharValue' --output table
}

# 1. Cache hit ratio
runq "SELECT x_edge_result_type, COUNT(*) AS n,
             ROUND(100.0*COUNT(*)/SUM(COUNT(*)) OVER (), 2) AS pct
      FROM cf_logs GROUP BY x_edge_result_type ORDER BY n DESC"

# 2. Worst cache offenders — your optimization worklist
runq "SELECT cs_uri_stem, COUNT(*) AS misses
      FROM cf_logs WHERE x_edge_result_type='Miss'
      GROUP BY cs_uri_stem ORDER BY misses DESC LIMIT 20"

# 3. Slowest paths
runq "SELECT cs_uri_stem, ROUND(AVG(time_taken),3) AS avg_s, COUNT(*) AS n
      FROM cf_logs GROUP BY cs_uri_stem HAVING COUNT(*) > 5
      ORDER BY avg_s DESC LIMIT 20"
```

More useful queries to keep around:

```sql
-- Error breakdown
SELECT sc_status, COUNT(*) FROM cf_logs WHERE sc_status >= 400 GROUP BY sc_status ORDER BY 2 DESC;

-- Traffic by edge location (where are your users, really?)
SELECT substr(x_edge_location, 1, 3) AS pop, COUNT(*) AS n, SUM(sc_bytes)/1048576 AS mb
FROM cf_logs GROUP BY 1 ORDER BY n DESC LIMIT 20;

-- Bandwidth hogs
SELECT cs_uri_stem, SUM(sc_bytes)/1048576 AS mb, COUNT(*) AS hits
FROM cf_logs GROUP BY 1 ORDER BY mb DESC LIMIT 20;

-- Top user agents (bot detection starting point)
SELECT cs_user_agent, COUNT(*) FROM cf_logs GROUP BY 1 ORDER BY 2 DESC LIMIT 20;

-- Requests that failed over or errored at the edge
SELECT x_edge_detailed_result_type, COUNT(*) FROM cf_logs
WHERE x_edge_result_type = 'Error' GROUP BY 1 ORDER BY 2 DESC;
```

### Step 6 — Enable additional CloudWatch metrics + alarms

```bash
aws cloudfront create-monitoring-subscription --distribution-id "$DIST_ID" \
  --monitoring-subscription 'RealtimeMetricsSubscriptionConfig={RealtimeMetricsSubscriptionStatus=Enabled}'

# 5xx alarm
aws cloudwatch put-metric-alarm --region us-east-1 \
  --alarm-name "CF-Lab-5xx-$DIST_ID" \
  --namespace AWS/CloudFront --metric-name 5xxErrorRate \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --statistic Average --period 300 --evaluation-periods 2 \
  --threshold 1 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching

# Cache hit rate alarm
aws cloudwatch put-metric-alarm --region us-east-1 \
  --alarm-name "CF-Lab-LowHitRate-$DIST_ID" \
  --namespace AWS/CloudFront --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --statistic Average --period 3600 --evaluation-periods 3 \
  --threshold 80 --comparison-operator LessThanThreshold \
  --treat-missing-data notBreaching

aws cloudwatch describe-alarms --region us-east-1 \
  --alarm-name-prefix "CF-Lab-" \
  --query 'MetricAlarms[].{Name:AlarmName,State:StateValue}' --output table
```

### Step 7 — Real-time logs (optional; costs money)

```bash
aws kinesis create-stream --stream-name cf-lab-realtime --shard-count 1
aws kinesis wait stream-exists --stream-name cf-lab-realtime
STREAM_ARN=$(aws kinesis describe-stream --stream-name cf-lab-realtime \
  --query 'StreamDescription.StreamARN' --output text)

cat > /tmp/kinesis-trust.json <<'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"cloudfront.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
aws iam create-role --role-name cf-lab-rtlogs-role \
  --assume-role-policy-document file:///tmp/kinesis-trust.json >/dev/null 2>&1 || true
aws iam put-role-policy --role-name cf-lab-rtlogs-role --policy-name kinesis-put \
  --policy-document "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",
    \"Action\":[\"kinesis:PutRecord\",\"kinesis:PutRecords\",\"kinesis:DescribeStreamSummary\"],
    \"Resource\":\"$STREAM_ARN\"}]}"
sleep 10

aws cloudfront create-realtime-log-config \
  --name cf-lab-realtime --sampling-rate 100 \
  --end-points "StreamType=Kinesis,KinesisStreamConfig={RoleARN=arn:aws:iam::$ACCOUNT_ID:role/cf-lab-rtlogs-role,StreamARN=$STREAM_ARN}" \
  --fields timestamp c-ip sc-status cs-method cs-uri-stem x-edge-result-type time-taken c-country
```

Read records straight off the stream:

```bash
SHARD=$(aws kinesis list-shards --stream-name cf-lab-realtime \
  --query 'Shards[0].ShardId' --output text)
ITER=$(aws kinesis get-shard-iterator --stream-name cf-lab-realtime \
  --shard-id "$SHARD" --shard-iterator-type LATEST --query ShardIterator --output text)

curl -sS -o /dev/null "https://$CF_DOMAIN/?rt=$RANDOM"
sleep 5
aws kinesis get-records --shard-iterator "$ITER" \
  --query 'Records[].Data' --output text | while read -r d; do echo "$d" | base64 -d; done
```

### What you learned

- Standard logs are minutes-delayed and cheap; real-time logs are seconds-delayed and metered.
- `x_edge_result_type` is the field that answers "is my CDN actually working?"
- `CacheHitRate` and `OriginLatency` require the paid additional-metrics subscription — worth it.
- CloudFront metrics live in `us-east-1` regardless of where anything else is.

---

## Lab 12 — Origin Shield, Compression & Price Class

**Objective:** Tune the last set of dials — the third cache tier, compression correctness, and cost
vs reach.

### Step 1 — Enable Origin Shield (in the origin's region, not the users')

```bash
source ~/cloudfront-labs/env.sh

cf_update "$DIST_ID" "$(cat <<EOF
.Origins.Items = [ .Origins.Items[] |
   if .Id == "s3-origin"
   then .OriginShield = { "Enabled": true, "OriginShieldRegion": "$AWS_REGION" }
   else . end ]
EOF
)"

aws cloudfront wait distribution-deployed --id "$DIST_ID"
aws cloudfront get-distribution-config --id "$DIST_ID" \
  --query 'DistributionConfig.Origins.Items[].{Id:Id,Shield:OriginShield}' --output json
```

Watch for `OriginShieldHit` appearing in your logs:

```sql
SELECT x_edge_detailed_result_type, COUNT(*) FROM cf_logs GROUP BY 1 ORDER BY 2 DESC;
```

**When to keep it:** origin is expensive or fragile, content is long-tail, you have global traffic.
**When to turn it off:** hit ratio is already 95%+, or content is uncacheable (you'd pay for a hop
that never serves a hit).

### Step 2 — Diagnose compression properly

```bash
# Text asset — should compress
curl -sSI -H 'Accept-Encoding: br,gzip' "https://$CF_DOMAIN/assets/style.css" \
  | grep -iE 'content-encoding|content-type|content-length'

# A file under 1000 bytes will NOT compress — this is by design
echo "tiny" > /tmp/tiny.txt
aws s3 cp /tmp/tiny.txt "s3://$BUCKET/tiny.txt" --content-type "text/plain"
curl -sSI -H 'Accept-Encoding: gzip' "https://$CF_DOMAIN/tiny.txt" | grep -i content-encoding
# → no content-encoding header. Correct behaviour.

# A file with the WRONG content type will NOT compress — the classic bug
aws s3 cp /tmp/product.html "s3://$BUCKET/wrongtype.html" \
  --content-type "binary/octet-stream" --metadata-directive REPLACE
curl -sSI -H 'Accept-Encoding: gzip' "https://$CF_DOMAIN/wrongtype.html" \
  | grep -iE 'content-type|content-encoding'
# → no compression, because binary/octet-stream isn't on the compressible list

# Fix it
aws s3 cp "s3://$BUCKET/wrongtype.html" "s3://$BUCKET/wrongtype.html" \
  --content-type "text/html; charset=utf-8" --metadata-directive REPLACE
aws cloudfront create-invalidation --distribution-id "$DIST_ID" --paths "/wrongtype.html" >/dev/null
sleep 30
curl -sSI -H 'Accept-Encoding: gzip' "https://$CF_DOMAIN/wrongtype.html" | grep -i content-encoding
```

### The compression checklist

```
□ Compress = true on the cache behavior
□ Cache policy has EnableAcceptEncodingGzip / Brotli = true
□ Viewer sends Accept-Encoding
□ Content-Type is on the compressible list (text/*, application/json, javascript, xml, svg+xml)
□ Size is between 1,000 and 10,000,000 bytes
□ Origin did NOT already set Content-Encoding
□ Content-Length header is present
```

### Step 3 — Measure the saving

```bash
UNCOMP=$(curl -sS -o /dev/null -w '%{size_download}' \
  -H 'Accept-Encoding: identity' "https://$CF_DOMAIN/assets/style.css")
COMP=$(curl -sS -o /dev/null -w '%{size_download}' \
  -H 'Accept-Encoding: br,gzip' "https://$CF_DOMAIN/assets/style.css")
echo "uncompressed=$UNCOMP  compressed=$COMP"
```

### Step 4 — Price class

```bash
cf_update "$DIST_ID" '.PriceClass = "PriceClass_200"'
aws cloudfront wait distribution-deployed --id "$DIST_ID"

aws cloudfront get-distribution-config --id "$DIST_ID" \
  --query 'DistributionConfig.PriceClass' --output text

# Compare which PoP serves you before and after
curl -sSI "https://$CF_DOMAIN/" | grep -i x-amz-cf-pop

# Put it back for the rest of the labs
cf_update "$DIST_ID" '.PriceClass = "PriceClass_All"'
```

**Use your own log data to decide:**

```sql
SELECT substr(x_edge_location,1,3) AS pop, COUNT(*) AS requests
FROM cf_logs GROUP BY 1 ORDER BY 2 DESC;
```

If nothing meaningful comes from South America or Oceania, PriceClass_200 costs you nothing in
practice.

### Step 5 — HTTP/3 and IPv6

```bash
cf_update "$DIST_ID" '.HttpVersion = "http2and3" | .IsIPV6Enabled = true'
aws cloudfront wait distribution-deployed --id "$DIST_ID"

curl -sSI --http2 "https://$CF_DOMAIN/" | head -1
curl -sSI --http3 "https://$CF_DOMAIN/" | head -1 2>/dev/null || echo "curl lacks HTTP/3 support — fine"
dig +short AAAA "$CF_DOMAIN"
```

### What you learned

- Origin Shield is a cost/benefit decision, not a default.
- Compression failures are almost always Content-Type or size, not configuration.
- Price class is free performance/cost tuning that most people never touch.

---

## Lab 13 — Continuous Deployment (Staging Distribution)

**Objective:** Change a distribution's configuration and validate it against a slice of real traffic
before promoting it.

### Step 1 — Create a staging distribution

```bash
source ~/cloudfront-labs/env.sh

STAGING_JSON=$(aws cloudfront copy-distribution \
  --primary-distribution-id "$DIST_ID" \
  --staging \
  --caller-reference "staging-$(date +%s)")

STAGING_ID=$(echo "$STAGING_JSON" | jq -r '.Distribution.Id')
STAGING_DOMAIN=$(echo "$STAGING_JSON" | jq -r '.Distribution.DomainName')

echo "export STAGING_ID=\"$STAGING_ID\""         >> ~/cloudfront-labs/env.sh
echo "export STAGING_DOMAIN=\"$STAGING_DOMAIN\"" >> ~/cloudfront-labs/env.sh
echo "Staging: $STAGING_ID / $STAGING_DOMAIN"

aws cloudfront wait distribution-deployed --id "$STAGING_ID"
```

> **If your primary uses OAC to S3, the staging distribution needs bucket access too.**

```bash
cat > /tmp/bp-staging.json <<EOF
{ "Version": "2012-10-17", "Statement": [{
    "Sid": "AllowCloudFrontPrimaryAndStaging",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/*",
    "Condition": { "ArnLike": { "AWS:SourceArn": [
      "arn:aws:cloudfront::$ACCOUNT_ID:distribution/$DIST_ID",
      "arn:aws:cloudfront::$ACCOUNT_ID:distribution/$STAGING_ID" ]}}}]}
EOF
aws s3api put-bucket-policy --bucket "$BUCKET" --policy file:///tmp/bp-staging.json
```

### Step 2 — Make a change on staging only

```bash
# Test a shorter TTL policy on staging
cat > /tmp/cp-staging.json <<'EOF'
{
  "Name": "LabStagingShortTTL",
  "DefaultTTL": 30, "MinTTL": 0, "MaxTTL": 300,
  "ParametersInCacheKeyAndForwardedToOrigin": {
    "EnableAcceptEncodingGzip": true, "EnableAcceptEncodingBrotli": true,
    "HeadersConfig": { "HeaderBehavior": "none" },
    "CookiesConfig": { "CookieBehavior": "none" },
    "QueryStringsConfig": { "QueryStringBehavior": "none" }
  }
}
EOF
CP_STAGING=$(aws cloudfront create-cache-policy \
  --cache-policy-config file:///tmp/cp-staging.json --query 'CachePolicy.Id' --output text)

cf_update "$STAGING_ID" ".DefaultCacheBehavior.CachePolicyId = \"$CP_STAGING\""
aws cloudfront wait distribution-deployed --id "$STAGING_ID"
```

### Step 3 — Header-based traffic config (safest — you control who hits staging)

```bash
CDP_ID=$(aws cloudfront create-continuous-deployment-policy \
  --continuous-deployment-policy-config "{
    \"StagingDistributionDnsNames\": { \"Quantity\": 1, \"Items\": [\"$STAGING_DOMAIN\"] },
    \"Enabled\": true,
    \"TrafficConfig\": {
      \"Type\": \"SingleHeader\",
      \"SingleHeaderConfig\": { \"Header\": \"aws-cf-cd-canary\", \"Value\": \"true\" }
    }
  }" --query 'ContinuousDeploymentPolicy.Id' --output text)

echo "export CDP_ID=\"$CDP_ID\"" >> ~/cloudfront-labs/env.sh

cf_update "$DIST_ID" ".ContinuousDeploymentPolicyId = \"$CDP_ID\""
aws cloudfront wait distribution-deployed --id "$DIST_ID"
```

### Step 4 — Test both paths

```bash
echo "--- production (no header) ---"
curl -sSI "https://$CF_DOMAIN/" | grep -iE 'x-cache|age'

echo "--- staging (canary header) ---"
curl -sSI -H "aws-cf-cd-canary: true" "https://$CF_DOMAIN/" | grep -iE 'x-cache|age'
```

Same URL, two configurations. Your QA team gets the new behaviour; everyone else doesn't.

### Step 5 — Switch to a weight-based rollout

```bash
CDP_ETAG=$(aws cloudfront get-continuous-deployment-policy --id "$CDP_ID" \
  --query ETag --output text)

aws cloudfront update-continuous-deployment-policy --id "$CDP_ID" --if-match "$CDP_ETAG" \
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
```

**Constraints to remember:** max weight is **0.15 (15%)**, session stickiness idle TTL is 300–3600
seconds, and staging distributions can't have their own alternate domain names.

### Step 6 — Compare metrics, then promote

```bash
for M in Requests 5xxErrorRate CacheHitRate; do
  for D in "$DIST_ID:PRIMARY" "$STAGING_ID:STAGING"; do
    ID="${D%%:*}"; LABEL="${D##*:}"
    V=$(aws cloudwatch get-metric-statistics --region us-east-1 \
      --namespace AWS/CloudFront --metric-name "$M" \
      --dimensions Name=DistributionId,Value="$ID" Name=Region,Value=Global \
      --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
      --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
      --period 3600 --statistics Average \
      --query 'Datapoints[0].Average' --output text 2>/dev/null)
    echo "$M $LABEL: ${V:-n/a}"
  done
done
```

```bash
# Promote staging config onto production
ETAG=$(aws cloudfront get-distribution-config --id "$DIST_ID" --query ETag --output text)
aws cloudfront update-distribution-with-staging-config \
  --id "$DIST_ID" --staging-distribution-id "$STAGING_ID" --if-match "$ETAG" \
  --query 'Distribution.Status' --output text
aws cloudfront wait distribution-deployed --id "$DIST_ID"

# Confirm production now uses the staging cache policy
aws cloudfront get-distribution-config --id "$DIST_ID" \
  --query 'DistributionConfig.DefaultCacheBehavior.CachePolicyId' --output text
```

**Rollback** is simply: don't promote. Disable the continuous deployment policy and 100% of traffic
returns to the current production config.

```bash
cf_update "$DIST_ID" '.ContinuousDeploymentPolicyId = ""'
```

### What you learned

- Continuous deployment tests **CDN configuration**, not application code. Use weighted target
  groups or feature flags for app changes.
- Header-based targeting is the safer first step; weight-based is for real canaries.
- The staging distribution needs its own permissions — the OAC bucket policy is the usual trap.

---

## Lab 14 — Everything as Code with Terraform

**Objective:** Rebuild the whole stack declaratively so it's reviewable, repeatable, and destroyable
in one command.

### Step 1 — Project layout

```bash
mkdir -p ~/cloudfront-labs/terraform/functions && cd ~/cloudfront-labs/terraform
```

```bash
cat > versions.tf <<'EOF'
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws    = { source = "hashicorp/aws", version = "~> 5.0" }
    random = { source = "hashicorp/random", version = "~> 3.5" }
  }
}

provider "aws" {
  region = var.region
}

# CloudFront-adjacent resources MUST live here
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}
EOF

cat > variables.tf <<'EOF'
variable "region"      { type = string, default = "ap-south-1" }
variable "project"     { type = string, default = "cf-lab-tf" }
variable "domain_name" { type = string, default = "" }   # leave empty to skip custom domain
variable "price_class" { type = string, default = "PriceClass_All" }
variable "tags" {
  type = map(string)
  default = { Project = "cloudfront-lab", ManagedBy = "terraform" }
}
EOF
```

### Step 2 — S3 origin

```bash
cat > s3.tf <<'EOF'
resource "random_id" "suffix" { byte_length = 4 }

resource "aws_s3_bucket" "site" {
  bucket        = "${var.project}-${random_id.suffix.hex}"
  force_destroy = true
  tags          = var.tags
}

resource "aws_s3_bucket_public_access_block" "site" {
  bucket                  = aws_s3_bucket.site.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "site" {
  bucket = aws_s3_bucket.site.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "site" {
  bucket = aws_s3_bucket.site.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
  }
}

resource "aws_s3_object" "index" {
  bucket        = aws_s3_bucket.site.id
  key           = "index.html"
  content       = "<!DOCTYPE html><html><body style=\"font-family:system-ui;padding:3rem\"><h1>Deployed by Terraform</h1><p>CloudFront + S3 + OAC</p></body></html>"
  content_type  = "text/html; charset=utf-8"
  cache_control = "no-cache"
  etag          = md5("terraform-index-v1")
}

resource "aws_s3_bucket_policy" "oac" {
  bucket = aws_s3_bucket.site.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowCloudFrontServicePrincipalReadOnly"
      Effect    = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.site.arn}/*"
      Condition = {
        StringEquals = { "AWS:SourceArn" = aws_cloudfront_distribution.site.arn }
      }
    }]
  })
}
EOF
```

### Step 3 — CloudFront

```bash
cat > functions/uri-rewrite.js <<'EOF'
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    if (uri.endsWith('/')) { request.uri = uri + 'index.html'; }
    else if (!uri.includes('.')) { request.uri = uri + '/index.html'; }
    return request;
}
EOF

cat > cloudfront.tf <<'EOF'
data "aws_cloudfront_cache_policy" "optimized" { name = "Managed-CachingOptimized" }
data "aws_cloudfront_cache_policy" "disabled"  { name = "Managed-CachingDisabled" }
data "aws_cloudfront_origin_request_policy" "cors_s3" { name = "Managed-CORS-S3Origin" }
data "aws_cloudfront_response_headers_policy" "security" {
  name = "Managed-SecurityHeadersPolicy"
}

resource "aws_cloudfront_origin_access_control" "s3" {
  name                              = "${var.project}-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_function" "rewrite" {
  name    = "${var.project}-uri-rewrite"
  runtime = "cloudfront-js-2.0"
  comment = "Directory index rewriting"
  publish = true
  code    = file("${path.module}/functions/uri-rewrite.js")
}

resource "aws_cloudfront_distribution" "site" {
  enabled             = true
  is_ipv6_enabled     = true
  http_version        = "http2and3"
  comment             = "${var.project} — managed by Terraform"
  default_root_object = "index.html"
  price_class         = var.price_class
  aliases             = var.domain_name == "" ? [] : [var.domain_name]
  tags                = var.tags

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
    origin_request_policy_id   = data.aws_cloudfront_origin_request_policy.cors_s3.id
    response_headers_policy_id = data.aws_cloudfront_response_headers_policy.security.id

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.rewrite.arn
    }
  }

  custom_error_response {
    error_code            = 403
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 10
  }

  custom_error_response {
    error_code            = 404
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 10
  }

  viewer_certificate {
    cloudfront_default_certificate = var.domain_name == "" ? true : false
    acm_certificate_arn            = var.domain_name == "" ? null : aws_acm_certificate.cert[0].arn
    ssl_support_method             = var.domain_name == "" ? null : "sni-only"
    minimum_protocol_version       = var.domain_name == "" ? null : "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }
}

resource "aws_acm_certificate" "cert" {
  count             = var.domain_name == "" ? 0 : 1
  provider          = aws.us_east_1
  domain_name       = var.domain_name
  validation_method = "DNS"
  lifecycle { create_before_destroy = true }
}
EOF

cat > outputs.tf <<'EOF'
output "distribution_id"     { value = aws_cloudfront_distribution.site.id }
output "distribution_domain" { value = aws_cloudfront_distribution.site.domain_name }
output "distribution_arn"    { value = aws_cloudfront_distribution.site.arn }
output "bucket_name"         { value = aws_s3_bucket.site.id }
output "site_url"            { value = "https://${aws_cloudfront_distribution.site.domain_name}" }
EOF
```

### Step 4 — Apply

```bash
cd ~/cloudfront-labs/terraform
terraform init
terraform fmt
terraform validate
terraform plan -out=tfplan
terraform apply tfplan

terraform output
TF_DOMAIN=$(terraform output -raw distribution_domain)
```

### Step 5 — Verify and iterate

```bash
curl -sSI "https://$TF_DOMAIN/" | grep -iE 'HTTP/|x-cache|strict-transport|x-frame'
curl -sS  "https://$TF_DOMAIN/"

# Change something and see a real plan diff
sed -i 's/PriceClass_All/PriceClass_200/' variables.tf
terraform plan | grep -A3 price_class
```

### The dependency detail worth noticing

`aws_s3_bucket_policy.oac` references `aws_cloudfront_distribution.site.arn`, and the distribution
references the bucket. Terraform resolves this correctly because the *policy* depends on the
distribution, not the other way round. This is exactly the ordering you had to do manually in Lab 1
(create distribution → then apply bucket policy).

### Step 6 — Destroy

```bash
terraform destroy -auto-approve
```

Terraform handles the disable→wait→delete dance for the distribution automatically, which is one of
the best reasons to use it.

### What you learned

- The `us_east_1` provider alias is mandatory for ACM/WAF/Lambda@Edge.
- `data.aws_cloudfront_cache_policy` lets you reference managed policies by name instead of
  hardcoding IDs that may differ.
- IaC makes the teardown a single command — which is the difference between a $0 lab and a
  surprise bill.

---

## Lab 15 — Complete Teardown

**Objective:** Delete everything. Run this even if you plan to come back — you can rebuild in 20
minutes from these labs.

> **Order matters.** CloudFront refuses to delete resources that are still referenced.

### Step 0 — Inventory what you created

```bash
source ~/cloudfront-labs/env.sh
cat ~/cloudfront-labs/env.sh
```

### Step 1 — Detach everything from the distributions

```bash
detach_all() {
  local D="$1"
  echo "Detaching from $D"
  aws cloudfront get-distribution-config --id "$D" > /tmp/t.json
  local E=$(jq -r '.ETag' /tmp/t.json)
  jq '.DistributionConfig
      | .WebACLId = ""
      | .ContinuousDeploymentPolicyId = ""
      | .DefaultCacheBehavior.FunctionAssociations = {"Quantity":0,"Items":[]}
      | .DefaultCacheBehavior.LambdaFunctionAssociations = {"Quantity":0,"Items":[]}
      | .DefaultCacheBehavior.TrustedKeyGroups = {"Enabled":false,"Quantity":0}
      | .DefaultCacheBehavior.RealtimeLogConfigArn = null
      | .CacheBehaviors = {"Quantity":0,"Items":[]}
      | .Logging = {"Enabled":false,"IncludeCookies":false,"Bucket":"","Prefix":""}' \
    /tmp/t.json > /tmp/t2.json
  aws cloudfront update-distribution --id "$D" --if-match "$E" \
    --distribution-config file:///tmp/t2.json --query 'Distribution.Status' --output text
  aws cloudfront wait distribution-deployed --id "$D"
}

[ -n "${STAGING_ID:-}" ] && detach_all "$STAGING_ID"
detach_all "$DIST_ID"
```

### Step 2 — Disable and delete the distributions

```bash
kill_dist() {
  local D="$1"
  echo "Disabling $D"
  aws cloudfront get-distribution-config --id "$D" > /tmp/k.json
  local E=$(jq -r '.ETag' /tmp/k.json)
  jq '.DistributionConfig | .Enabled = false' /tmp/k.json > /tmp/k2.json
  aws cloudfront update-distribution --id "$D" --if-match "$E" \
    --distribution-config file:///tmp/k2.json >/dev/null
  echo "Waiting for Deployed (this takes a few minutes)..."
  aws cloudfront wait distribution-deployed --id "$D"
  local E2=$(aws cloudfront get-distribution-config --id "$D" --query ETag --output text)
  aws cloudfront delete-distribution --id "$D" --if-match "$E2"
  echo "Deleted $D"
}

[ -n "${STAGING_ID:-}" ] && kill_dist "$STAGING_ID"
kill_dist "$DIST_ID"
```

### Step 3 — Continuous deployment policy

```bash
if [ -n "${CDP_ID:-}" ]; then
  E=$(aws cloudfront get-continuous-deployment-policy --id "$CDP_ID" --query ETag --output text)
  aws cloudfront delete-continuous-deployment-policy --id "$CDP_ID" --if-match "$E"
fi
```

### Step 4 — Functions, KVS, and Lambda@Edge

```bash
# CloudFront Functions
for F in lab-uri-rewrite lab-redirects; do
  E=$(aws cloudfront describe-function --name "$F" --query ETag --output text 2>/dev/null) || continue
  aws cloudfront delete-function --name "$F" --if-match "$E" && echo "deleted function $F"
done

# KeyValueStore
if [ -n "${KVS_ARN:-}" ]; then
  E=$(aws cloudfront describe-key-value-store --name lab-redirects --query ETag --output text)
  aws cloudfront delete-key-value-store --name lab-redirects --if-match "$E"
fi

# Lambda@Edge — the slow one
echo "Attempting Lambda@Edge deletion (replicas may take HOURS to clear)"
aws lambda delete-function --region us-east-1 --function-name cf-lab-edge 2>&1 | head -2
```

> **The Lambda@Edge message you'll see:**
> `Lambda was unable to delete ... because it is a replicated function.`
> This is normal. Replicas are removed asynchronously after the distribution is deleted. Retry in a
> few hours. The function costs nothing while idle, so it is safe to leave and clean up later.

```bash
# Retry script — run it tomorrow
cat > ~/cloudfront-labs/retry-edge-delete.sh <<'EOF'
#!/usr/bin/env bash
if aws lambda delete-function --region us-east-1 --function-name cf-lab-edge 2>/dev/null; then
  echo "Lambda@Edge deleted."
else
  echo "Still replicated. Try again later."
fi
EOF
chmod +x ~/cloudfront-labs/retry-edge-delete.sh
```

### Step 5 — Policies, keys, OACs

```bash
# Cache policies (custom only)
for ID in $(aws cloudfront list-cache-policies --type custom \
    --query 'CachePolicyList.Items[].CachePolicy.Id' --output text); do
  E=$(aws cloudfront get-cache-policy --id "$ID" --query ETag --output text)
  aws cloudfront delete-cache-policy --id "$ID" --if-match "$E" 2>/dev/null \
    && echo "deleted cache policy $ID"
done

# Response headers policies (custom only)
for ID in $(aws cloudfront list-response-headers-policies --type custom \
    --query 'ResponseHeadersPolicyList.Items[].ResponseHeadersPolicy.Id' --output text); do
  E=$(aws cloudfront get-response-headers-policy --id "$ID" --query ETag --output text)
  aws cloudfront delete-response-headers-policy --id "$ID" --if-match "$E" 2>/dev/null \
    && echo "deleted RHP $ID"
done

# Key group must go BEFORE the public keys it contains
if [ -n "${KG_ID:-}" ]; then
  E=$(aws cloudfront get-key-group --id "$KG_ID" --query ETag --output text)
  aws cloudfront delete-key-group --id "$KG_ID" --if-match "$E"
fi

for ID in $(aws cloudfront list-public-keys --query 'PublicKeyList.Items[].Id' --output text); do
  E=$(aws cloudfront get-public-key --id "$ID" --query ETag --output text)
  aws cloudfront delete-public-key --id "$ID" --if-match "$E" 2>/dev/null && echo "deleted key $ID"
done

# OACs
for ID in "${OAC_ID:-}" "${OAC2_ID:-}" "${LAMBDA_OAC:-}"; do
  [ -z "$ID" ] && continue
  E=$(aws cloudfront get-origin-access-control --id "$ID" --query ETag --output text 2>/dev/null) || continue
  aws cloudfront delete-origin-access-control --id "$ID" --if-match "$E" && echo "deleted OAC $ID"
done
```

### Step 6 — Logging, WAF, alarms

```bash
# Real-time log config + Kinesis
aws cloudfront delete-realtime-log-config --name cf-lab-realtime 2>/dev/null || true
aws kinesis delete-stream --stream-name cf-lab-realtime --enforce-consumer-deletion 2>/dev/null || true

# Standard logging v2 delivery
aws logs describe-deliveries --region us-east-1 \
  --query 'deliveries[].id' --output text 2>/dev/null | tr '\t' '\n' | while read -r D; do
  [ -n "$D" ] && aws logs delete-delivery --region us-east-1 --id "$D"
done
aws logs delete-delivery-source --region us-east-1 --name "cf-src-$DIST_ID" 2>/dev/null || true
aws logs delete-delivery-destination --region us-east-1 --name "cf-dest-s3" 2>/dev/null || true

# WAF
if [ -n "${WAF_ID:-}" ]; then
  LOCK=$(aws wafv2 get-web-acl --region us-east-1 --scope CLOUDFRONT \
    --name cf-lab-protection --id "$WAF_ID" --query LockToken --output text)
  aws wafv2 delete-web-acl --region us-east-1 --scope CLOUDFRONT \
    --name cf-lab-protection --id "$WAF_ID" --lock-token "$LOCK" && echo "WAF deleted"
fi

# Monitoring subscription + alarms
aws cloudfront delete-monitoring-subscription --distribution-id "$DIST_ID" 2>/dev/null || true
aws cloudwatch delete-alarms --region us-east-1 \
  --alarm-names "CF-Lab-5xx-$DIST_ID" "CF-Lab-LowHitRate-$DIST_ID" 2>/dev/null || true
```

### Step 7 — S3, Lambda, IAM

```bash
# Buckets (versioned buckets need all versions removed)
purge_bucket() {
  local B="$1"
  aws s3api list-object-versions --bucket "$B" \
    --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json > /tmp/v.json 2>/dev/null
  jq -e '.Objects != null and (.Objects|length) > 0' /tmp/v.json >/dev/null 2>&1 \
    && aws s3api delete-objects --bucket "$B" --delete file:///tmp/v.json >/dev/null
  aws s3api list-object-versions --bucket "$B" \
    --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' --output json > /tmp/d.json 2>/dev/null
  jq -e '.Objects != null and (.Objects|length) > 0' /tmp/d.json >/dev/null 2>&1 \
    && aws s3api delete-objects --bucket "$B" --delete file:///tmp/d.json >/dev/null
  aws s3 rb "s3://$B" --force && echo "deleted bucket $B"
}

purge_bucket "$BUCKET"
[ -n "${BUCKET2:-}" ] && purge_bucket "$BUCKET2"
purge_bucket "$LOG_BUCKET"
aws s3 rb "s3://athena-results-$ACCOUNT_ID" --force 2>/dev/null || true

# Lambda (the regional API from Lab 4)
aws lambda delete-function-url-config --region "$AWS_REGION" --function-name cf-lab-api 2>/dev/null || true
aws lambda delete-function --region "$AWS_REGION" --function-name cf-lab-api 2>/dev/null || true

# IAM roles
for R in cf-lab-api-role cf-lab-edge-role cf-lab-rtlogs-role; do
  for P in $(aws iam list-attached-role-policies --role-name "$R" \
      --query 'AttachedPolicies[].PolicyArn' --output text 2>/dev/null); do
    aws iam detach-role-policy --role-name "$R" --policy-arn "$P"
  done
  for P in $(aws iam list-role-policies --role-name "$R" \
      --query 'PolicyNames' --output text 2>/dev/null); do
    aws iam delete-role-policy --role-name "$R" --policy-name "$P"
  done
  aws iam delete-role --role-name "$R" 2>/dev/null && echo "deleted role $R"
done

# Athena
aws athena start-query-execution --query-string "DROP DATABASE IF EXISTS cflabs CASCADE" \
  --result-configuration "OutputLocation=s3://athena-results-$ACCOUNT_ID/" 2>/dev/null || true
```

### Step 8 — Terraform and DNS

```bash
cd ~/cloudfront-labs/terraform 2>/dev/null && terraform destroy -auto-approve

# Route 53 records (if you did Lab 2) — change UPSERT to DELETE in the same JSON
# aws route53 change-resource-record-sets --hosted-zone-id "$HZ_ID" --change-batch file:///tmp/dns-delete.json

# ACM certificate (only after all distributions using it are gone)
# aws acm delete-certificate --region us-east-1 --certificate-arn "$CERT_ARN"
```

### Step 9 — Final verification

```bash
echo "=== Distributions ==="
aws cloudfront list-distributions \
  --query 'DistributionList.Items[].{Id:Id,Status:Status}' --output table 2>/dev/null || echo "none"

echo "=== Functions ==="
aws cloudfront list-functions --query 'FunctionList.Items[].Name' --output text

echo "=== OACs ==="
aws cloudfront list-origin-access-controls \
  --query 'OriginAccessControlList.Items[].Name' --output text

echo "=== Buckets ==="
aws s3 ls | grep -E "cf-lab|cf-labs" || echo "none"

echo "=== WAF (CLOUDFRONT scope) ==="
aws wafv2 list-web-acls --region us-east-1 --scope CLOUDFRONT \
  --query 'WebACLs[].Name' --output text

echo
echo "Remaining known cost sources to check manually:"
echo "  • Lambda@Edge replicas (retry ~/cloudfront-labs/retry-edge-delete.sh tomorrow)"
echo "  • ACM certificate (free, but delete if unused)"
echo "  • Route 53 hosted zone (\$0.50/month if you created one for this)"
```

### What you learned

- CloudFront deletions are dependency-ordered: detach → disable → wait → delete.
- Lambda@Edge replicas are the one thing you can't rush.
- Terraform's `destroy` handles the whole sequence, which is a strong argument for using it from day
  one.

---

## What You Built

By the end of these labs you have hands-on experience with:

```
□ Private S3 origin with OAC and a SourceArn-scoped bucket policy
□ Custom domain, ACM in us-east-1, Route 53 alias records
□ Custom cache policies and a real understanding of cache keys and TTL precedence
□ Multiple origins and path-based routing on a single domain
□ Origin groups with automatic failover, and the limits of that failover
□ Versioned-asset deployment instead of invalidation-driven deployment
□ CloudFront Functions with test-before-publish, plus KeyValueStore-driven config
□ Lambda@Edge, including versioning, regional logs, and slow deletes
□ Signed URLs and signed cookies with key groups and rotation
□ WAF managed rules with a COUNT-then-BLOCK rollout, geo restriction, security headers
□ Standard logs + Athena, real-time logs + Kinesis, CloudWatch metrics and alarms
□ Origin Shield, compression diagnosis, price class tuning, HTTP/3
□ Continuous deployment with a staging distribution and traffic splitting
□ The whole stack as Terraform
□ A complete, ordered teardown
```

**Next:** put a real project behind it. Take an existing site, front it with CloudFront, and use
[troubleshooting.md](./troubleshooting.md) when something breaks — because something will.
