# Amazon CloudFront — Troubleshooting Guide

> Every error you'll realistically hit, with the symptom, the actual root cause, the fix, and how to
> stop it happening again. Organized so you can `Ctrl+F` the exact error string.

---

## Contents

- [How to Diagnose Anything in Five Minutes](#how-to-diagnose-anything-in-five-minutes)
- [1. Access Denied & 403 Errors](#1-access-denied--403-errors)
- [2. 502, 503 & 504 Errors](#2-502-503--504-errors)
- [3. Caching Problems](#3-caching-problems)
- [4. Custom Domain, DNS & Certificate Problems](#4-custom-domain-dns--certificate-problems)
- [5. Origin Configuration Problems](#5-origin-configuration-problems)
- [6. Routing & Cache Behavior Problems](#6-routing--cache-behavior-problems)
- [7. CORS Problems](#7-cors-problems)
- [8. Compression Problems](#8-compression-problems)
- [9. CloudFront Functions Problems](#9-cloudfront-functions-problems)
- [10. Lambda@Edge Problems](#10-lambdaedge-problems)
- [11. Signed URL & Signed Cookie Problems](#11-signed-url--signed-cookie-problems)
- [12. WAF & Security Problems](#12-waf--security-problems)
- [13. CLI & API Errors](#13-cli--api-errors)
- [14. Logging & Monitoring Problems](#14-logging--monitoring-problems)
- [15. Deployment & Propagation Problems](#15-deployment--propagation-problems)
- [16. Performance Problems](#16-performance-problems)
- [17. Cost Surprises](#17-cost-surprises)
- [18. Quota Errors](#18-quota-errors)
- [Error Code Quick Index](#error-code-quick-index)
- [Escalating to AWS Support](#escalating-to-aws-support)

---

## How to Diagnose Anything in Five Minutes

### The universal first command

```bash
curl -sSIv https://your-domain.com/the-failing-path 2>&1 | grep -iE \
  'HTTP/|x-cache|x-amz-cf-pop|x-amz-cf-id|age|content-type|content-encoding|location|server'
```

Those seven headers answer most questions before you open the console.

### The decision tree

```
Is the response an ERROR?
│
├── 403 ──► Is the body CloudFront's XML error page ("The request could not be satisfied")?
│           ├── YES ──► CloudFront rejected it: CNAME mismatch, geo block, WAF, signed URL
│           └── NO  ──► The ORIGIN returned 403: bucket policy, OAC, KMS, missing object
│
├── 404 ──► Does the object exist at that EXACT key (case-sensitive)?
│           ├── NO  ──► Upload it, or fix the URI rewrite
│           └── YES ──► Origin path prepending a prefix? Behavior pointing at the wrong origin?
│
├── 502 ──► Almost always TLS between CloudFront and the origin.
│           Check: cert expiry, hostname match, CA trust, cipher/protocol overlap
│
├── 503 ──► Capacity, Lambda@Edge failure, or origin refusing connections
│
├── 504 ──► Origin didn't answer in time, or a firewall/SG/NACL silently dropped the packet
│
└── 200 but WRONG ──► Cache problem. Check X-Cache and Age, then the cache key.
```

### The four-layer isolation test

Run these in order. The first one that fails tells you which layer is broken.

```bash
# LAYER 1 — Is the origin itself healthy? (bypass CloudFront entirely)
curl -sSI https://your-origin.example.com/path
#   For a private S3 origin, use the AWS API instead:
aws s3api head-object --bucket my-bucket --key path/file.html

# LAYER 2 — Does CloudFront's default domain work?
curl -sSI https://d111abcdef8.cloudfront.net/path

# LAYER 3 — Does your custom domain work?
curl -sSI https://cdn.example.com/path

# LAYER 4 — Does DNS resolve correctly?
dig +short cdn.example.com
```

| Layer that fails | Where the problem is |
|---|---|
| 1 | Origin — nothing to do with CloudFront |
| 2 (but 1 works) | Distribution config: origin settings, OAC, behaviors, permissions |
| 3 (but 2 works) | Alias, certificate, or WAF/geo rules tied to the custom domain |
| 4 | DNS records |

### The five questions that solve most tickets

1. **Is it cached?** `X-Cache` header. A stale `Hit` explains a lot.
2. **Is it deployed?** `aws cloudfront get-distribution --id X --query 'Distribution.Status'` — if
   it says `InProgress`, you're testing the old config.
3. **Is the origin reachable?** Test it directly, bypassing CloudFront.
4. **Which behavior matched?** Path patterns are ordered and case-sensitive.
5. **Which PoP?** `X-Amz-Cf-Pop`. Regional inconsistency usually means partial propagation.

---

## 1. Access Denied & 403 Errors

### 1.1 `AccessDenied` from S3 — the OAC bucket policy is missing or wrong

**Symptom:**
```xml
<Error><Code>AccessDenied</Code><Message>Access Denied</Message></Error>
```
delivered through CloudFront with `x-cache: Error from cloudfront`.

**Root causes, in order of likelihood:**

1. **The bucket policy was never applied.** OAC on the distribution does nothing without the
   matching bucket policy. This is the single most common CloudFront error in existence.
2. **`AWS:SourceArn` doesn't match this distribution.** Copy-pasted from another distribution, or
   the distribution was recreated with a new ID.
3. **Block Public Access is fine, but the policy grants the wrong action** (needs `s3:GetObject`).
4. **The bucket uses SSE-KMS and the key policy doesn't allow CloudFront.**
5. **The object genuinely doesn't exist** — private buckets return 403, not 404, to avoid leaking
   key names.
6. **The distribution still has the OAC configured as `never` sign**, or the origin was created
   before the OAC was attached.

**Diagnosis:**

```bash
DIST_ID=E1234ABCDEFGH
BUCKET=my-bucket

# 1. Does the object actually exist?
aws s3api head-object --bucket $BUCKET --key index.html

# 2. What does the bucket policy say?
aws s3api get-bucket-policy --bucket $BUCKET --query Policy --output text | jq .

# 3. Is OAC actually attached to the origin?
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.Origins.Items[].{Id:Id,Domain:DomainName,OAC:OriginAccessControlId}' \
  --output table

# 4. Does the SourceArn in the policy match THIS distribution?
aws cloudfront get-distribution --id $DIST_ID --query 'Distribution.ARN' --output text

# 5. Is the bucket encrypted with KMS?
aws s3api get-bucket-encryption --bucket $BUCKET
```

**Fix — the correct bucket policy:**

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > /tmp/bp.json <<EOF
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

aws s3api put-bucket-policy --bucket $BUCKET --policy file:///tmp/bp.json
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

**Prevention:** manage the distribution and the bucket policy in the same IaC module, so the policy
can never drift from the distribution ARN.

---

### 1.2 403 "The request could not be satisfied" — `CNAMEMismatch`

**Symptom:** an HTML error page from CloudFront, often with `ERROR: The request could not be
satisfied` and a request ID. The `X-Amz-Cf-Id` header is present but no origin was contacted.

**Root cause:** the `Host` header on the request doesn't match any alternate domain name on any
distribution. Usually one of:

- DNS points at CloudFront but you never added the domain to the distribution's `Aliases`.
- You added it to the wrong distribution.
- You're testing with an IP address or a `--resolve` override and the SNI/Host don't match.
- The certificate covers the domain but the alias list doesn't (these are two separate settings).

**Diagnosis:**

```bash
# What aliases does the distribution actually have?
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.Aliases' --output json

# Is another distribution claiming this name?
aws cloudfront list-conflicting-aliases --distribution-id $DIST_ID --alias cdn.example.com

# Where does DNS actually point?
dig +short cdn.example.com
```

**Fix:**

```bash
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/f.json
ETAG=$(jq -r .ETag /tmp/f.json)
jq '.DistributionConfig
    | .Aliases.Items += ["cdn.example.com"]
    | .Aliases.Quantity = (.Aliases.Items | length)' /tmp/f.json > /tmp/n.json
aws cloudfront update-distribution --id $DIST_ID --if-match $ETAG \
  --distribution-config file:///tmp/n.json
```

Remember: adding an alias also requires a certificate that covers it.

---

### 1.3 403 from geo restriction

**Symptom:** works for you, 403 for users in specific countries. CloudFront error page, no origin
hit, and nothing in the origin logs.

**Diagnosis:**

```bash
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.Restrictions.GeoRestriction' --output json
```

**Fix:** adjust the list, or switch to a WAF geo-match rule if you need per-path control.

```bash
# Remove all geo restrictions
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/f.json
ETAG=$(jq -r .ETag /tmp/f.json)
jq '.DistributionConfig | .Restrictions.GeoRestriction =
    {"RestrictionType":"none","Quantity":0}' /tmp/f.json > /tmp/n.json
aws cloudfront update-distribution --id $DIST_ID --if-match $ETAG \
  --distribution-config file:///tmp/n.json
```

**Note:** geo restriction is distribution-wide, not per behavior. If you need "block country X only
on `/admin`", that's a WAF rule.

---

### 1.4 403 with `MethodNotAllowed` — the HTTP method isn't in AllowedMethods

**Symptom:**
```xml
<Error><Code>MethodNotAllowed</Code>
<Message>The specified method is not allowed against this resource.</Message></Error>
```
`POST`/`PUT`/`DELETE` fail; `GET` works.

**Root cause:** the matched cache behavior only allows `GET, HEAD`.

**Fix:**

```bash
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/f.json
ETAG=$(jq -r .ETag /tmp/f.json)
jq '.DistributionConfig
    | .CacheBehaviors.Items = [ .CacheBehaviors.Items[]?
        | if .PathPattern == "/api/*" then
            .AllowedMethods = {
              "Quantity": 7,
              "Items": ["GET","HEAD","OPTIONS","PUT","POST","PATCH","DELETE"],
              "CachedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] }
            }
          else . end ]' /tmp/f.json > /tmp/n.json
aws cloudfront update-distribution --id $DIST_ID --if-match $ETAG \
  --distribution-config file:///tmp/n.json
```

**Watch out:** `CachedMethods` must be a subset of `AllowedMethods`, and can only be `GET,HEAD` or
`GET,HEAD,OPTIONS`.

---

### 1.5 403 on a Lambda Function URL or API Gateway origin

**Symptom:** the origin works when called directly (or with IAM auth), but 403 through CloudFront.

**Root causes:**

| Cause | Fix |
|---|---|
| Function URL auth type is `AWS_IAM` but CloudFront has no OAC | Create an OAC with `OriginAccessControlOriginType: lambda` |
| The Lambda resource policy doesn't allow the CloudFront service principal | `aws lambda add-permission ... --principal cloudfront.amazonaws.com --source-arn <dist-arn> --function-url-auth-type AWS_IAM` |
| The `Host` header is being forwarded | Use `AllViewerExceptHostHeader`, not `AllViewer` |
| API Gateway resource policy restricts source | Add the distribution or CloudFront ranges |

```bash
aws lambda add-permission --region ap-south-1 \
  --function-name my-api \
  --statement-id AllowCloudFront \
  --action lambda:InvokeFunctionUrl \
  --principal cloudfront.amazonaws.com \
  --source-arn "arn:aws:cloudfront::$ACCOUNT_ID:distribution/$DIST_ID" \
  --function-url-auth-type AWS_IAM
```

---

### 1.6 403 that only happens for some objects

**Almost always:** those specific objects don't exist, or their keys differ in case.

```bash
# S3 keys are case-sensitive. /Images/Logo.PNG ≠ /images/logo.png
aws s3 ls "s3://$BUCKET/images/" --recursive | head -20
aws s3api head-object --bucket $BUCKET --key "images/logo.png"
```

Also check `OriginPath` — it silently prepends a prefix:

```bash
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.Origins.Items[].{Id:Id,Path:OriginPath}' --output table
```

If `OriginPath` is `/public`, then a request for `/logo.png` fetches `s3://bucket/public/logo.png`.

---

## 2. 502, 503 & 504 Errors

### 2.1 502 Bad Gateway — `OriginSSLHandshakeFailure` (the most common 502)

**Symptom:**
```
502 ERROR
The request could not be satisfied.
CloudFront wasn't able to connect to the origin.
```
Log field `x-edge-detailed-result-type` shows `OriginSSLHandshakeFailure` or similar.

**Root causes, ranked:**

1. **Origin certificate expired.**
2. **Certificate CN/SAN doesn't match the origin domain name** you configured in CloudFront.
3. **Self-signed or private CA certificate** — CloudFront requires a publicly trusted CA.
4. **Incomplete certificate chain** — the origin serves the leaf but not the intermediates.
   Browsers often paper over this; CloudFront doesn't.
5. **No shared TLS protocol/cipher.** You set `OriginSslProtocols: [TLSv1.2]` but the origin only
   speaks TLS 1.3, or vice versa.
6. **`OriginProtocolPolicy: https-only`** against an origin that only listens on HTTP.

**Diagnosis:**

```bash
ORIGIN=origin.example.com

# Certificate validity and chain
echo | openssl s_client -connect $ORIGIN:443 -servername $ORIGIN 2>/dev/null \
  | openssl x509 -noout -dates -subject -issuer -ext subjectAltName

# Full chain — look for "Verify return code: 0 (ok)"
echo | openssl s_client -connect $ORIGIN:443 -servername $ORIGIN -showcerts 2>&1 \
  | grep -E 'Verify return code|s:|i:'

# Which protocols does the origin accept?
for P in tls1 tls1_1 tls1_2 tls1_3; do
  printf "%-8s " $P
  echo | openssl s_client -connect $ORIGIN:443 -$P 2>&1 | grep -q 'Verify' \
    && echo OK || echo FAIL
done

# What did you configure?
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.Origins.Items[].{Id:Id,Domain:DomainName,Protocol:CustomOriginConfig.OriginProtocolPolicy,SSL:CustomOriginConfig.OriginSslProtocols.Items}' \
  --output json
```

**Fixes:**

```bash
# Broaden the accepted origin SSL protocols
jq '.DistributionConfig | .Origins.Items[0].CustomOriginConfig.OriginSslProtocols =
    {"Quantity":2,"Items":["TLSv1.1","TLSv1.2"]}' /tmp/f.json > /tmp/n.json

# Or, if the origin genuinely has no TLS (internal only), use VPC origins with http-only
jq '.DistributionConfig | .Origins.Items[0].CustomOriginConfig.OriginProtocolPolicy = "http-only"' \
  /tmp/f.json > /tmp/n.json
```

**Real fix for the chain problem:** configure the origin to serve the full chain (leaf +
intermediates). On nginx that's `ssl_certificate fullchain.pem;`, not `cert.pem`.

---

### 2.2 502 — origin returned a malformed or oversized response

**Other 502 causes:**

| Cause | Detail |
|---|---|
| Response headers exceed the limit | Total request/origin-response header size cap is 32,768 bytes. A huge `Set-Cookie` or oversized JWT in a header will do it. |
| Origin closed the connection mid-response | Application crash, upstream timeout, worker restart |
| Invalid HTTP in the response | Bad status line, duplicate `Content-Length`, illegal header characters |
| Lambda@Edge returned an invalid object | Missing `status`, wrong header shape (see section 10) |
| Origin returned a body on a 204/304 | Protocol violation |

```bash
# Check the header size your origin sends
curl -sSI https://origin.example.com/path | wc -c
```

---

### 2.3 504 Gateway Timeout — origin didn't respond in time

**Symptom:** requests hang for ~30 seconds then return 504.

**Root causes:**

1. **Origin is genuinely slow** (heavy query, cold Lambda, report generation).
2. **Security group / NACL / firewall silently drops CloudFront traffic** — packets vanish instead
   of being rejected, so you get a timeout rather than a connection refusal.
3. **Origin is overloaded** — the classic cascade after an invalidation storm.
4. **Origin is in a private subnet with no route** and you're not using VPC origins.

**Diagnosis:**

```bash
# 1. Time the origin directly
curl -sS -o /dev/null -w 'ttfb=%{time_starttransfer}s total=%{time_total}s\n' \
  https://origin.example.com/slow-path

# 2. Current timeout settings
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.Origins.Items[].{Id:Id,Read:CustomOriginConfig.OriginReadTimeout,KeepAlive:CustomOriginConfig.OriginKeepaliveTimeout,ConnTimeout:ConnectionTimeout,Attempts:ConnectionAttempts}' \
  --output table

# 3. Is the security group letting CloudFront in?
aws ec2 describe-security-groups --group-ids sg-xxxx \
  --query 'SecurityGroups[].IpPermissions' --output json

# 4. Origin latency metric
aws cloudwatch get-metric-statistics --region us-east-1 \
  --namespace AWS/CloudFront --metric-name OriginLatency \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --start-time $(date -u -d '3 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average,Maximum
```

**Fix — raise the timeout (max 120 s, may need a quota increase above the default):**

```bash
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/f.json
ETAG=$(jq -r .ETag /tmp/f.json)
jq '.DistributionConfig
    | .Origins.Items[0].CustomOriginConfig.OriginReadTimeout = 60
    | .Origins.Items[0].CustomOriginConfig.OriginKeepaliveTimeout = 30' \
  /tmp/f.json > /tmp/n.json
aws cloudfront update-distribution --id $DIST_ID --if-match $ETAG \
  --distribution-config file:///tmp/n.json
```

**Fix — allow CloudFront in the security group using the managed prefix list:**

```bash
PL_ID=$(aws ec2 describe-managed-prefix-lists --region ap-south-1 \
  --filters "Name=prefix-list-name,Values=com.amazonaws.global.cloudfront.origin-facing" \
  --query 'PrefixLists[0].PrefixListId' --output text)

aws ec2 authorize-security-group-ingress --group-id sg-xxxx \
  --ip-permissions "IpProtocol=tcp,FromPort=443,ToPort=443,PrefixListIds=[{PrefixListId=$PL_ID}]"
```

> **Never hardcode CloudFront IP ranges.** They change. Use the prefix list, or better, VPC origins.

**The real fix for genuinely slow origins:** raising the timeout hides the problem. Cache the
expensive response, precompute it, or make the endpoint asynchronous.

---

### 2.4 503 Service Unavailable

| Variant | Meaning | Fix |
|---|---|---|
| `CapacityExceeded` / `LimitExceeded` | You exceeded the requests-per-second or data-rate quota for the distribution | Request a quota increase; check for an attack |
| `LambdaExecutionError` / `LambdaValidationError` | Your Lambda@Edge function threw or returned an invalid object | See section 10 |
| Origin returned 503 | Your app is unhealthy | Fix the origin; consider an origin group |
| `OriginConnectError` | Couldn't establish a connection at all | Check DNS for the origin domain, SG, and that the origin is running |

```bash
# Which 503 variant is it? Look at the detailed result type in the logs.
# In Athena:
#   SELECT x_edge_detailed_result_type, COUNT(*) FROM cf_logs
#   WHERE sc_status = 503 GROUP BY 1 ORDER BY 2 DESC;

# Check whether you're near the RPS quota
aws service-quotas list-service-quotas --service-code cloudfront \
  --query "Quotas[?contains(QuotaName,'Requests per second')]" --output table
```

---

### 2.5 Intermittent 5xx that you can't reproduce

**Approach:**

```bash
# 1. Get the exact scope from CloudWatch
aws cloudwatch get-metric-statistics --region us-east-1 \
  --namespace AWS/CloudFront --metric-name 5xxErrorRate \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average,Maximum --output table

# 2. Find the pattern in the logs
#   SELECT cs_uri_stem, sc_status, x_edge_detailed_result_type, COUNT(*) n
#   FROM cf_logs WHERE sc_status >= 500 GROUP BY 1,2,3 ORDER BY n DESC LIMIT 30;

# 3. Is it regional? (points at partial propagation or a regional origin problem)
#   SELECT substr(x_edge_location,1,3) pop, COUNT(*) FROM cf_logs
#   WHERE sc_status >= 500 GROUP BY 1 ORDER BY 2 DESC;
```

**Common causes of intermittent 5xx:**
- Origin autoscaling lag during traffic spikes
- Connection pool exhaustion at the origin (raise `OriginKeepaliveTimeout` to reuse connections)
- One unhealthy backend in an ALB target group
- Lambda@Edge cold starts hitting the timeout
- A cron job on the origin causing periodic latency spikes

---

## 3. Caching Problems

### 3.1 "I deployed but users still see the old version"

**Diagnosis:**

```bash
curl -sSI https://cdn.example.com/index.html | grep -iE 'x-cache|age|cache-control|etag|last-modified'
```

| What you see | Meaning |
|---|---|
| `x-cache: Hit from cloudfront` + high `Age` | CloudFront is serving a cached copy. Invalidate or wait for TTL. |
| `x-cache: Miss from cloudfront` but old content | The **origin** is serving old content. Check S3 actually has the new object. |
| Correct in `curl`, old in browser | **Browser cache.** Your problem is `Cache-Control`, not CloudFront. |

```bash
# Confirm the origin has the new object
aws s3api head-object --bucket $BUCKET --key index.html \
  --query '{Modified:LastModified,ETag:ETag,CacheControl:CacheControl}'

# Invalidate
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/index.html"
```

**The permanent fix:**

```
/index.html          Cache-Control: no-cache          ← revalidated every time
/assets/*.[hash].js  Cache-Control: max-age=31536000, immutable
```

With content-hashed asset filenames you only ever invalidate `/index.html` — one free path, atomic
switch, no stale-asset mismatch.

---

### 3.2 Cache hit ratio is terrible

**Diagnose with the console first** — Reports → Cache statistics and Popular objects show per-object
hit/miss counts with no SQL required.

**Then check the cache key:**

```bash
CP_ID=$(aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.DefaultCacheBehavior.CachePolicyId' --output text)
aws cloudfront get-cache-policy --id $CP_ID \
  --query 'CachePolicy.CachePolicyConfig.ParametersInCacheKeyAndForwardedToOrigin' --output json
```

**The offender list:**

| Found in the cache key | Effect | Fix |
|---|---|---|
| `User-Agent` | Millions of unique values → ~0% hit ratio | Remove it. Use `CloudFront-Is-Mobile-Viewer` if you need device detection. |
| All headers (`allViewer`) | Same disaster | Whitelist only what varies the response |
| All cookies | Session cookies are unique per user | Whitelist, or `none` |
| All query strings | `utm_*`, `fbclid` fragment everything | Whitelist only the params that change the response |
| `Authorization` | Per-user cache entries | Don't cache authenticated responses at all |

**Then check the origin's headers:**

```bash
# What is the origin telling CloudFront?
curl -sSI https://origin.example.com/asset.js | grep -i cache-control
```

`no-store`, `private`, or a missing `Cache-Control` with `DefaultTTL: 0` all produce a permanent
miss.

**Then check for `Set-Cookie`:**

```bash
curl -sSI https://cdn.example.com/asset.js | grep -i set-cookie
```

An origin that sets a cookie on every response (analytics middleware, session middleware applied
globally) will suppress caching. Exclude static paths from that middleware.

**Athena worklist query:**

```sql
SELECT cs_uri_stem, COUNT(*) AS misses
FROM cf_logs
WHERE x_edge_result_type = 'Miss' AND "date" >= current_date - interval '1' day
GROUP BY cs_uri_stem ORDER BY misses DESC LIMIT 50;
```

---

### 3.3 Different users get each other's content

**This is a correctness bug, not a performance one. Treat it as an incident.**

**Immediate mitigation:**

```bash
# Switch the affected behavior to CachingDisabled, then invalidate
CACHING_DISABLED=$(aws cloudfront list-cache-policies --type managed \
  --query "CachePolicyList.Items[?CachePolicy.CachePolicyConfig.Name=='Managed-CachingDisabled'].CachePolicy.Id" \
  --output text)

aws cloudfront get-distribution-config --id $DIST_ID > /tmp/f.json
ETAG=$(jq -r .ETag /tmp/f.json)
jq --arg cp "$CACHING_DISABLED" '.DistributionConfig
    | .DefaultCacheBehavior.CachePolicyId = $cp' /tmp/f.json > /tmp/n.json
aws cloudfront update-distribution --id $DIST_ID --if-match $ETAG \
  --distribution-config file:///tmp/n.json

aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/*"
```

**Root cause:** the cache key didn't include the thing that distinguishes users — a query string, a
cookie, or the `Authorization` header.

**Permanent fix — pick one:**

1. **Don't cache authenticated responses.** Send `Cache-Control: private, no-store` from the origin
   for anything user-specific. This is the right answer 95% of the time.
2. **Separate the paths.** Put user-specific content under `/api/*` or `/me/*` with
   `CachingDisabled`, and keep truly public content cacheable.
3. **If you must cache per user**, add the distinguishing input to the cache key and accept the low
   hit ratio. Rarely worth it.

**Prevention:** any endpoint that reads a session should send `Cache-Control: private` from the
origin. Make it a middleware default, not a per-route decision.

---

### 3.4 Invalidation completed but content is still old

**Checks:**

```bash
# 1. Did it actually complete?
aws cloudfront get-invalidation --distribution-id $DIST_ID --id I2ABCDEF \
  --query 'Invalidation.{Status:Status,Paths:InvalidationBatch.Paths.Items}'

# 2. Bypass all caches
curl -sSI -H 'Cache-Control: no-cache' "https://cdn.example.com/index.html?cb=$RANDOM"
```

**Common causes:**

| Cause | Detail |
|---|---|
| Path didn't match | Invalidation paths are **case-sensitive** and must start with `/`. `/Index.html` ≠ `/index.html`. |
| Query strings not covered | `/file.js` does not invalidate `/file.js?v=2`. Use `/file.js*`. |
| Browser cache | The browser is holding it, not CloudFront. Check in a private window or with `curl`. |
| Origin still has old content | Invalidation clears the CDN, not your S3 bucket. |
| Special characters not URL-encoded | `/my file.jpg` must be `/my%20file.jpg`. |
| An intermediate corporate proxy | Outside your control; test from a different network. |

---

### 3.5 Content is cached when it shouldn't be

```bash
# What is the origin sending?
curl -sSI https://origin.example.com/api/data | grep -i cache-control

# What are the policy TTLs?
aws cloudfront get-cache-policy --id $CP_ID \
  --query 'CachePolicy.CachePolicyConfig.{Min:MinTTL,Default:DefaultTTL,Max:MaxTTL}'
```

**The trap:** a `MinTTL` above 0 **overrides** the origin's `no-cache`. If your policy has
`MinTTL: 300` and the origin says `Cache-Control: no-cache`, CloudFront serves stale content for
five minutes.

**Fix:** set `MinTTL: 0` and let the origin decide, or use the `CachingDisabled` managed policy for
dynamic paths.

---

### 3.6 `X-Cache` is always `Miss` even for static files

Walk this list:

```bash
# 1. Is the method cacheable? Only GET/HEAD (and optionally OPTIONS) are cached.
# 2. Is the status code cacheable?
curl -sSI https://cdn.example.com/file.js | head -1

# 3. Is there a Set-Cookie?
curl -sSI https://cdn.example.com/file.js | grep -i set-cookie

# 4. Is Cache-Control saying no-store / private?
curl -sSI https://cdn.example.com/file.js | grep -i cache-control

# 5. Is the cache key over-specified? (see 3.2)

# 6. Is traffic just too low? An unpopular object gets evicted from a small edge cache
#    between your two test requests. Enable Origin Shield or test the same PoP repeatedly.
curl -sSI https://cdn.example.com/file.js | grep -i x-amz-cf-pop
```

Point 6 is under-appreciated: on a low-traffic site, objects legitimately fall out of the edge cache.
`RefreshHit` in the logs means the Regional Edge Cache still had it.

---

## 4. Custom Domain, DNS & Certificate Problems

### 4.1 The ACM certificate doesn't appear in the console dropdown

**Cause:** it's not in `us-east-1`. There is no other cause.

```bash
# Look in us-east-1
aws acm list-certificates --region us-east-1 \
  --query 'CertificateSummaryList[].{Domain:DomainName,Status:Status,Arn:CertificateArn}' --output table

# Look where you accidentally created it
aws acm list-certificates --region ap-south-1 \
  --query 'CertificateSummaryList[].DomainName' --output table
```

**Fix:** request it again in `us-east-1`. Certificates cannot be moved between regions.

---

### 4.2 `InvalidViewerCertificate`

**Symptom:**
```
An error occurred (InvalidViewerCertificate) when calling the UpdateDistribution operation
```

**Checklist:**

| Check | Command |
|---|---|
| Certificate is in `us-east-1` | `aws acm describe-certificate --region us-east-1 --certificate-arn $ARN` |
| Status is `ISSUED` (not `PENDING_VALIDATION`) | `--query 'Certificate.Status'` |
| It covers **every** alias on the distribution | `--query 'Certificate.SubjectAlternativeNames'` |
| Wildcard depth is right | `*.example.com` covers `cdn.example.com` but **not** `example.com` or `a.b.example.com` |
| `SSLSupportMethod` is set | `sni-only` for ACM certs |
| `MinimumProtocolVersion` is compatible | `TLSv1.2_2021` with `sni-only` |

```bash
aws acm describe-certificate --region us-east-1 --certificate-arn "$CERT_ARN" \
  --query 'Certificate.{Status:Status,Domain:DomainName,SANs:SubjectAlternativeNames,NotAfter:NotAfter}'
```

---

### 4.3 `CNAMEAlreadyExists`

**Symptom:**
```
One or more of the CNAMEs you provided are already associated with a different resource.
```

**Cause:** an alternate domain name can only be on **one distribution at a time, globally across all
AWS accounts**.

```bash
# Find the conflict (works even if the other distribution is in another account you own)
aws cloudfront list-conflicting-aliases --distribution-id $DIST_ID --alias cdn.example.com
```

**Fixes:**

```bash
# If you own the other distribution: move the alias with no downtime
aws cloudfront associate-alias --target-distribution-id E_NEW --alias cdn.example.com

# Or remove it from the old distribution first, then add it to the new one
```

If someone else's account holds it, you need to prove domain ownership to AWS Support.

---

### 4.4 Certificate validation stuck at `PENDING_VALIDATION`

```bash
# What record does ACM want?
aws acm describe-certificate --region us-east-1 --certificate-arn "$CERT_ARN" \
  --query 'Certificate.DomainValidationOptions[].{Domain:DomainName,Name:ResourceRecord.Name,Type:ResourceRecord.Type,Value:ResourceRecord.Value}' \
  --output table

# Does that record actually resolve?
dig +short _abc123.cdn.example.com CNAME
```

**Common causes:**
- The CNAME was added with the domain name doubled (`_abc.cdn.example.com.example.com`) — many DNS
  UIs append the zone automatically.
- A trailing dot mismatch.
- CAA records on the domain that don't permit `amazon.com`.

```bash
dig +short example.com CAA
# If CAA records exist, they must include: 0 issue "amazon.com"
```

---

### 4.5 Domain resolves but returns CloudFront's error page

Order of operations matters:

```
1. Add the alias to the distribution   ← do this FIRST
2. Attach a certificate covering it
3. Wait for Deployed
4. THEN point DNS at CloudFront
```

If you point DNS first, every request gets `CNAMEMismatch` 403 until step 1 completes.

```bash
aws cloudfront get-distribution --id $DIST_ID --query 'Distribution.Status' --output text
aws cloudfront get-distribution-config --id $DIST_ID --query 'DistributionConfig.Aliases' --output json
```

---

### 4.6 Apex domain (`example.com`) won't work

**Cause:** DNS doesn't allow a CNAME at the zone apex.

| DNS provider | Solution |
|---|---|
| Route 53 | **A record with Alias = yes**, target the distribution, hosted zone ID `Z2FDTNDATAQYW2`. Add AAAA too. |
| Cloudflare | CNAME flattening (automatic) |
| Others | ALIAS or ANAME record if supported; otherwise redirect apex → `www` via an S3 redirect bucket or a separate distribution |

---

### 4.7 Mixed content warnings in the browser

**Cause:** the page is HTTPS but references `http://` resources.

**Fixes:**
- Use protocol-relative or absolute HTTPS URLs in your HTML.
- Set `ViewerProtocolPolicy: redirect-to-https` on every behavior.
- Add a CSP `upgrade-insecure-requests` directive via a response headers policy.

```bash
curl -sS https://cdn.example.com/ | grep -o 'http://[^"'\'']*' | sort -u | head
```

---

## 5. Origin Configuration Problems

### 5.1 S3 REST endpoint vs S3 website endpoint — picking the wrong one

| | REST endpoint | Website endpoint |
|---|---|---|
| Domain | `bucket.s3.region.amazonaws.com` | `bucket.s3-website-region.amazonaws.com` |
| CloudFront treats it as | S3 origin | **Custom** origin |
| Can the bucket be private? | ✅ (with OAC) | ❌ must be public |
| HTTPS to origin? | ✅ | ❌ HTTP only |
| Directory index (`/about/` → `index.html`) | ❌ | ✅ |
| S3 redirect rules | ❌ | ✅ |
| SSE-KMS | ✅ | ❌ |

**Symptom of picking wrong:** you used the REST endpoint and `/about/` returns 403 or 404.

**Fix (recommended):** keep the REST endpoint + OAC, and add a CloudFront Function to rewrite URIs:

```javascript
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    if (uri.endsWith('/')) { request.uri = uri + 'index.html'; }
    else if (!uri.includes('.')) { request.uri = uri + '/index.html'; }
    return request;
}
```

**Symptom of the reverse mistake:** you configured a website endpoint with `S3OriginConfig` and OAC.
CloudFront won't sign correctly and you'll get errors. A website endpoint **must** be configured as
a `CustomOriginConfig` with `OriginProtocolPolicy: http-only`.

---

### 5.2 `OriginPath` doubling or missing a prefix

```bash
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.Origins.Items[].{Id:Id,Domain:DomainName,Path:OriginPath}' --output table
```

**Rules:**
- `OriginPath` is prepended to the request path: `OriginPath=/prod` + request `/users` →
  origin receives `/prod/users`.
- It must **not** end with `/` — that produces a double slash.
- It must start with `/` if non-empty.
- API Gateway stage names go here (`/prod`), not in the domain.

**Symptom of doubling:** origin logs show `/prod/prod/users` because you set both `OriginPath` and
a path prefix in your rewrite function.

---

### 5.3 The `Host` header breaks S3 or API Gateway

**Symptom (S3 + OAC):** 403 on everything after you attached an origin request policy.
**Symptom (API Gateway):** `{"message":"Forbidden"}` or a 403 from API Gateway.

**Cause:** forwarding the viewer's `Host` header. For S3 it breaks SigV4 signing. For API Gateway it
breaks stage routing and the custom-domain mapping.

```bash
ORP=$(aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.DefaultCacheBehavior.OriginRequestPolicyId' --output text)
aws cloudfront get-origin-request-policy --id $ORP \
  --query 'OriginRequestPolicy.OriginRequestPolicyConfig.HeadersConfig' --output json
```

**Fix:**

| Origin type | Correct origin request policy |
|---|---|
| S3 + OAC | `CORS-S3Origin`, or none at all |
| API Gateway | `AllViewerExceptHostHeader` |
| Lambda Function URL | `AllViewerExceptHostHeader` |
| ALB (app reads Host for vhosts) | `AllViewer` |
| ALB (app doesn't care) | `AllViewerExceptHostHeader` |

---

### 5.4 Origin group failover isn't triggering

**Checklist:**

```bash
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.OriginGroups' --output json
```

| Requirement | Detail |
|---|---|
| The behavior's `TargetOriginId` must be the **origin group ID**, not an origin ID | Easy to miss |
| The status code must be in `FailoverCriteria.StatusCodes` | 502 isn't included by default in every setup |
| The method must be `GET`, `HEAD`, or `OPTIONS` | `POST` is never retried |
| The secondary origin must actually have the content | And its own bucket policy/OAC if it's S3 |
| Both origins must be defined in `Origins` | The group only references them |

**Reminder:** origin groups are reactive, not health-checked. There is no background probing.

---

### 5.5 Direct origin access isn't blocked

**Test it:**

```bash
# S3
curl -sS -o /dev/null -w '%{http_code} (want 403)\n' \
  "https://$BUCKET.s3.$AWS_REGION.amazonaws.com/index.html"

# ALB
curl -sS -o /dev/null -w '%{http_code} (want 403)\n' \
  "https://my-alb-123.ap-south-1.elb.amazonaws.com/"
```

**Fixes, best to worst:**

1. **VPC origins** — put the ALB/NLB/EC2 in private subnets. There is no bypass path to close.
2. **Shared secret header** — CloudFront injects `X-Origin-Verify: <secret>`; an ALB listener rule
   returns a fixed 403 for anything without it.
3. **Managed prefix list** in the security group — restricts *who* can connect, but anyone can still
   craft requests from a CloudFront distribution of their own unless combined with #2.

```bash
# ALB listener rule (concept)
# IF http-header X-Origin-Verify = <secret> THEN forward to target group
# ELSE fixed-response 403
```

---

## 6. Routing & Cache Behavior Problems

### 6.1 The wrong behavior is matching

```bash
# Behaviors are evaluated IN ORDER; first match wins
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.CacheBehaviors.Items[].{Pattern:PathPattern,Origin:TargetOriginId}' \
  --output table
```

**Rules:**
- Most specific patterns must come **first**. `/api/v2/*` before `/api/*`.
- Patterns are **case-sensitive**. `/Images/*` does not match `/images/logo.png`.
- The default `*` behavior always evaluates **last**, regardless of the list.
- Query strings are not part of the pattern match.
- `?` is not a wildcard; only `*` is.

**Reorder:**

```bash
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/f.json
ETAG=$(jq -r .ETag /tmp/f.json)
# Sort so longer (more specific) patterns come first
jq '.DistributionConfig
    | .CacheBehaviors.Items |= sort_by(-(.PathPattern | length))' /tmp/f.json > /tmp/n.json
aws cloudfront update-distribution --id $DIST_ID --if-match $ETAG \
  --distribution-config file:///tmp/n.json
```

Length-sorting is a heuristic, not a rule — review the result before applying.

---

### 6.2 SPA client-side routes 404 on refresh

**Symptom:** `/dashboard` works when navigated to in-app, 403/404 on a hard refresh.

**Cause:** the browser asks CloudFront for `/dashboard`, which doesn't exist as an object.

**Fix A — custom error responses:**

```bash
jq '.DistributionConfig | .CustomErrorResponses = {
  "Quantity": 2,
  "Items": [
    {"ErrorCode":403,"ResponsePagePath":"/index.html","ResponseCode":"200","ErrorCachingMinTTL":10},
    {"ErrorCode":404,"ResponsePagePath":"/index.html","ResponseCode":"200","ErrorCachingMinTTL":10}
  ]}' /tmp/f.json > /tmp/n.json
```

You need **both** 403 and 404: private S3 buckets return 403 for missing keys.

**Fix B — a CloudFront Function (cleaner):**

```javascript
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    // Leave API paths and real files alone
    if (uri.startsWith('/api/')) { return request; }
    if (uri.includes('.')) { return request; }
    request.uri = '/index.html';
    return request;
}
```

**Why B is better:** Fix A masks *all* 403/404s, including genuine API errors, which makes
debugging miserable. If you use Fix A, keep `/api/*` on a separate behavior.

---

### 6.3 Trailing slash / redirect loops

**Symptom:** `ERR_TOO_MANY_REDIRECTS`.

**Common causes:**

| Cause | Fix |
|---|---|
| S3 website endpoint + `ViewerProtocolPolicy: redirect-to-https` + `OriginProtocolPolicy: match-viewer` | Set origin protocol to `http-only` (website endpoints are HTTP-only) |
| Origin redirects HTTP→HTTPS and CloudFront forwards HTTP | Set origin protocol to `https-only` |
| A CloudFront Function redirecting to a URL that the function itself rewrites again | Add a guard condition |
| The app redirects `/page` → `/page/` while your rewrite does the reverse | Pick one canonical form |

```bash
curl -sSIL --max-redirs 5 https://cdn.example.com/page 2>&1 | grep -iE 'HTTP/|location'
```

---

## 7. CORS Problems

### 7.1 `No 'Access-Control-Allow-Origin' header is present`

**The three-layer checklist:**

```bash
# 1. Does the ORIGIN send CORS headers?
curl -sSI -H 'Origin: https://app.example.com' https://origin.example.com/api/data \
  | grep -i access-control

# 2. Does CloudFront FORWARD the Origin header to the origin?
ORP=$(aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.DefaultCacheBehavior.OriginRequestPolicyId' --output text)
aws cloudfront get-origin-request-policy --id $ORP --output json

# 3. Is OPTIONS allowed?
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.DefaultCacheBehavior.AllowedMethods' --output json
```

**Fixes:**

```bash
# Allow OPTIONS
jq '.DistributionConfig | .DefaultCacheBehavior.AllowedMethods = {
      "Quantity": 3, "Items": ["GET","HEAD","OPTIONS"],
      "CachedMethods": { "Quantity": 3, "Items": ["GET","HEAD","OPTIONS"] }
    }' /tmp/f.json > /tmp/n.json

# Use the managed CORS-S3Origin origin request policy (forwards Origin + preflight headers)
# or attach a response headers policy with CORS config
```

### 7.2 The cached-CORS-response trap

**Symptom:** CORS works from one site and fails from another, seemingly at random.

**Cause:** CloudFront cached the response *including* its `Access-Control-Allow-Origin` header, and
the `Origin` request header isn't in the cache key. Site A's response gets served to site B.

**Fix — pick one:**

1. Include `Origin` in the **cache key** (via a cache policy header whitelist). Lower hit ratio, but
   correct.
2. Use a **response headers policy** with a fixed `Access-Control-Allow-Origin`. CloudFront applies
   it on the way out regardless of what's cached.
3. Use `Access-Control-Allow-Origin: *` for genuinely public assets.

Also set `Vary: Origin` at the origin — it's correct HTTP semantics and helps browser caches.

### 7.3 S3 CORS is separate from CloudFront CORS

```bash
aws s3api get-bucket-cors --bucket $BUCKET
```

Both layers must agree. S3's CORS config governs what S3 returns; CloudFront's response headers
policy governs what CloudFront adds or overrides.

---

## 8. Compression Problems

### 8.1 Content isn't being compressed

**Run the full checklist — all of these must be true:**

```bash
URL=https://cdn.example.com/app.js

# 1. Compress enabled on the behavior?
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.DefaultCacheBehavior.Compress' --output text     # true

# 2. Cache policy has the encoding flags?
aws cloudfront get-cache-policy --id $CP_ID \
  --query 'CachePolicy.CachePolicyConfig.ParametersInCacheKeyAndForwardedToOrigin.{Gzip:EnableAcceptEncodingGzip,Brotli:EnableAcceptEncodingBrotli}'

# 3. Content-Type compressible?  4. Size in range?  5. Already encoded?
curl -sSI -H 'Accept-Encoding: br,gzip' "$URL" \
  | grep -iE 'content-type|content-length|content-encoding'
```

| Requirement | Detail |
|---|---|
| `Compress: true` on the behavior | |
| Cache policy encoding flags on | Both gzip and brotli |
| Viewer sends `Accept-Encoding` | |
| **Compressible `Content-Type`** | text/*, application/json, application/javascript, application/xml, image/svg+xml. `binary/octet-stream` is **not**. |
| **Size between 1,000 and 10,000,000 bytes** | Files under 1 KB are never compressed |
| Origin didn't already set `Content-Encoding` | |
| `Content-Length` present | Chunked responses without length aren't compressed |

**The #1 cause: wrong `Content-Type` on S3 objects.**

```bash
# Find objects with a bad content type
aws s3api list-objects-v2 --bucket $BUCKET --query 'Contents[].Key' --output text \
  | tr '\t' '\n' | head -50 | while read -r K; do
    CT=$(aws s3api head-object --bucket $BUCKET --key "$K" --query ContentType --output text)
    echo "$CT  $K"
  done | grep -i 'binary/octet-stream'

# Fix them
aws s3 cp "s3://$BUCKET" "s3://$BUCKET" --recursive --exclude "*" --include "*.js" \
  --content-type "application/javascript" --metadata-directive REPLACE \
  --cache-control "public, max-age=31536000, immutable"
```

Then invalidate, because the wrong-typed response is already cached.

---

## 9. CloudFront Functions Problems

### 9.1 `The function failed validation` on create/update

| Cause | Fix |
|---|---|
| Function isn't named `handler` | Must be `function handler(event) { ... }` |
| Code exceeds 10 KB | Trim it, or move logic to Lambda@Edge |
| Used unsupported JS | No `fetch`, no `require`, no `import` (except `cloudfront` for KVS), no timers, no `eval`, no dynamic code |
| Used `async` on `cloudfront-js-1.0` | `async` requires runtime `cloudfront-js-2.0` |

```bash
wc -c my-function.js       # must be < 10240
```

### 9.2 Function isn't running

```bash
# Is it published to LIVE?
aws cloudfront describe-function --name my-fn --stage LIVE \
  --query 'FunctionSummary.FunctionMetadata.Stage' --output text

# Is it associated?
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.DefaultCacheBehavior.FunctionAssociations' --output json
```

**The most common miss:** you associated it to the **default** behavior but the request matches a
different behavior. Each behavior needs its own association.

**Second most common:** you updated the code but never re-ran `publish-function`. `update-function`
puts the code back in `DEVELOPMENT`.

### 9.3 `The CloudFront function associated with the CloudFront distribution is invalid or could not run`

**Cause:** a runtime error, or the function exceeded the ~1 ms compute budget.

```bash
# Check compute utilization with the test harness
aws cloudfront test-function --name my-fn --if-match "$FN_ETAG" --stage DEVELOPMENT \
  --event-object fileb://test-event.json \
  --query 'TestResult.{Compute:ComputeUtilization,Error:FunctionErrorMessage}'

# Runtime logs (us-east-1, ALWAYS)
aws logs tail /aws/cloudfront/function/my-fn --region us-east-1 --since 30m
```

**Common runtime errors:**

| Mistake | Correct form |
|---|---|
| `request.headers.host` returns an object | Use `request.headers.host.value` |
| Assigning a bare string to a header | `headers['x-foo'] = { value: 'bar' }` |
| Returning nothing | Must `return request` or `return response` |
| Modifying `event` instead of `event.request` | Return the request object itself |
| Header names in mixed case | CloudFront Functions header keys are **lowercase** |
| Using `Object.entries` / newer built-ins | Check runtime support; prefer simple loops |

### 9.4 KeyValueStore reads fail

```bash
# Is the store associated with the function?
aws cloudfront describe-function --name my-fn --stage LIVE \
  --query 'FunctionSummary.FunctionConfig.KeyValueStoreAssociations' --output json

# Is the store READY?
aws cloudfront describe-key-value-store --name my-kvs --query 'KeyValueStore.Status' --output text
```

**Gotchas:**
- The runtime must be `cloudfront-js-2.0` and the handler must be `async`.
- A missing key **throws**; always wrap `kvs.get()` in try/catch.
- One store per function.
- Key updates take a short time to propagate globally (seconds, not instant).

---

## 10. Lambda@Edge Problems

### 10.1 `The function execution role must be assumable with these principals: ["lambda.amazonaws.com","edgelambda.amazonaws.com"]`

```bash
aws iam get-role --role-name my-edge-role --query 'Role.AssumeRolePolicyDocument' --output json
```

**Fix:**

```bash
cat > /tmp/trust.json <<'EOF'
{ "Version": "2012-10-17", "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": ["lambda.amazonaws.com", "edgelambda.amazonaws.com"] },
    "Action": "sts:AssumeRole" }]}
EOF
aws iam update-assume-role-policy --role-name my-edge-role \
  --policy-document file:///tmp/trust.json
```

### 10.2 `InvalidLambdaFunctionAssociation` — must specify a version

**Cause:** you used `$LATEST` or an alias ARN.

```bash
# Publish a numbered version and use THAT ARN
aws lambda publish-version --region us-east-1 --function-name my-edge-fn \
  --query FunctionArn --output text
# arn:aws:lambda:us-east-1:111122223333:function:my-edge-fn:3  ← use this
```

Also confirm the function is in `us-east-1`. Lambda@Edge functions authored anywhere else can't be
associated.

### 10.3 503 `LambdaExecutionError` / `LambdaValidationError`

| Error | Cause |
|---|---|
| `LambdaExecutionError` | The function threw, timed out, or ran out of memory |
| `LambdaValidationError` | The returned object doesn't match the expected schema |
| `LambdaLimitExceeded` | Response body too large, or concurrency limits |
| `LambdaRuntimeError` | Handler not found, syntax error, bad module resolution |

**Timeouts:** 5 s for viewer triggers, 30 s for origin triggers. Exceeding them is a 503.

**Response schema requirements:**

```javascript
// Generating a response — status MUST be a STRING
return {
  status: '200',                                  // string, not number
  statusDescription: 'OK',
  headers: {
    'content-type': [{ key: 'Content-Type', value: 'text/html' }]   // array of {key,value}
  },
  body: '<html>...</html>'
};
```

**Body size limits:** ~1 MB for viewer-generated responses, ~1 MB body for origin responses.
Exceeding this gives `LambdaLimitExceeded`.

**Read-only headers you must not modify:** `Content-Length`, `Transfer-Encoding`, `Via`, `Connection`,
and the `X-Amz-Cf-*` set. Attempting to gives a validation error.

### 10.4 Can't find the logs

Logs go to the region **closest to the Regional Edge Cache that executed the function** — not
us-east-1.

```bash
for R in us-east-1 us-east-2 us-west-2 eu-west-1 eu-central-1 ap-south-1 \
         ap-southeast-1 ap-northeast-1 sa-east-1; do
  G=$(aws logs describe-log-groups --region "$R" \
      --log-group-name-prefix "/aws/lambda/us-east-1.my-edge-fn" \
      --query 'logGroups[].logGroupName' --output text 2>/dev/null)
  [ -n "$G" ] && echo "$R → $G"
done
```

Log group naming: `/aws/lambda/us-east-1.<function-name>`.

Also confirm the execution role has `AWSLambdaBasicExecutionRole` — without it there are no logs at
all.

### 10.5 `Lambda was unable to delete ... because it is a replicated function`

**This is normal and expected.**

```
1. Disassociate the function from EVERY cache behavior on EVERY distribution
2. Wait for the distributions to reach Deployed
3. Wait for replicas to be removed — this can take SEVERAL HOURS
4. Retry the delete
```

There is no way to force it. The function costs nothing while idle. Write a retry script and move on.

```bash
cat > retry-delete.sh <<'EOF'
#!/usr/bin/env bash
aws lambda delete-function --region us-east-1 --function-name my-edge-fn \
  && echo "deleted" || echo "still replicated — retry later"
EOF
chmod +x retry-delete.sh
```

### 10.6 Environment variables don't work

**They aren't supported in Lambda@Edge.** Options:

- Bake configuration into the deployment package.
- Read from a versioned config file in S3 (with a short in-memory cache).
- Use SSM Parameter Store / Secrets Manager at runtime (adds latency — cache the result).
- If the config is small and read-only, consider CloudFront Functions + KeyValueStore instead.

### 10.7 The function runs but nothing changes

**Check the trigger type:**

| Trigger | Fires on | Can modify |
|---|---|---|
| viewer-request | every request | request only |
| origin-request | **cache misses only** | request only |
| origin-response | **cache misses only** | response (gets cached) |
| viewer-response | every request | response (not cached) |

If you attached at `origin-response` and you're testing a cached URL, the function isn't running —
the cached response is being served. Bust the cache to test:

```bash
curl -sSI "https://cdn.example.com/page?bust=$RANDOM"
```

---

## 11. Signed URL & Signed Cookie Problems

### 11.1 403 `Missing Key-Pair-Id query parameter or cookie value`

**Cause:** the behavior has `TrustedKeyGroups` enabled but the request carries no signature.

Either sign the request, or check whether the behavior should be restricted at all:

```bash
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.CacheBehaviors.Items[].{Pattern:PathPattern,Trusted:TrustedKeyGroups}' \
  --output json
```

### 11.2 403 with a valid-looking signature

| Cause | Check |
|---|---|
| Clock skew | `date -u` on the signing server; expiry in the past |
| Wrong `Key-Pair-Id` | It must be the **public key ID** (`K...`), not the key group ID |
| Public key not in the key group attached to the behavior | `aws cloudfront get-key-group --id <id>` |
| Base64 not URL-safe | CloudFront uses `+`→`-`, `=`→`_`, `/`→`~` |
| Signed the wrong URL | The resource in the policy must match the request exactly, including scheme and host |
| Extra whitespace/newlines in the policy JSON | Strip them before signing |
| Signed with the wrong private key | Must pair with the uploaded public key |
| Wrong hash algorithm | CloudFront signatures use **SHA-1** with RSA PKCS#1 v1.5 |

```bash
# Verify your key pair matches
openssl rsa -in private_key.pem -pubout | diff - public_key.pem && echo "keys match"
```

### 11.3 Signed cookies don't work in the browser

| Cause | Fix |
|---|---|
| Cookie domain doesn't match | Set the cookie on the same domain as CloudFront, or a parent domain |
| `Secure` / `SameSite` blocking | Use `Secure; SameSite=None` for cross-site, and serve over HTTPS |
| Cookies not sent by the player/XHR | Set `withCredentials: true` / `crossorigin="use-credentials"` |
| Cookies sent but ignored | All **three** cookies are required: `CloudFront-Policy`, `CloudFront-Signature`, `CloudFront-Key-Pair-Id` |
| Signed URL and cookie both present | CloudFront prefers the query string; be consistent |

### 11.4 Legacy CloudFront key pairs no longer manageable

The old root-account "CloudFront key pairs" mechanism is deprecated. Migrate to **public keys + key
groups**, which any IAM principal can manage and which support rotation.

```bash
aws cloudfront create-public-key --public-key-config '{...}'
aws cloudfront create-key-group --key-group-config '{"Name":"kg","Items":["K..."]}'
# then set TrustedKeyGroups on the behavior and remove TrustedSigners
```

---

## 12. WAF & Security Problems

### 12.1 WAF is blocking legitimate users

**Symptom:** 403s with a CloudFront error page, and WAF metrics show blocks.

```bash
WAF_ARN=$(aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.WebACLId' --output text)

# Which rule is blocking?
aws wafv2 get-sampled-requests --region us-east-1 \
  --web-acl-arn "$WAF_ARN" --rule-metric-name CommonRuleSet --scope CLOUDFRONT \
  --time-window StartTime=$(( $(date +%s) - 3600 )),EndTime=$(date +%s) \
  --max-items 50 \
  --query 'SampledRequests[?Action==`BLOCK`].{URI:Request.URI,Rule:RuleNameWithinRuleGroup,IP:Request.ClientIP}' \
  --output table
```

**Notorious false positives:**

| Rule | Blocks |
|---|---|
| `SizeRestrictions_BODY` | File uploads and large JSON payloads (default 8 KB body inspection) |
| `CrossSiteScripting_BODY` | Rich text editors, markdown with HTML, code snippets |
| `SQLi_QUERYSTRING` | Search boxes containing quotes, `OR`, `--` |
| `NoUserAgent_HEADER` | Legitimate API clients and health checks |
| `GenericRFI_QUERYSTRING` | URLs passed as query parameters (redirect params, OAuth callbacks) |

**Fix — scope down the offending rule rather than disabling the whole group:**

```json
{
  "Name": "CommonRuleSet",
  "Priority": 0,
  "OverrideAction": { "None": {} },
  "Statement": {
    "ManagedRuleGroupStatement": {
      "VendorName": "AWS",
      "Name": "AWSManagedRulesCommonRuleSet",
      "RuleActionOverrides": [
        { "Name": "SizeRestrictions_BODY", "ActionToUse": { "Count": {} } },
        { "Name": "CrossSiteScripting_BODY", "ActionToUse": { "Count": {} } }
      ]
    }
  }
}
```

**Prevention:** always deploy new rules in `Count` mode, review sampled requests for a few days,
then promote to `Block`.

### 12.2 WAF web ACL won't attach

```
WAFNonexistentItemException / WAFInvalidParameterException
```

**Cause:** the web ACL wasn't created with `--scope CLOUDFRONT` in `us-east-1`. A `REGIONAL` web ACL
cannot be attached to CloudFront.

```bash
aws wafv2 list-web-acls --region us-east-1 --scope CLOUDFRONT --output table
```

You must also use the full **ARN** (not the ID) in `WebACLId` for WAFv2.

### 12.3 Rate limiting isn't working as expected

- The rate-based rule window is a rolling 5 minutes; the count is evaluated with a short delay.
- `AggregateKeyType: IP` uses the client IP CloudFront sees. Behind a corporate NAT, many users
  share one IP — set the limit accordingly.
- Use `FORWARDED_IP` with the `X-Forwarded-For` header only if there's a proxy in front of
  CloudFront.
- Minimum practical limits are relatively high; for stricter control combine with a custom rule.

### 12.4 Security headers aren't appearing

```bash
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.DefaultCacheBehavior.ResponseHeadersPolicyId' --output text
```

| Cause | Fix |
|---|---|
| Policy attached to the default behavior only | Attach it to every behavior |
| Origin already sends the header and `Override: false` | Set `Override: true` |
| Testing a cached response from before the change | Invalidate |
| Distribution still `InProgress` | Wait for `Deployed` |

---

## 13. CLI & API Errors

### 13.1 `PreconditionFailed`

```
An error occurred (PreconditionFailed): The request failed because it didn't meet the
preconditions in one or more request-header fields.
```

**Cause:** your `--if-match` ETag is stale. Someone (or another shell of yours) changed the
distribution.

**Fix:** always fetch config and ETag together, immediately before the update:

```bash
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/f.json
ETAG=$(jq -r '.ETag' /tmp/f.json)
jq '.DistributionConfig | <your changes>' /tmp/f.json > /tmp/n.json
aws cloudfront update-distribution --id $DIST_ID --if-match "$ETAG" \
  --distribution-config file:///tmp/n.json
```

### 13.2 `Unknown field: DistributionConfig` / `Parameter validation failed`

**Cause:** you passed the entire `get-distribution-config` output (which wraps the config in
`{"ETag":..., "DistributionConfig":{...}}`) instead of just the inner object.

```bash
# WRONG
aws cloudfront get-distribution-config --id $DIST_ID > cfg.json
aws cloudfront update-distribution --id $DIST_ID --if-match $E --distribution-config file://cfg.json

# RIGHT
jq '.DistributionConfig' cfg.json > inner.json
aws cloudfront update-distribution --id $DIST_ID --if-match $E --distribution-config file://inner.json
```

### 13.3 `InconsistentQuantities`

```
The value of Quantity and the size of Items don't match.
```

**Cause:** you added or removed an array element without updating the sibling `Quantity` field.

```bash
# Always update both
jq '.Aliases.Items += ["new.example.com"]
    | .Aliases.Quantity = (.Aliases.Items | length)' cfg.json > new.json

# Generic fixer for the common arrays
jq '
  .Origins.Quantity        = (.Origins.Items | length)
| .CacheBehaviors.Quantity = ((.CacheBehaviors.Items // []) | length)
| .Aliases.Quantity        = ((.Aliases.Items // []) | length)
| .CustomErrorResponses.Quantity = ((.CustomErrorResponses.Items // []) | length)
' cfg.json > new.json
```

### 13.4 `DistributionNotDisabled`

**Cause:** you tried to delete an enabled distribution.

```bash
# Disable, wait, then delete with the NEW ETag
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/f.json
ETAG=$(jq -r .ETag /tmp/f.json)
jq '.DistributionConfig | .Enabled = false' /tmp/f.json > /tmp/n.json
aws cloudfront update-distribution --id $DIST_ID --if-match "$ETAG" \
  --distribution-config file:///tmp/n.json
aws cloudfront wait distribution-deployed --id $DIST_ID
NEW=$(aws cloudfront get-distribution-config --id $DIST_ID --query ETag --output text)
aws cloudfront delete-distribution --id $DIST_ID --if-match "$NEW"
```

### 13.5 `ResourceInUse` when deleting a policy, OAC, key group, or function

**Cause:** something still references it. Detach first.

```bash
# Who is using this cache policy?
aws cloudfront list-distributions-by-cache-policy-id --cache-policy-id <id>
aws cloudfront list-distributions-by-origin-request-policy-id --origin-request-policy-id <id>
aws cloudfront list-distributions-by-response-headers-policy-id --response-headers-policy-id <id>
aws cloudfront list-distributions-by-key-group --key-group-id <id>
```

Deletion order that works: distributions → functions/policies/key groups → public keys → OACs.

### 13.6 `NoSuchDistribution` right after creating one

Propagation of the control-plane record can lag by a few seconds. Retry, or use
`aws cloudfront wait distribution-deployed`.

### 13.7 `InvalidArgument: The parameter Origin DomainName does not refer to a valid S3 bucket`

| Cause | Fix |
|---|---|
| You used the website endpoint with `S3OriginConfig` | Use `CustomOriginConfig` for website endpoints |
| Wrong region in the domain | Use `bucket.s3.<region>.amazonaws.com` |
| Bucket doesn't exist or is in another account without access | Verify with `aws s3api head-bucket` |
| Legacy global endpoint `bucket.s3.amazonaws.com` for a non-us-east-1 bucket | Use the regional endpoint |

---

## 14. Logging & Monitoring Problems

### 14.1 No access logs appearing

```bash
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.Logging' --output json
aws s3 ls "s3://$LOG_BUCKET/cloudfront/" --recursive | tail
```

| Cause | Fix |
|---|---|
| Logging never enabled | Enable it |
| Bucket ACLs disabled (v1 logging) | `put-bucket-ownership-controls` → `BucketOwnerPreferred` |
| Log bucket in an unsupported configuration | Use a standard S3 bucket in a supported region |
| No traffic yet | Generate some |
| Just waiting | Standard logs are best-effort and can lag by minutes to an hour |
| v2 delivery not fully wired | You need source + destination + `create-delivery` — all three |

```bash
aws logs describe-deliveries --region us-east-1 --output table
```

### 14.2 CloudWatch metrics are empty

**Cause:** you're looking in the wrong region. CloudFront metrics are **always in `us-east-1`** with
namespace `AWS/CloudFront`, and require **both** dimensions:

```bash
aws cloudwatch get-metric-statistics --region us-east-1 \
  --namespace AWS/CloudFront --metric-name Requests \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --start-time $(date -u -d '3 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Sum
```

`Name=Region,Value=Global` is mandatory and easy to forget.

### 14.3 `CacheHitRate` and `OriginLatency` don't exist

These are **additional metrics** and require a paid monitoring subscription:

```bash
aws cloudfront create-monitoring-subscription --distribution-id $DIST_ID \
  --monitoring-subscription 'RealtimeMetricsSubscriptionConfig={RealtimeMetricsSubscriptionStatus=Enabled}'
```

### 14.4 Athena returns zero rows

| Cause | Fix |
|---|---|
| `LOCATION` doesn't match the actual log prefix | `aws s3 ls s3://bucket/ --recursive | head` and correct it |
| Missing `skip.header.line.count = 2` | CloudFront logs have two header lines |
| Wrong field delimiter | Standard logs are **tab**-separated |
| Log format is w3c/JSON/Parquet, not plain | Your DDL must match the configured output format |
| Column count mismatch | CloudFront has added fields over time; align the DDL with your log version |

```sql
-- Sanity check
SELECT * FROM cf_logs LIMIT 5;
```

### 14.5 Real-time logs aren't arriving

```bash
aws cloudfront get-realtime-log-config --name my-rt-config --output json

# Is the config attached to a behavior?
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.DefaultCacheBehavior.RealtimeLogConfigArn' --output text

# Is the Kinesis stream healthy?
aws kinesis describe-stream-summary --stream-name my-stream
```

Common causes: the IAM role can't `kinesis:PutRecord`, the sampling rate is very low, or the stream
is throttled (add shards).

---

## 15. Deployment & Propagation Problems

### 15.1 Distribution stuck `InProgress`

```bash
aws cloudfront get-distribution --id $DIST_ID --query 'Distribution.Status' --output text
```

Normal is 1–5 minutes; occasionally longer. While `InProgress`, the **previous** configuration is
still being served — which explains "my change didn't take effect."

If it's been over an hour, open a support case with the distribution ID and a recent `X-Amz-Cf-Id`.

### 15.2 Changes work in some regions but not others

Partial propagation. Test from multiple PoPs:

```bash
for i in 1 2 3 4 5; do
  curl -sSI "https://cdn.example.com/?t=$RANDOM" | grep -iE 'x-amz-cf-pop|x-cache'
done
```

Wait for `Deployed`, then re-test.

### 15.3 Deploy pipeline race condition

**Symptom:** users occasionally get an `index.html` referencing assets that 404.

**Cause:** you invalidated `/index.html` before all the new assets finished uploading, or you
uploaded HTML before assets.

**Fix — enforce the order:**

```bash
set -euo pipefail

# 1. Assets FIRST (new hashed filenames — don't collide with anything live)
aws s3 sync ./dist/assets "s3://$BUCKET/assets" \
  --cache-control "public, max-age=31536000, immutable"

# 2. HTML SECOND
aws s3 cp ./dist/index.html "s3://$BUCKET/index.html" \
  --cache-control "no-cache" --content-type "text/html; charset=utf-8"

# 3. Invalidate the entry point LAST, and wait
INV=$(aws cloudfront create-invalidation --distribution-id "$DIST_ID" \
  --paths "/index.html" --query 'Invalidation.Id' --output text)
aws cloudfront wait invalidation-completed --distribution-id "$DIST_ID" --id "$INV"
```

Also: **never `--delete` old assets in the same deploy.** Users on the previous `index.html` still
need them. Delete them on the next deploy, or with a lifecycle rule.

---

## 16. Performance Problems

### 16.1 CloudFront is slower than going direct

**Diagnose:**

```bash
echo "--- through CloudFront ---"
curl -sS -o /dev/null -w 'dns=%{time_namelookup} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' \
  https://cdn.example.com/path

echo "--- direct to origin ---"
curl -sS -o /dev/null -w 'dns=%{time_namelookup} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' \
  https://origin.example.com/path
```

**Likely causes:**

| Cause | Fix |
|---|---|
| Everything is a cache miss | Fix the cache key (section 3.2) |
| You're testing from right next to the origin | CloudFront helps *distant* users; test from elsewhere |
| Origin Shield adds a hop on misses | Turn it off if hit ratio is already high |
| `OriginKeepaliveTimeout` too low → new TLS handshake per request | Raise it to 30–60 s |
| Price class excludes your users' region | Use PriceClass_All |
| Origin is slow | `OriginLatency` metric will show it |

### 16.2 First byte is slow but the transfer is fast

That's origin latency, not CloudFront. Check `OriginLatency` and the origin's own timing.

Enable the `Server-Timing` header via a response headers policy to get CloudFront's internal
breakdown in browser devtools:

```bash
curl -sSI https://cdn.example.com/ | grep -i server-timing
```

### 16.3 Large file downloads are slow or fail

| Cause | Fix |
|---|---|
| Range requests not supported by the origin | Ensure the origin honours `Range` (S3 does) |
| Origin read timeout too short for a slow generator | Raise `OriginReadTimeout` |
| Compression attempted on a huge file | Files over 10 MB aren't compressed — expected |
| Client on a lossy network | Enable HTTP/3 (QUIC handles loss much better) |

```bash
curl -sSI -H 'Range: bytes=0-1023' https://cdn.example.com/big.zip | grep -iE 'HTTP/|content-range'
# Expect: 206 Partial Content
```

---

## 17. Cost Surprises

### 17.1 The bill is higher than expected

**Diagnostic order:**

```bash
# 1. Cache hit rate — every miss is an origin request AND double the data path
aws cloudwatch get-metric-statistics --region us-east-1 \
  --namespace AWS/CloudFront --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 86400 --statistics Average

# 2. Where is the traffic going? Some regions cost several times more.
#   SELECT substr(x_edge_location,1,3) pop, SUM(sc_bytes)/1073741824 gb
#   FROM cf_logs GROUP BY 1 ORDER BY gb DESC;

# 3. What is consuming bandwidth?
#   SELECT cs_uri_stem, SUM(sc_bytes)/1073741824 gb, COUNT(*) hits
#   FROM cf_logs GROUP BY 1 ORDER BY gb DESC LIMIT 20;

# 4. Bots?
#   SELECT cs_user_agent, COUNT(*) n FROM cf_logs GROUP BY 1 ORDER BY n DESC LIMIT 20;
```

**Usual culprits:**

| Cause | Fix |
|---|---|
| Low cache hit ratio | Fix the cache key; this is almost always #1 |
| Compression disabled or broken | Fix Content-Types; enable gzip + brotli |
| Serving huge uncompressed images/video | WebP/AVIF, modern codecs, responsive sizes |
| Invalidating thousands of paths per deploy | Versioned filenames |
| Real-time logs at 100% sampling | Drop to 1–10% |
| Scrapers and AI crawlers | WAF rate limiting + bot control |
| Origin Shield on uncacheable content | Turn it off |
| Dedicated IP SSL or Anycast static IPs left enabled | Remove unless contractually required |
| PriceClass_All for a regional audience | PriceClass_200 or 100 |

### 17.2 Unexpected Lambda@Edge charges

Lambda@Edge bills per request **and** per GB-second, and viewer triggers run on **every** request
including cache hits. Moving a viewer-request function to a CloudFront Function typically cuts that
line item by ~85%.

```bash
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig.DefaultCacheBehavior.LambdaFunctionAssociations.Items[].EventType' \
  --output text
```

If it says `viewer-request` or `viewer-response` and the logic is simple, port it.

### 17.3 Charges from a distribution you thought you deleted

```bash
# Disabled but not deleted still shows up
aws cloudfront list-distributions \
  --query 'DistributionList.Items[].{Id:Id,Enabled:Enabled,Status:Status,Aliases:Aliases.Items}' \
  --output table
```

Also check: Lambda@Edge replicas, Kinesis streams from real-time logs, WAF web ACLs, and log buckets
accumulating objects with no lifecycle rule.

---

## 18. Quota Errors

| Error | Quota | Default | Adjustable |
|---|---|---|---|
| `TooManyDistributions` | Distributions per account | 500 | ✅ |
| `TooManyCacheBehaviors` | Cache behaviors per distribution | 75 | ✅ |
| `TooManyOrigins` | Origins per distribution | 25 | ✅ |
| `TooManyDistributionCNAMEs` | CNAMEs per distribution | 100 | ✅ |
| `TooManyCachePolicies` | Cache policies per account | 20 | ✅ |
| `TooManyOriginRequestPolicies` | Origin request policies per account | 20 | ✅ |
| `TooManyResponseHeadersPolicies` | Response headers policies per account | 20 | ✅ |
| `TooManyKeyGroups` | Key groups per account | 10 | ✅ |
| `TooManyInvalidationsInProgress` | Wildcard invalidations in progress | 15 | ❌ |
| `TooManyFunctionAssociations` | 1 function per event type per behavior | 1 | ❌ |
| `TooManyDistributionsAssociatedToKeyGroup` | — | — | ✅ |

```bash
# Check current usage vs quota
aws service-quotas list-service-quotas --service-code cloudfront \
  --query 'Quotas[].{Name:QuotaName,Value:Value,Adjustable:Adjustable}' --output table

# Request an increase
aws service-quotas request-service-quota-increase \
  --service-code cloudfront --quota-code L-XXXXXXXX --desired-value 800
```

**If you're hitting the distributions quota** because you run a multi-tenant SaaS with many customer
domains, the right answer isn't a quota increase — it's **distribution tenants (SaaS Manager)**, one
template with many tenants.

---

## Error Code Quick Index

| Status / Error | Most likely cause | Section |
|---|---|---|
| 400 Bad Request | Malformed request; header/URL over the size limit | 2.2 |
| 403 `AccessDenied` (XML) | S3 bucket policy / OAC missing or mismatched | 1.1 |
| 403 "The request could not be satisfied" | CNAME mismatch, geo block, WAF, or signed URL | 1.2, 1.3, 12.1, 11.1 |
| 403 `MethodNotAllowed` | Method not in `AllowedMethods` | 1.4 |
| 403 `Missing Key-Pair-Id` | Signed URL required but not supplied | 11.1 |
| 404 Not Found | Object missing, wrong key case, or `OriginPath` prefix | 1.6, 5.2 |
| 502 `OriginSSLHandshakeFailure` | Origin cert expired / mismatched / untrusted / no cipher overlap | 2.1 |
| 502 `LambdaValidationError` | Lambda@Edge returned an invalid object | 10.3 |
| 503 `CapacityExceeded` | RPS or data-rate quota hit | 2.4 |
| 503 `LambdaExecutionError` | Lambda@Edge threw or timed out | 10.3 |
| 504 Gateway Timeout | Origin too slow, or SG/NACL dropping traffic | 2.3 |
| `CNAMEAlreadyExists` | Alias already on another distribution | 4.3 |
| `InvalidViewerCertificate` | Cert not in us-east-1, not ISSUED, or doesn't cover the alias | 4.2 |
| `PreconditionFailed` | Stale ETag | 13.1 |
| `InconsistentQuantities` | `Quantity` doesn't match `Items` length | 13.3 |
| `DistributionNotDisabled` | Delete attempted while enabled | 13.4 |
| `ResourceInUse` | Something still references the resource | 13.5 |
| `InvalidLambdaFunctionAssociation` | Used `$LATEST` or an alias, or wrong region | 10.2 |
| `Lambda was unable to delete...replicated function` | Replicas not yet removed — wait | 10.5 |
| `TooManyDistributions` etc. | Quota | 18 |
| `WAFNonexistentItemException` | Web ACL not in `CLOUDFRONT` scope / us-east-1 | 12.2 |

---

## Escalating to AWS Support

### What to collect before you open a case

```bash
cat > /tmp/support-bundle.sh <<'EOF'
#!/usr/bin/env bash
DIST_ID="$1"
URL="$2"

echo "=== Distribution status ==="
aws cloudfront get-distribution --id "$DIST_ID" \
  --query 'Distribution.{Id:Id,Status:Status,Domain:DomainName,LastModified:LastModifiedTime}'

echo "=== Distribution config ==="
aws cloudfront get-distribution-config --id "$DIST_ID" --query 'DistributionConfig'

echo "=== Failing request headers (capture X-Amz-Cf-Id!) ==="
curl -sSI "$URL"

echo "=== Timing ==="
curl -sS -o /dev/null -w 'dns=%{time_namelookup} conn=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total} code=%{http_code}\n' "$URL"

echo "=== Recent 5xx rate ==="
aws cloudwatch get-metric-statistics --region us-east-1 \
  --namespace AWS/CloudFront --metric-name 5xxErrorRate \
  --dimensions Name=DistributionId,Value="$DIST_ID" Name=Region,Value=Global \
  --start-time $(date -u -d '3 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) --period 300 --statistics Average
EOF
chmod +x /tmp/support-bundle.sh
/tmp/support-bundle.sh E1234ABCDEFGH https://cdn.example.com/failing-path
```

### Include in the case

```
□ Distribution ID
□ The X-Amz-Cf-Id value from a FAILING request (this is the single most useful item)
□ The X-Amz-Cf-Pop value
□ Exact timestamp in UTC
□ The full request URL and method
□ What you expected vs what happened
□ Whether it's constant or intermittent, and what percentage of requests
□ Whether it's regional (which PoPs)
□ What changed recently and when
□ Steps you've already taken
```

`X-Amz-Cf-Id` lets AWS Support trace the exact request through their internal logs. Without it,
expect a much slower back-and-forth.

---

## Prevention Checklist

Most of this document exists because these steps were skipped.

```
CONFIGURATION
□ Distribution and bucket policy managed together in IaC
□ OAC (not OAI), with the AWS:SourceArn condition
□ Cache behaviors ordered most-specific first
□ Cache key contains exactly what varies the response — no User-Agent, no unbounded query strings
□ MinTTL = 0 unless there's a documented reason
□ TTL driven by origin Cache-Control headers, not console settings
□ Correct Content-Type on every uploaded object
□ Compression enabled and verified with curl

RELIABILITY
□ Origin timeouts tuned to the origin's real p99
□ OriginKeepaliveTimeout raised for chatty origins
□ Origin group configured if you have a secondary
□ Custom error responses configured (and /api/* on a separate behavior so real errors surface)
□ Config changes tested via a staging distribution before promoting

SECURITY
□ Origin unreachable directly (VPC origin, or prefix list + secret header)
□ Minimum TLS = TLSv1.2_2021
□ WAF attached, rules rolled out in COUNT first
□ Response headers policy with HSTS, CSP, nosniff, frame-deny
□ Signing private keys in Secrets Manager, never in git
□ Lambda@Edge / CF Function code reviewed — they see every request

OPERATIONS
□ Standard logging enabled with a lifecycle rule on the log bucket
□ Additional CloudWatch metrics subscription enabled on production
□ Alarms on 5xxErrorRate and CacheHitRate
□ Deploy pipeline: assets → HTML → invalidate /index.html only
□ Distribution ID, ARN, and OAC ID recorded in the runbook
□ Everything tagged (Environment, Owner, CostCenter)
```

---

**Still stuck?** Work back through [How to Diagnose Anything in Five Minutes](#how-to-diagnose-anything-in-five-minutes)
— nine times out of ten the four-layer isolation test tells you exactly which layer owns the problem,
and the fix follows from there.
