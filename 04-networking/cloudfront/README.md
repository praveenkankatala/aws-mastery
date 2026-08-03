# Amazon CloudFront — The Complete Practical Guide

> A from-zero-to-production learning repository for Amazon CloudFront: the theory explained in plain
> English, the architecture drawn out, and every setting explained in terms of *what it does*,
> *why it exists*, and *when you should touch it*.

---

## Repository Map

| File | What's inside | Read it when |
|------|---------------|--------------|
| **README.md** (you are here) | Concepts, architecture, deep-dives, configuration walkthrough, use cases, best practices | You want to *understand* CloudFront |
| **[commands-cheatsheet.md](./commands-cheatsheet.md)** | Every AWS CLI command, grouped by task, with real flags and JSON payloads | You're at a terminal and need the exact syntax |
| **[hands-on-labs.md](./hands-on-labs.md)** | 15 labs, from a first static site to edge compute, private origins, and Terraform | You learn by building |
| **[troubleshooting.md](./troubleshooting.md)** | Every error message you'll realistically hit, with root cause and fix | Something is broken and you need it fixed now |

---

## Table of Contents

1. [What Is CloudFront, Really?](#1-what-is-cloudfront-really)
2. [The Problem It Solves](#2-the-problem-it-solves)
3. [Prerequisites](#3-prerequisites)
4. [Vocabulary — Learn These 20 Words First](#4-vocabulary--learn-these-20-words-first)
5. [High-Level Architecture](#5-high-level-architecture)
6. [Service Flow — What Happens to a Single Request](#6-service-flow--what-happens-to-a-single-request)
7. [Core Features Deep-Dive](#7-core-features-deep-dive)
8. [Step-by-Step Configuration & Implementation Guide](#8-step-by-step-configuration--implementation-guide)
9. [Where to Use CloudFront — Target Use Cases](#9-where-to-use-cloudfront--target-use-cases)
10. [Reference Architectures](#10-reference-architectures)
11. [Security Model](#11-security-model)
12. [Observability, Logging & Monitoring](#12-observability-logging--monitoring)
13. [Pricing & Cost Optimization](#13-pricing--cost-optimization)
14. [Quotas & Hard Limits](#14-quotas--hard-limits)
15. [Best Practices & Anti-Patterns](#15-best-practices--anti-patterns)
16. [CloudFront vs. Everything Else](#16-cloudfront-vs-everything-else)
17. [Interview-Grade Q&A](#17-interview-grade-qa)
18. [Glossary](#18-glossary)

---

## 1. What Is CloudFront, Really?

Amazon CloudFront is **AWS's Content Delivery Network (CDN)** — a globally distributed fleet of
servers that sits *in front of* your application, catches incoming requests as close to the user as
physically possible, and either answers them from a local copy or fetches them from your real
server over an optimized private network path.

### The one-paragraph mental model

Imagine you run a bakery in Chennai and customers order from all over the world. Today, every single
order travels to Chennai and back — slow, expensive, and if 10,000 people order at once, your one
oven melts. Now imagine you open **small pickup counters in 440 cities**. Each counter keeps the
popular items in stock. A customer in Berlin walks to the Berlin counter, gets bread instantly, and
your Chennai kitchen never hears about it. Only when someone orders something unusual does the
counter phone the kitchen — and even then it uses a dedicated private phone line instead of the
public one.

CloudFront is those pickup counters. Your bakery is the **origin**. The counters are **edge
locations**. Keeping bread in stock is **caching**. The private phone line is the **AWS backbone
network**.

### The important nuance most tutorials skip

**CloudFront is not just a cache.** People assume CloudFront is only useful for static files. It is
equally valuable for content that is never cached at all, because it also gives you:

- **TLS termination at the edge** — the expensive TLS handshake finishes ~20 ms from the user instead
  of ~250 ms away, so even a fully dynamic API gets meaningfully faster.
- **Connection reuse to the origin** — CloudFront keeps warm, persistent connections to your origin,
  so a fresh viewer doesn't pay for a fresh origin handshake.
- **A security perimeter** — WAF, DDoS absorption, geo-blocking, signed URLs, mTLS, bot control.
- **A single front door** — one hostname routing to S3, ALB, Lambda, and API Gateway based on URL
  path, which lets you retire messy client-side routing.
- **Free data transfer from AWS origins** — S3 → CloudFront and ALB → CloudFront egress is $0.

So the correct framing is: **CloudFront is a programmable global reverse proxy that also caches.**

---

## 2. The Problem It Solves

| Problem without a CDN | How CloudFront fixes it |
|---|---|
| **Latency.** Physics caps you: light in fibre travels ~200 km/ms. Mumbai → Virginia round-trip is ~200 ms *before* your app does any work. A page needing 10 sequential round-trips is 2 seconds of pure distance. | Terminates the connection at an edge PoP typically 10–40 ms from the viewer. Cached responses never cross an ocean. |
| **Origin load.** 100,000 users requesting the same logo = 100,000 hits on your servers. | One origin fetch populates the cache; the other 99,999 are served at the edge. Cache hit ratios of 90–99% are normal for static content. |
| **Bandwidth cost.** EC2/ALB egress to the internet is billed at standard (high) data-transfer rates. | Origin → CloudFront transfer is free for AWS origins; CloudFront → viewer rates are lower than EC2 egress, and flat-rate plans cap it entirely. |
| **Traffic spikes.** A product launch or a viral post melts your Auto Scaling group before it can scale. | The edge absorbs the burst. Origin Shield collapses even cache misses into a single origin request. |
| **Attacks.** Volumetric DDoS saturates your pipe; L7 floods exhaust your app threads. | AWS Shield Standard is always on and free; the anycast edge absorbs volume; AWS WAF filters L7 at the edge before traffic reaches you. |
| **TLS complexity.** Certificates on every server, rotation, cipher config. | Free, auto-renewing ACM certificates terminated at the edge. Origin can even be plain HTTP inside a VPC. |
| **Global compliance/routing logic.** "Serve EU users the EU privacy banner." | CloudFront Functions / Lambda@Edge run your logic at the edge in microseconds. |

### The blunt version

If your content is **read far more than it's written**, and your users are **not all sitting next to
your origin**, CloudFront is close to free money. If neither is true, you may still want it purely as
a security and TLS layer.

---

## 3. Prerequisites

### Accounts and access

- An **AWS account** with the ability to create IAM roles and CloudFront distributions.
- IAM permissions. For learning, `CloudFrontFullAccess` plus `AmazonS3FullAccess` is fine. For
  production, scope down (see the least-privilege policy in the Security section).
- **AWS CLI v2** installed and configured (`aws --version` should print `aws-cli/2.x`).

```bash
aws configure
# AWS Access Key ID:     ...
# AWS Secret Access Key: ...
# Default region name:   ap-south-1     # your working region; CloudFront itself is global
# Default output format: json
```

- Confirm you're authenticated:

```bash
aws sts get-caller-identity
```

### Knowledge you should already have

You do **not** need to be an expert, but these will make everything click faster:

| Concept | Why it matters here |
|---|---|
| HTTP methods, status codes, headers | Cache behaviour is driven entirely by headers |
| `Cache-Control`, `Expires`, `ETag` | These control TTL more than the CloudFront console does |
| DNS: A, AAAA, CNAME, and Route 53 alias records | Required to point your domain at a distribution |
| TLS/SSL basics, SNI, certificate chains | Custom domains, origin protocol, mTLS |
| S3 buckets, bucket policies | The most common origin |
| Basic IAM (policies, principals, conditions) | OAC bucket policies use service principals + conditions |

### Tools that make life easier

```bash
# JSON wrangling — near-mandatory when using the CLI
sudo apt install jq -y            # Debian/Ubuntu
brew install jq                   # macOS

# HTTP inspection
curl --version
# Optional: httpie, dig / nslookup, openssl
```

### Cost warning (read this)

Everything in the labs fits comfortably in the AWS Free Tier if you clean up. The permanent
always-free tier includes **1 TB/month data transfer out, 10 million HTTP/HTTPS requests, and
2 million CloudFront Functions invocations**. The things that *will* cost you money if you're
careless:

- Leaving a **Lambda@Edge** function replicated across regions (delete carefully — see labs).
- **Anycast static IPs** (enterprise pricing, four figures per month). Don't enable them to "try it".
- **Field-level encryption** and **real-time logs** have per-request charges.
- Excessive **invalidations** beyond the 1,000 free paths per month.

The final lab is a complete teardown script. Run it.

---

## 4. Vocabulary — Learn These 20 Words First

Everything else in this repo assumes these. Read once, refer back as needed.

| Term | Plain-English meaning |
|---|---|
| **Distribution** | The CloudFront "thing" you create. It's a configuration object with a global hostname like `d111abcdef8.cloudfront.net`. Everything hangs off it. |
| **Edge location / PoP** | A physical data centre where CloudFront servers live. 750+ of them in 440+ cities. Where your users actually connect. |
| **Regional Edge Cache (REC)** | A larger, second-tier cache sitting between edge locations and your origin. Automatic and free. Catches objects evicted from the small edge caches. |
| **Origin** | The real source of truth for your content: an S3 bucket, ALB, EC2, API Gateway, Lambda Function URL, or any public HTTP server (even outside AWS). |
| **Origin group** | Two origins bundled with failover rules — primary and secondary. |
| **Viewer** | The end user / browser / mobile app / API client making the request. |
| **Cache behavior** | A rule that says "for URLs matching this path pattern, use these settings." Ordered, first match wins. |
| **Path pattern** | The glob that a cache behavior matches on: `/images/*`, `*.jpg`, `/api/*`. |
| **Default cache behavior** | The catch-all `*` behavior. Always exists, always evaluated last. |
| **Cache key** | The unique fingerprint CloudFront builds for a request to decide "have I seen this before?" By default: hostname + path. You choose what else goes in. |
| **Cache policy** | A reusable object defining the cache key + TTLs. |
| **Origin request policy** | A reusable object defining what gets *forwarded to the origin* (which may be more than what's in the cache key). |
| **Response headers policy** | A reusable object defining headers CloudFront *adds to the response* (CORS, HSTS, custom). |
| **TTL** | Time To Live — how long an object stays fresh in the cache before CloudFront must revalidate. |
| **Cache hit / miss** | Hit = served from cache. Miss = had to go to origin. Visible in the `X-Cache` response header. |
| **Invalidation** | Manually telling CloudFront "forget this path, it's stale." |
| **OAC (Origin Access Control)** | The modern mechanism that lets CloudFront — and only CloudFront — read a private S3 bucket. Replaces the legacy OAI. |
| **Signed URL / signed cookie** | A time-limited, cryptographically signed link that grants access to private content. |
| **CloudFront Functions** | Tiny JavaScript that runs at the edge in sub-millisecond time on viewer request/response. |
| **Lambda@Edge** | Full Lambda functions (Node.js/Python) that run at Regional Edge Caches, with network access, on all four trigger points. |

---

## 5. High-Level Architecture

### 5.1 The three-tier cache hierarchy

```
                            ┌──────────────────────────────────────┐
                            │            VIEWERS                    │
                            │  browsers • mobile apps • API clients │
                            └──────────────┬───────────────────────┘
                                           │  DNS resolves to nearest PoP
                                           │  (Anycast + latency-based routing)
                                           ▼
        ╔══════════════════════════════════════════════════════════════════════╗
        ║  TIER 1 — EDGE LOCATIONS  (750+ PoPs / 440+ cities)                  ║
        ║  ------------------------------------------------------------------  ║
        ║  • TLS termination           • CloudFront Functions (JS, <1 ms)      ║
        ║  • HTTP/2 + HTTP/3 (QUIC)    • AWS WAF evaluation                    ║
        ║  • Small, hot cache          • Shield Standard DDoS absorption       ║
        ║  • Geo restriction           • Signed URL / cookie validation        ║
        ╚═════════════════════════════════╤════════════════════════════════════╝
                                          │  CACHE MISS only
                                          ▼
        ╔══════════════════════════════════════════════════════════════════════╗
        ║  TIER 2 — REGIONAL EDGE CACHES  (13+ RECs, automatic, free)          ║
        ║  ------------------------------------------------------------------  ║
        ║  • Much larger cache than a PoP    • Lambda@Edge executes here       ║
        ║  • Catches objects evicted from the edge (long-tail content)         ║
        ║  • Fan-in: many PoPs → one REC → fewer origin requests               ║
        ╚═════════════════════════════════╤════════════════════════════════════╝
                                          │  STILL A MISS
                                          ▼
        ╔══════════════════════════════════════════════════════════════════════╗
        ║  TIER 3 — ORIGIN SHIELD  (optional, you pick the region, you pay)    ║
        ║  ------------------------------------------------------------------  ║
        ║  • ONE designated caching layer directly in front of the origin      ║
        ║  • Request collapsing: 1,000 simultaneous misses → 1 origin request  ║
        ╚═════════════════════════════════╤════════════════════════════════════╝
                                          │  AWS private backbone network
                                          ▼
        ┌──────────────────────────────────────────────────────────────────────┐
        │                              ORIGIN                                   │
        │  S3 bucket (OAC) │ ALB / NLB (VPC origin) │ EC2 │ API Gateway │       │
        │  Lambda Function URL │ MediaPackage │ any public HTTP server          │
        └──────────────────────────────────────────────────────────────────────┘
```

**Key insight:** the hierarchy exists to protect your origin. Each tier reduces the number of
requests reaching the next. A well-tuned distribution has a 90%+ hit ratio at Tier 1, and of the
remaining 10%, most never make it past Tier 2.

### 5.2 The control plane vs. the data plane

This trips people up constantly.

```
   CONTROL PLANE (global, but API endpoint lives in us-east-1)
   ───────────────────────────────────────────────────────────
   • CreateDistribution / UpdateDistribution / CreateInvalidation
   • ACM certificates for CloudFront MUST be in us-east-1
   • Lambda@Edge functions MUST be authored in us-east-1
   • WAF web ACLs for CloudFront MUST use scope = CLOUDFRONT (us-east-1)
   • Changes propagate to all PoPs — usually 1–5 minutes, occasionally longer

   DATA PLANE (truly global)
   ─────────────────────────
   • Actual request serving happens at 750+ PoPs
   • No region to choose, no capacity to provision
```

> **Rule of thumb:** if you're creating something *for* CloudFront and the CLI asks for a region,
> the answer is almost always `us-east-1`. The exception is the S3 bucket or ALB itself, which lives
> wherever you put it.

### 5.3 Anatomy of a distribution

```
DISTRIBUTION  (d111abcdef8.cloudfront.net)
│
├── General settings
│   ├── Price class ............... All / 200 / 100 (which PoPs to use)
│   ├── Alternate domain names .... cdn.example.com, www.example.com
│   ├── SSL certificate ........... ACM cert in us-east-1
│   ├── Security policy ........... TLSv1.2_2021 (minimum viewer TLS)
│   ├── Supported HTTP versions ... HTTP/2, HTTP/3
│   ├── Default root object ....... index.html
│   ├── Standard logging .......... S3 / CloudWatch Logs / Firehose
│   ├── IPv6 ...................... enabled
│   ├── WAF web ACL ............... optional
│   └── Anycast static IP list .... optional (enterprise)
│
├── ORIGINS  (up to 25)
│   ├── Origin "s3-assets"
│   │   ├── Domain ................ my-bucket.s3.ap-south-1.amazonaws.com
│   │   ├── Origin path ........... /public          (prefix added to every request)
│   │   ├── Origin access ......... OAC / public / legacy OAI
│   │   ├── Custom headers ........ X-Origin-Verify: <secret>
│   │   ├── Connection attempts ... 3      (1–3)
│   │   ├── Connection timeout .... 10 s   (1–10)
│   │   ├── Response timeout ...... 30 s   (1–120)
│   │   ├── Keep-alive timeout .... 5 s    (1–120)
│   │   └── Origin Shield ......... optional, choose region
│   └── Origin "alb-api"
│       ├── Protocol policy ....... HTTPS only / HTTP only / Match viewer
│       ├── Origin SSL protocols .. TLSv1.2
│       └── VPC origin ............ optional (private ALB/NLB/EC2)
│
├── ORIGIN GROUPS  (up to 10)
│   └── "api-failover": primary=alb-api, secondary=alb-api-dr
│       └── Failover status codes: 500, 502, 503, 504, 403, 404
│
├── CACHE BEHAVIORS  (up to 75, ORDERED — precedence 0 wins)
│   ├── [0] /api/*      → alb-api    │ CachingDisabled  │ AllViewerExceptHostHeader
│   ├── [1] /static/*   → s3-assets  │ CachingOptimized │ CORS-S3Origin
│   ├── [2] *.jpg       → s3-assets  │ CachingOptimized │ + Lambda@Edge resize
│   └── [*] DEFAULT     → s3-assets  │ CachingOptimized │ + CF Function rewrite
│
├── CUSTOM ERROR RESPONSES  (up to 25)
│   ├── 403 → /index.html with HTTP 200, TTL 10   (SPA routing)
│   └── 404 → /404.html   with HTTP 404, TTL 300
│
├── GEO RESTRICTIONS ......... allow-list or block-list of ISO country codes
├── CONTINUOUS DEPLOYMENT .... optional staging distribution + traffic policy
└── INVALIDATIONS ............ on-demand cache purges
```

---

## 6. Service Flow — What Happens to a Single Request

### 6.1 The complete request lifecycle

```
 ┌─────────┐
 │ VIEWER  │  GET https://cdn.example.com/images/hero.jpg
 └────┬────┘
      │ 1. DNS: cdn.example.com → CNAME/ALIAS → d111abcdef8.cloudfront.net
      │    Anycast + latency routing → nearest healthy PoP
      ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │ EDGE LOCATION                                                           │
 │                                                                         │
 │  2. TCP/QUIC handshake + TLS termination (cert must match SNI host)     │
 │                                                                         │
 │  3. Does the Host header match a CNAME on some distribution?            │
 │       NO  → 403 "The request could not be satisfied" (CNAMEMismatch)    │
 │                                                                         │
 │  4. GEO RESTRICTION check → blocked country? → 403                      │
 │                                                                         │
 │  5. AWS WAF web ACL evaluation → BLOCK / CAPTCHA / CHALLENGE / ALLOW    │
 │                                                                         │
 │  6. Match CACHE BEHAVIOR by path pattern (precedence order, first win)  │
 │                                                                         │
 │  7. ══ VIEWER REQUEST TRIGGER ══                                        │
 │       CloudFront Function  (JS 2.0, <1 ms, no network)   ─── or ───     │
 │       Lambda@Edge          (5 s max, network allowed)                   │
 │       Typical work: URI rewrite, redirect, auth header check, A/B split │
 │                                                                         │
 │  8. If behavior restricts access → validate SIGNED URL / SIGNED COOKIE  │
 │       invalid/expired → 403 MissingKey / policy error                   │
 │                                                                         │
 │  9. Build CACHE KEY from the cache policy:                              │
 │       host + path + [selected query strings] + [headers] + [cookies]    │
 │       + compression flag (gzip/br) + HTTP method                        │
 │                                                                         │
 │ 10. Look up cache key in the EDGE cache                                 │
 │       ┌──────────────────────────── HIT & FRESH ───────────────────┐    │
 │       │  → jump to step 17.  X-Cache: Hit from cloudfront          │    │
 │       └────────────────────────────────────────────────────────────┘    │
 └───────────────────────────────────┬─────────────────────────────────────┘
                                     │ MISS (or stale)
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ REGIONAL EDGE CACHE                                                      │
 │  11. Same cache-key lookup in the larger regional cache                  │
 │       HIT → return to PoP, populate edge cache, serve                    │
 │       X-Cache: RefreshHit / Hit from cloudfront                          │
 └───────────────────────────────────┬─────────────────────────────────────┘
                                     │ STILL A MISS
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │  12. ══ ORIGIN REQUEST TRIGGER ══  (Lambda@Edge only — runs at the REC)  │
 │        Can rewrite the URI, swap the origin, add auth headers,           │
 │        or short-circuit and generate a response without calling origin.  │
 │                                                                          │
 │  13. Build the ORIGIN REQUEST from the origin request policy:            │
 │        forwarded headers + query strings + cookies (superset of key)     │
 │        + origin custom headers + Sigv4 signature if OAC                  │
 │                                                                          │
 │  14. COLLAPSE simultaneous identical misses into one origin fetch        │
 │        (Origin Shield makes this global instead of per-REC)              │
 └───────────────────────────────────┬─────────────────────────────────────┘
                                     │ AWS backbone
                                     ▼
                       ┌──────────────────────────┐
                       │  ORIGIN SHIELD (opt.)    │──► ORIGIN (S3 / ALB / …)
                       └──────────────────────────┘
                                     │ 200 OK + Cache-Control: max-age=86400
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │  15. ══ ORIGIN RESPONSE TRIGGER ══  (Lambda@Edge only)                   │
 │        Modify status/headers/body. Common: image resize, custom errors.  │
 │                                                                          │
 │  16. STORE decision:                                                     │
 │        • Compute TTL: Cache-Control / Expires vs. Min/Default/Max TTL    │
 │        • Cacheable status codes: 200,203,204,206,300,301,302,303,304,    │
 │          307,308,404,405,410,414,416,451,500,502,503,504                 │
 │        • Do NOT cache if Cache-Control: no-store / private, or           │
 │          Set-Cookie present without an explicit cache policy allowing it │
 │        • Auto-compress with gzip/brotli if eligible                      │
 └───────────────────────────────────┬─────────────────────────────────────┘
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │  17. ══ VIEWER RESPONSE TRIGGER ══                                       │
 │        CloudFront Function or Lambda@Edge. Runs on BOTH hits and misses. │
 │        Typical: inject security headers, set cookies, strip Server hdr.  │
 │                                                                          │
 │  18. Apply RESPONSE HEADERS POLICY (CORS, HSTS, CSP, custom, removals)   │
 │                                                                          │
 │  19. Send to viewer with:                                                │
 │        X-Cache: Hit from cloudfront | Miss from cloudfront               │
 │        X-Amz-Cf-Pop: BOM51-P2         (which edge served it)             │
 │        X-Amz-Cf-Id: <opaque request id — quote this in support tickets>  │
 │        Age: <seconds this object has been cached>                        │
 └─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Trigger points at a glance

```
 VIEWER                      EDGE                    REGIONAL EDGE CACHE            ORIGIN
   │                          │                              │                        │
   │────── request ──────────►│                              │                        │
   │                     [viewer request]                    │                        │
   │                    CF Function ✔ / L@E ✔                │                        │
   │                          │──── cache miss ─────────────►│                        │
   │                          │                        [origin request]               │
   │                          │                        CF Function ✘ / L@E ✔          │
   │                          │                              │──── fetch ────────────►│
   │                          │                              │◄─── response ──────────│
   │                          │                        [origin response]              │
   │                          │                        CF Function ✘ / L@E ✔          │
   │                          │◄─────────────────────────────│                        │
   │                    [viewer response]                    │                        │
   │                    CF Function ✔ / L@E ✔                │                        │
   │◄───── response ──────────│                              │                        │
```

**Critical detail:** *origin request* and *origin response* triggers only fire on a **cache miss**.
*Viewer request* and *viewer response* fire on **every single request**, hit or miss. This is why
you put cheap, high-frequency logic (URI rewriting, redirects) in CloudFront Functions at the viewer
stage, and expensive, rare logic (image transformation) in Lambda@Edge at the origin stage.

---

## 7. Core Features Deep-Dive

### 7.1 Distributions

A **distribution** is the top-level configuration object. Creating one gives you a domain name of
the form `d1234abcdefgh.cloudfront.net` which is immediately usable over HTTPS.

**States:**

| Status | Meaning |
|---|---|
| `InProgress` | Config change is propagating to edge locations. The *previous* config is still serving. Usually 1–5 minutes. |
| `Deployed` | All edge locations have the new config. |
| `Enabled: false` | Distribution exists but returns errors. You must disable before you can delete. |

**The ETag dance.** Every distribution has an `ETag`. To update it you must:
1. `get-distribution-config` → gives you the config JSON *and* the current ETag.
2. Modify the JSON.
3. `update-distribution --if-match <ETag>`.

If someone else changed it in between, you get `PreconditionFailed` and must re-read. This is
optimistic concurrency control, and it's the #1 source of CLI frustration. See the cheatsheet for a
copy-paste-safe pattern.

> **Legacy note:** CloudFront used to offer "RTMP distributions" for Adobe Flash streaming. These
> were retired on 31 December 2020. If a tutorial mentions RTMP distributions, it is out of date.

---

### 7.2 Origins

An origin is where the real content lives. CloudFront supports:

| Origin type | Domain to use | Notes |
|---|---|---|
| **S3 (REST endpoint)** | `bucket.s3.<region>.amazonaws.com` | Use with **OAC**. Bucket stays private. Supports SSE-KMS. **No** directory-index behaviour — `/about/` will 404 unless you rewrite. |
| **S3 static website endpoint** | `bucket.s3-website-<region>.amazonaws.com` | Treated as a *custom* origin. Gives you index documents and redirect rules, but **cannot be private** and is **HTTP-only** (no HTTPS between CloudFront and S3). |
| **Application Load Balancer** | `my-alb-123.<region>.elb.amazonaws.com` | Public ALB, or private via **VPC origins**. |
| **Network Load Balancer** | `my-nlb-123.elb.<region>.amazonaws.com` | Via VPC origins, or public. |
| **EC2 instance** | Public DNS name or via VPC origin | Fine for labs; use an ALB in production. |
| **API Gateway** | `abc123.execute-api.<region>.amazonaws.com` | Set origin path to the stage, or use a custom domain. |
| **Lambda Function URL** | `<id>.lambda-url.<region>.on.aws` | Use OAC with signing service `lambda`. |
| **MediaPackage / MediaStore** | Provided endpoint | Live and VOD streaming. |
| **Any public HTTP server** | `origin.example.com` | On-prem, another cloud, anywhere reachable. |
| **Elastic Load Balancer in another account** | — | Supported; cross-account VPC origins are also supported. |

#### Origin settings that actually matter

| Setting | Range / options | What it does and when to change it |
|---|---|---|
| **Origin path** | e.g. `/prod` | Prefix silently prepended to every request path before it hits the origin. Great for API Gateway stages or serving from a subfolder. Must **not** end in `/`. |
| **Protocol policy** | HTTP Only / HTTPS Only / Match Viewer | `Match Viewer` means a viewer on HTTP causes CloudFront to talk HTTP to origin. Prefer **HTTPS Only** for public origins. S3 website endpoints force HTTP Only. |
| **Origin SSL protocols** | SSLv3, TLSv1, TLSv1.1, TLSv1.2 | Which protocols CloudFront will offer to the origin. Leave TLSv1.2 only. |
| **HTTP/HTTPS ports** | Any | Default 80/443. |
| **Connection attempts** | 1–3 (default 3) | How many times CloudFront retries connecting before failing over / erroring. |
| **Connection timeout** | 1–10 s (default 10) | Time to establish the TCP connection per attempt. |
| **Response timeout** | 1–120 s (default 30) | Also called *origin read timeout*. How long CloudFront waits for the first byte, and between bytes. **Raise this for slow APIs and long report generation.** Values above the default may need a quota increase. |
| **Keep-alive timeout** | 1–120 s (default 5) | How long an idle persistent connection to the origin is kept. Raising it improves connection reuse for chatty origins. Custom and VPC origins only. |
| **Custom headers** | Up to 10 name/value pairs | Injected into every origin request. The classic use: a shared secret (`X-Origin-Verify`) that your ALB rule requires, so nobody can bypass CloudFront and hit the ALB directly. |
| **Origin Shield** | Off / choose a region | Adds the third caching tier. See 7.14. |

#### Origin groups & automatic failover

An origin group pairs a **primary** and a **secondary** origin plus a list of failover status codes
(`400, 403, 404, 416, 500, 502, 503, 504` are selectable).

```
                     ┌── 200 OK ──────────────────► serve & cache
   CloudFront ──────►│
      request        └── 502/503/504 or timeout ──► retry against SECONDARY origin
```

**Rules you must know:**
- Failover applies only to `GET`, `HEAD`, and `OPTIONS` requests. `POST`/`PUT`/`PATCH`/`DELETE` are
  **not** retried (correct — they aren't idempotent).
- Failover also triggers on connection timeouts and connection failures, not just status codes.
- The failed response isn't cached; the successful secondary response is.
- This is *not* health checking. There's no background probe. Failover is reactive, per-request.
- For true active-active with health checks, put Route 53 health checks in front, or use two
  distributions.

---

### 7.3 Cache Behaviors & Path Patterns

Cache behaviors are the routing table of your distribution. **They are ordered and evaluated top to
bottom; the first match wins and evaluation stops.**

```
Precedence   Path pattern    Origin        Cache policy         Notes
──────────   ─────────────   ───────────   ──────────────────   ─────────────────────
    0        /api/v2/*       alb-api-v2    CachingDisabled      most specific first
    1        /api/*          alb-api       CachingDisabled
    2        /static/*       s3-assets     CachingOptimized     1 year TTL
    3        *.woff2         s3-assets     CachingOptimized     + CORS headers
    4        /admin/*        alb-admin     CachingDisabled      + WAF stricter rules
    *        (default)       s3-site       CachingOptimized     catch-all, always last
```

#### Path pattern syntax

| Pattern | Matches | Does not match |
|---|---|---|
| `/images/*` | `/images/a.jpg`, `/images/2024/b.png` | `/img/a.jpg`, `/images` (no trailing slash content) |
| `*.jpg` | `/a.jpg`, `/deep/path/b.jpg` | `/a.jpeg`, `/a.JPG` (**patterns are case-sensitive**) |
| `/api/*` | `/api/users`, `/api/` | `/apiv2/users` |
| `*` | everything | — |

**Gotchas:**
- `?` is not a wildcard. Only `*` is.
- Query strings are **not** part of the path pattern match.
- You cannot use a path pattern of `/` alone — use `*` (default behavior) or `/index.html`.
- Case sensitivity bites people constantly. `/Images/*` ≠ `/images/*`.

#### Per-behavior settings

| Setting | Options | Meaning |
|---|---|---|
| **Viewer protocol policy** | HTTP and HTTPS / Redirect HTTP to HTTPS / HTTPS Only | Almost always **Redirect HTTP to HTTPS** for websites, **HTTPS Only** for APIs. |
| **Allowed HTTP methods** | `GET,HEAD` / `GET,HEAD,OPTIONS` / `GET,HEAD,OPTIONS,PUT,POST,PATCH,DELETE` | Requests using a disallowed method get **403**. |
| **Cached HTTP methods** | `GET,HEAD` or `GET,HEAD,OPTIONS` | Only these are ever stored. `OPTIONS` caching is useful for CORS preflight. |
| **Restrict viewer access** | Yes/No + trusted key groups | Enables signed URL/cookie enforcement for this behavior. |
| **Compress objects automatically** | Yes/No | Gzip + Brotli. See 7.9. |
| **Field-level encryption** | config id | See 7.13. |
| **Real-time log config** | config id | See 12.3. |
| **Function associations** | CF Functions / Lambda@Edge | Up to 1 CF Function per event type; up to 1 Lambda@Edge per event type. |
| **gRPC support** | Yes/No | Enables gRPC-over-HTTP/2 proxying to the origin. |

---

### 7.4 The Cache Key — The Single Most Important Concept

The cache key is the fingerprint CloudFront computes for each request. **Two requests with the same
cache key get the same cached object.** Everything about hit ratio comes down to this.

#### Default cache key

```
CACHE KEY = <distribution / behavior>
          + <request path>
          + <compression flag: none | gzip | br>
```

That's it. By default **no query strings, no headers, no cookies** are included.

#### Why that matters — worked examples

```
Request A: /product?id=1        ┐
Request B: /product?id=2        ├─ Same cache key (query strings excluded)!
Request C: /product?id=3        ┘   All three viewers get whatever the first one cached.
                                    ← This is the classic "wrong product page" bug.

FIX: add `id` to the cache key via a cache policy query-string allow-list.
```

```
Request A: /home  with  Accept-Language: en    ┐
Request B: /home  with  Accept-Language: fr    ├─ Same cache key (headers excluded)
                                                ┘   French user gets English page.

FIX: include Accept-Language in the cache key — but ONLY if the origin actually varies on it.
```

#### The cardinality trade-off

```
   Cache key too NARROW                      Cache key too WIDE
   ────────────────────                      ──────────────────
   Hit ratio: 99% 🎉                          Hit ratio: 4% 💀
   Correctness: BROKEN                        Correctness: perfect
   (users see each other's content)           (origin is melting; CDN is pointless)

   Example: forgetting ?id                    Example: forwarding ALL headers
                                              (User-Agent alone has millions of values)
```

**The rule:** include in the cache key *exactly* the inputs the origin's response actually varies on
— no more, no less.

**Never include in the cache key:**
- `User-Agent` (millions of unique values → near-zero hit ratio). Use `CloudFront-Is-Mobile-Viewer`
  instead if you need device detection.
- `Referer`, `Date`, `Authorization` (unless you genuinely cache per-user, which you usually
  shouldn't), or any header with a timestamp/nonce.
- Analytics query strings: `utm_source`, `utm_medium`, `utm_campaign`, `fbclid`, `gclid`. Use a
  query-string **allow-list** (`include only specified`) rather than a deny-list so these are
  automatically ignored.

#### Cache key vs. origin request — the distinction people miss

These are **two different sets**:

```
   ┌─────────────────────────────────────────────────────────────┐
   │  ORIGIN REQUEST POLICY  (what gets FORWARDED to origin)      │
   │  ┌───────────────────────────────────────────────┐          │
   │  │  CACHE POLICY  (what's in the CACHE KEY)      │          │
   │  │  • path                                        │          │
   │  │  • query string: id, page                      │          │
   │  │  • header: Accept-Language                     │          │
   │  └───────────────────────────────────────────────┘          │
   │  + CloudFront-Viewer-Country  (origin logs it, doesn't       │
   │                                 change the response)         │
   │  + User-Agent                 (origin analytics only)        │
   │  + all cookies                (session tracking)             │
   └─────────────────────────────────────────────────────────────┘
```

The origin request policy must be a **superset** — anything in the cache key is automatically
forwarded. This lets you give the origin rich context (country, device, user agent) for logging or
personalization *without* destroying your hit ratio.

---

### 7.5 Cache Policies, Origin Request Policies & Response Headers Policies

These three reusable objects replaced the old "legacy cache settings" (forward query strings /
cookies / headers directly on the behavior). **Always use policies; the legacy settings are
deprecated and can't express modern options.**

#### Cache policy fields

```jsonc
{
  "Name": "MyAppCachePolicy",
  "DefaultTTL": 86400,      // used when origin sends NO Cache-Control/Expires
  "MinTTL": 1,              // floor — origin can never make it shorter than this
  "MaxTTL": 31536000,       // ceiling — origin can never make it longer than this
  "ParametersInCacheKeyAndForwardedToOrigin": {
    "EnableAcceptEncodingGzip": true,
    "EnableAcceptEncodingBrotli": true,
    "HeadersConfig":      { "HeaderBehavior": "whitelist",
                            "Headers": { "Quantity": 1, "Items": ["Accept-Language"] } },
    "CookiesConfig":      { "CookieBehavior": "none" },
    "QueryStringsConfig": { "QueryStringBehavior": "whitelist",
                            "QueryStrings": { "Quantity": 2, "Items": ["id","page"] } }
  }
}
```

**Behavior options** for headers / cookies / query strings:

| Value | Meaning |
|---|---|
| `none` | Not in the cache key (and, for origin request policies, not forwarded) |
| `whitelist` | Only the listed items |
| `allExcept` | Everything except the listed items (query strings & cookies) |
| `all` / `allViewer` | Everything (dangerous for cache keys) |

#### How TTL is actually decided

This is the exact precedence logic. Memorize it.

```
Does the origin response include Cache-Control: max-age / s-maxage, or Expires?

 ├── NO ──► CloudFront caches for DefaultTTL.
 │
 └── YES ─► CloudFront takes the origin's value, then CLAMPS it:
             effective_ttl = min( max( origin_value, MinTTL ), MaxTTL )

Special cases:
 • Cache-Control: s-maxage  overrides  max-age  for CloudFront (shared cache).
 • Cache-Control: no-cache  → object IS stored but must be revalidated every time.
 • Cache-Control: no-store  → object is NOT stored at all.
 • Cache-Control: private   → not stored by CloudFront (it's a shared cache).
 • MinTTL = 0 + no origin headers → CloudFront still uses DefaultTTL.
 • MinTTL > 0 with Cache-Control: no-cache → CloudFront will honour MinTTL and serve
   stale content. This surprises people. Keep MinTTL at 0 unless you mean it.
```

> **The single best practice in this entire document:** control TTL from the **origin** using
> `Cache-Control` headers, not from the CloudFront console. Then a deploy can change caching without
> a distribution update, and browsers, CloudFront, and any proxy in between all agree.

#### AWS managed cache policies

Don't build your own until you need to. Fetch current IDs with
`aws cloudfront list-cache-policies --type managed`.

| Managed policy | TTLs | Cache key | Use for |
|---|---|---|---|
| **CachingOptimized** | min 1 s, default 1 day, max 1 year | path only, gzip+brotli enabled | Static assets, S3 origins. The default choice. |
| **CachingOptimizedForUncompressedObjects** | same | same, compression flags off | Already-compressed content (images, video, zip) |
| **CachingDisabled** | all TTLs = 0 | path only | APIs, dynamic content, anything with per-user data |
| **Elemental-MediaPackage** | tuned for HLS/DASH | includes specific query strings | MediaPackage origins |
| **Amplify** | tuned for Amplify hosting | — | Amplify apps |
| **UseOriginCacheControlHeaders** | fully origin-driven | path only | You want origin `Cache-Control` to be authoritative |
| **UseOriginCacheControlHeaders-QueryStrings** | fully origin-driven | path + all query strings | Same, but query-string-varying content |

#### AWS managed origin request policies

| Managed policy | Forwards | Use for |
|---|---|---|
| **AllViewer** | all headers (incl. `Host`), cookies, query strings | Custom origins that need everything |
| **AllViewerExceptHostHeader** | everything except `Host` | **API Gateway, Lambda Function URLs, and most ALB origins** — these reject or misroute on a mismatched `Host` |
| **AllViewerAndCloudFrontHeaders-2022-06** | everything + `CloudFront-*` headers | You need viewer country/device at the origin |
| **CORS-S3Origin** | `Origin`, `Access-Control-Request-Headers`, `Access-Control-Request-Method` | S3 origins serving cross-origin assets |
| **CORS-CustomOrigin** | the three CORS headers | Non-S3 origins with CORS |
| **UserAgentRefererHeaders** | `User-Agent`, `Referer` | Origin-side analytics |
| **Elemental-MediaTailor-PersonalizedManifests** | tuned | MediaTailor |

> **Critical gotcha:** forwarding the `Host` header to an **S3 REST origin** breaks SigV4/OAC
> signing and returns 403. Never use `AllViewer` with S3+OAC — use `CORS-S3Origin` or none.

#### Response headers policies

These add/override headers on the way *out*, with no origin change and no Lambda needed.

```jsonc
{
  "Name": "SecureSiteHeaders",
  "SecurityHeadersConfig": {
    "StrictTransportSecurity": { "AccessControlMaxAgeSec": 63072000,
                                 "IncludeSubdomains": true, "Preload": true, "Override": true },
    "ContentTypeOptions":     { "Override": true },                 // nosniff
    "FrameOptions":           { "FrameOption": "DENY", "Override": true },
    "ReferrerPolicy":         { "ReferrerPolicy": "strict-origin-when-cross-origin",
                                "Override": true },
    "XSSProtection":          { "Protection": true, "ModeBlock": true, "Override": true },
    "ContentSecurityPolicy":  { "ContentSecurityPolicy": "default-src 'self'", "Override": true }
  },
  "CorsConfig": {
    "AccessControlAllowOrigins": { "Quantity": 1, "Items": ["https://app.example.com"] },
    "AccessControlAllowMethods": { "Quantity": 3, "Items": ["GET","HEAD","OPTIONS"] },
    "AccessControlAllowHeaders": { "Quantity": 1, "Items": ["*"] },
    "AccessControlAllowCredentials": false,
    "AccessControlMaxAgeSec": 600,
    "OriginOverride": true
  },
  "CustomHeadersConfig": {
    "Items": [{ "Header": "X-Powered-By", "Value": "AWS CloudFront", "Override": true }]
  },
  "RemoveHeadersConfig": {
    "Items": [{ "Header": "Server" }, { "Header": "X-Powered-By" }]
  },
  "ServerTimingHeadersConfig": { "Enabled": true, "SamplingRate": 1.0 }
}
```

**Managed response headers policies:** `SimpleCORS`, `CORS-With-Preflight`, `SecurityHeadersPolicy`,
`CORS-and-SecurityHeadersPolicy`.

**`Override: true` vs `false`** — if the origin already sends the header, `Override: true` replaces
it; `false` leaves the origin's value alone.

**Server-Timing header** is a hidden gem: enable it and every response carries detailed CloudFront
timing data (`cdn-cache-hit`, `cdn-rtt`, `cdn-downstream-fbl`) that you can read in browser devtools
or with `curl -I`. Set a low sampling rate in production.

---

### 7.6 TTLs, Freshness & Invalidation

#### Invalidation

```bash
aws cloudfront create-invalidation --distribution-id E123 --paths "/index.html" "/css/*"
```

| Fact | Detail |
|---|---|
| Cost | First **1,000 paths per month free**, then ~$0.005 per path. A wildcard `/*` counts as **one path**. |
| Speed | Typically 30 s – 5 minutes to complete globally. |
| Concurrency | Limited number of in-progress wildcard invalidations (15) and paths per request (3,000 for non-wildcard). |
| Scope | Case-sensitive, must start with `/`, and **must be URL-encoded** if the path contains special characters. |
| Query strings | `/file.jpg?v=1` invalidates only that exact key unless you use `/file.jpg*`. |

#### The better pattern: versioned filenames

```
❌  /css/app.css        →  deploy  →  invalidate /css/app.css  →  wait  →  hope
✅  /css/app.a3f9c1.css →  deploy  →  new URL, new cache key, instant, free, atomic
```

Every modern bundler (Webpack, Vite, esbuild, Next.js) does content hashing for you. With versioned
assets you can set `Cache-Control: public, max-age=31536000, immutable` and never invalidate again.

**The hybrid pattern most production sites use:**

```
/index.html          Cache-Control: no-cache            ← short/zero TTL, revalidated
/assets/*.[hash].js  Cache-Control: max-age=31536000, immutable
/assets/*.[hash].css Cache-Control: max-age=31536000, immutable
/api/*               Cache-Control: no-store
```

Deploy = upload new hashed assets, then overwrite `index.html`, then invalidate only `/index.html`.
One free invalidation path, atomic switch, no stale-asset mismatch.

#### Stale-while-revalidate and stale-if-error

CloudFront honours `stale-while-revalidate` and `stale-if-error` in `Cache-Control`. These let the
edge serve slightly-stale content instantly while refreshing in the background, and keep serving
stale content if the origin is down — a cheap and very effective resilience win.

```
Cache-Control: public, max-age=60, stale-while-revalidate=600, stale-if-error=86400
```

---

### 7.7 SSL/TLS & Custom Domains

#### The default certificate

Out of the box, your distribution serves HTTPS on `*.cloudfront.net` with an AWS-managed
certificate. Nothing to configure. This is fine for testing and internal use.

#### Using your own domain — the four required steps

```
 1. REQUEST A CERTIFICATE IN ACM — REGION MUST BE us-east-1
    ────────────────────────────────────────────────────────
    This is the #1 mistake. CloudFront can only attach certificates from
    N. Virginia, no matter where your users, buckets, or ALBs are.
    Request cdn.example.com, or a wildcard *.example.com.

 2. VALIDATE IT (DNS validation recommended)
    ────────────────────────────────────────
    ACM gives you a CNAME record. Add it to your hosted zone. If Route 53 is
    your DNS, ACM can create the record for you with one click / one CLI call.
    Status must reach ISSUED.

 3. ATTACH TO THE DISTRIBUTION
    ─────────────────────────
    • Add the domain to Alternate Domain Names (CNAMEs)
    • Select the ACM certificate
    • Choose SNI (free) unless you need legacy client support
    • Set minimum security policy (TLSv1.2_2021 recommended)

 4. POINT DNS AT CLOUDFRONT
    ───────────────────────
    Route 53:  A record, Alias = yes, target = the distribution (hosted zone
               Z2FDTNDATAQYW2 for all CloudFront distributions). Also add AAAA.
    Other DNS: CNAME cdn.example.com → d111abcdef8.cloudfront.net
               (apex domains need ALIAS/ANAME support from your provider)
```

#### SNI vs. dedicated IP

| | SNI (Server Name Indication) | Dedicated IP |
|---|---|---|
| Cost | **Free** | ~$600/month per distribution |
| Client support | Everything since ~2012 | Ancient clients (Windows XP/IE, Android 2.x) |
| Recommendation | **Use this** | Only for a documented legacy requirement |

#### Security policies (minimum viewer TLS version)

| Policy | Minimum TLS | Notes |
|---|---|---|
| `TLSv1.2_2021` | 1.2 | **Recommended default.** Modern cipher suites only. |
| `TLSv1.2_2019` | 1.2 | Slightly broader cipher list |
| `TLSv1.2_2018` | 1.2 | Broader still |
| `TLSv1.1_2016` | 1.1 | Deprecated; avoid |
| `TLSv1_2016` / `TLSv1` | 1.0 | Fails PCI DSS. Avoid. |
| `SSLv3` | SSLv3 | Only possible with dedicated IP. Never. |

#### CNAME rules you will trip over

- An alternate domain name can be attached to **only one distribution at a time, account-wide and
  globally**. Duplicates give `CNAMEAlreadyExists`.
- `aws cloudfront list-conflicting-aliases` tells you where a name is already in use.
- `associate-alias` moves an alias between distributions you own without downtime.
- The certificate must cover every alternate domain name. `*.example.com` covers `cdn.example.com`
  but **not** `example.com` itself or `a.b.example.com`.
- Wildcards match exactly one label.

#### Mutual TLS (mTLS)

CloudFront supports **viewer mTLS** — requiring clients to present a valid X.509 client certificate,
validated against a trust store you upload. Use it for B2B APIs, partner integrations, and IoT
fleets where API keys aren't strong enough.

#### HTTP versions

| Version | Status | Notes |
|---|---|---|
| HTTP/1.1 | Always on | Fallback |
| **HTTP/2** | On by default | Multiplexing, header compression. Keep it on. |
| **HTTP/3 (QUIC)** | Opt-in toggle | UDP-based, faster handshake, better on lossy mobile networks. Turn it on — clients that don't support it silently use HTTP/2. |
| gRPC | Opt-in per behavior | gRPC over HTTP/2 to the origin |

**Origin-side:** CloudFront speaks HTTP/1.1 to most origins; HTTP/2 to origin is supported for
specific configurations. Don't assume end-to-end HTTP/2.

---

### 7.8 Origin Access — Locking Down Your Origin

#### Origin Access Control (OAC) — the current standard

OAC makes CloudFront sign every origin request with **SigV4**, so the origin can verify that the
request genuinely came from your distribution.

```
   Viewer ──► CloudFront ──[SigV4-signed request]──► S3 bucket (PRIVATE, no public access)
                                                       │
   Viewer ──X── direct to bucket URL ─────────────────►│  403 AccessDenied ✔
```

**Supported OAC origin types:** `s3`, `mediastore`, `lambda` (Function URLs), `mediapackagev2`.

**Signing behaviours:**

| Value | Meaning |
|---|---|
| `always` | Always sign. **Use this.** |
| `never` | Don't sign (effectively disables OAC) |
| `no-override` | Sign only if the viewer request has no `Authorization` header |

**Signing protocol:** `sigv4` (the only option today).

**The required S3 bucket policy** — note the `AWS:SourceArn` condition, which prevents a
"confused deputy" where someone else's distribution reads your bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipalReadOnly",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::111122223333:distribution/E1234ABCDEFGH"
      }
    }
  }]
}
```

If the bucket uses **SSE-KMS**, you must also grant the CloudFront service principal
`kms:Decrypt` on the key, with the same `AWS:SourceArn` condition. Forgetting this yields a
confusing 403.

#### OAC vs OAI

| | OAI (legacy) | **OAC (current)** |
|---|---|---|
| Identity | Special CloudFront user, referenced in bucket policy | Service principal + SigV4 signing |
| SSE-KMS support | ❌ | ✅ |
| Dynamic requests (`POST`, `PUT`) to S3 | ❌ | ✅ |
| All AWS Regions incl. newer ones | ❌ | ✅ |
| MediaStore / Lambda URLs | ❌ | ✅ |
| Recommended | No — migrate | **Yes** |

OAI still works and existing setups won't break, but AWS has stopped adding features to it. Migrate.

#### VPC Origins — private ALB/NLB/EC2

VPC origins let CloudFront reach resources in **private subnets with no public IP and no internet
gateway path**. AWS creates a managed VPC origin endpoint (an ENI-backed construct) inside your VPC.

```
   Internet ──► CloudFront edge ──► AWS backbone ──► VPC Origin Endpoint
                                                       │ (private subnet)
                                                       ▼
                                             Internal ALB / NLB / EC2
                                             SG allows only the VPC origin
```

**Why this is a big deal:** previously, the only ways to force traffic through CloudFront were
(a) a shared-secret custom header your ALB checked, or (b) security-group rules built from the
`CLOUDFRONT_ORIGIN_FACING` managed prefix list. Both leak. With VPC origins the origin simply has no
public route at all. Cross-account VPC origins are also supported.

#### The shared-secret pattern (still useful for non-VPC origins)

```
CloudFront origin custom header:  X-Origin-Verify: 7f3c9a2e-...secret...
              │
              ▼
ALB listener rule:  IF http-header X-Origin-Verify == <secret>  THEN forward
                    ELSE return fixed response 403
```

Rotate the secret with a scheduled Lambda that updates both the distribution and the ALB rule.
Combine with the `com.amazonaws.global.cloudfront.origin-facing` managed prefix list in your
security group so only CloudFront IPs can even connect.

---

### 7.9 Compression

CloudFront can compress objects on the fly with **gzip** and **Brotli**.

**All of these must be true:**
1. "Compress objects automatically" is `true` on the cache behavior.
2. The cache policy has `EnableAcceptEncodingGzip` / `EnableAcceptEncodingBrotli` set to `true`.
3. The viewer sends `Accept-Encoding: gzip` and/or `br`.
4. The origin response has a **compressible `Content-Type`** (text/*, application/json,
   application/javascript, application/xml, image/svg+xml, and a long documented list).
5. The response has a `Content-Length` header, and the size is **between 1,000 and 10,000,000 bytes**.
6. The origin response is **not already compressed** (no `Content-Encoding` header).
7. The response is not a `Range` request or chunked without length.

**Why your compression "isn't working"** is almost always #4, #5, or #6 — most often a missing or
generic `Content-Type` on S3 objects (`binary/octet-stream` is not compressible), or a file under
1,000 bytes.

> Brotli typically saves another 15–20% over gzip on text. Enable both; CloudFront picks per viewer.

---

### 7.10 Edge Compute — CloudFront Functions vs Lambda@Edge

#### The comparison table you should screenshot

| | **CloudFront Functions** | **Lambda@Edge** |
|---|---|---|
| Runtime | JavaScript (ECMAScript 5.1 + subset of ES6+), `cloudfront-js-2.0` | Node.js and Python (current supported versions) |
| Where it runs | **Edge locations** (all 750+) | **Regional Edge Caches** (13+) |
| Trigger points | viewer request, viewer response | **all four**: viewer request/response, origin request/response |
| Max execution time | **< 1 ms** (hard enforced) | 5 s (viewer triggers) / 30 s (origin triggers) |
| Memory | ~2 MB | 128 MB – 10 GB |
| Max package size | **10 KB** | 1 MB (viewer) / 50 MB (origin) |
| Network access | ❌ none | ✅ full (can call AWS APIs, databases, other services) |
| File system access | ❌ | ✅ `/tmp` |
| Access to request body | ❌ | ✅ (with `include-body`) |
| Access to response body | ❌ | ✅ (origin response, generate/modify) |
| Key-value data | ✅ **CloudFront KeyValueStore** | via DynamoDB/S3 over the network |
| Cost | ~$0.10 per million invocations | ~$0.60 per million + GB-second duration |
| Free tier | 2 M invocations/month, permanent | — |
| Logging | `console.log` → CloudWatch Logs in **us-east-1** | CloudWatch Logs in the **region closest to execution** |
| Authored in | Any region (global service) | **us-east-1 only**, published with a numbered version |
| Scale | Millions of RPS | Thousands of RPS |

#### Choosing between them — a decision tree

```
Do you need to run on a cache MISS only (origin request/response)?
 ├── YES ────────────────────────────────────► Lambda@Edge
 └── NO
      │
      Do you need network calls, the request/response body, or > 10 KB of code?
       ├── YES ───────────────────────────────► Lambda@Edge
       └── NO
            │
            Is it URI rewriting, redirects, header manipulation, simple auth
            checks, cache-key normalization, or A/B cookie assignment?
             └── YES ──────────────────────────► CloudFront Functions ✔ (cheaper, faster)
```

#### CloudFront Functions — canonical examples

**URI rewrite for directory indexes (fixes the `/about/` → 404 problem on S3 REST origins):**

```javascript
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
```

**Redirect and normalize (canonical host, force trailing slash):**

```javascript
function handler(event) {
    var request = event.request;
    var host = request.headers.host.value;

    if (host === 'example.com') {
        return {
            statusCode: 301,
            statusDescription: 'Moved Permanently',
            headers: { 'location': { value: 'https://www.example.com' + request.uri } }
        };
    }
    return request;
}
```

**Simple auth gate (basic auth for a staging site):**

```javascript
function handler(event) {
    var request = event.request;
    var headers = request.headers;
    var expected = 'Basic dXNlcjpwYXNzd29yZA==';   // base64("user:password")

    if (!headers.authorization || headers.authorization.value !== expected) {
        return {
            statusCode: 401,
            statusDescription: 'Unauthorized',
            headers: { 'www-authenticate': { value: 'Basic realm="Staging"' } }
        };
    }
    return request;
}
```

**Cache-key normalization (strip tracking params so they don't fragment the cache):**

```javascript
function handler(event) {
    var qs = event.request.querystring;
    ['utm_source','utm_medium','utm_campaign','utm_term','utm_content','fbclid','gclid']
        .forEach(function (k) { delete qs[k]; });
    return event.request;
}
```

**Add security headers on viewer response:**

```javascript
function handler(event) {
    var response = event.response;
    response.headers['strict-transport-security'] =
        { value: 'max-age=63072000; includeSubDomains; preload' };
    response.headers['x-content-type-options'] = { value: 'nosniff' };
    response.headers['x-frame-options']        = { value: 'DENY' };
    return response;
}
```

*(For headers specifically, a response headers policy is usually simpler and free — prefer it.)*

#### CloudFront KeyValueStore

A global, low-latency key-value store readable from CloudFront Functions in **microseconds**, with
no network call. Up to 5 MB per store; one store per function.

```javascript
import cf from 'cloudfront';
const kvsHandle = cf.kvs();

async function handler(event) {
    const request = event.request;
    try {
        const target = await kvsHandle.get(request.uri);   // e.g. redirect map
        if (target) {
            return { statusCode: 302, statusDescription: 'Found',
                     headers: { location: { value: target } } };
        }
    } catch (e) { /* key miss — fall through */ }
    return request;
}
```

Perfect for: redirect maps, feature flags, A/B test allocations, blocked-path lists, tenant routing
tables. Update the data without redeploying the function.

#### Lambda@Edge — canonical examples

**Origin request: route to a different origin based on country**

```javascript
export const handler = async (event) => {
    const request = event.Records[0].cf.request;
    const country = request.headers['cloudfront-viewer-country']?.[0]?.value;

    if (country === 'DE' || country === 'FR') {
        request.origin.custom.domainName = 'eu-origin.example.com';
        request.headers.host = [{ key: 'host', value: 'eu-origin.example.com' }];
    }
    return request;
};
```

**Origin response: serve a custom 404 body**

```javascript
export const handler = async (event) => {
    const response = event.Records[0].cf.response;
    if (response.status === '404') {
        response.status = '404';
        response.statusDescription = 'Not Found';
        response.body = '<html><body><h1>Not here</h1></body></html>';
        response.headers['content-type'] = [{ key: 'Content-Type', value: 'text/html' }];
    }
    return response;
};
```

**Lambda@Edge quirks that will bite you:**
- Must be in **us-east-1** and you associate a **numbered version**, never `$LATEST` or an alias.
- The execution role trust policy must include **both** `lambda.amazonaws.com` and
  `edgelambda.amazonaws.com`.
- **No environment variables.** Bake config in, or fetch it at runtime, or use a versioned config file.
- Deleting is a two-step wait: disassociate from all behaviors, wait for replicas to be removed
  (can take **several hours**), *then* delete the function.
- Logs land in the CloudWatch region nearest the executing REC — check `ap-south-1`, `eu-west-1`,
  `us-east-1`, etc., not just one.
- Cold starts are real; keep the package tiny.

---

### 7.11 Signed URLs & Signed Cookies — Private Content

Use these when content must be restricted but you don't want a login on every asset: paid videos,
premium downloads, customer-specific documents, time-limited share links.

```
   ┌──────────────┐  1. authenticate    ┌────────────────────┐
   │   Viewer     │────────────────────►│  Your application  │
   │              │◄────────────────────│  (signs with the   │
   └──────┬───────┘  2. signed URL      │   PRIVATE key)     │
          │             or Set-Cookie   └────────────────────┘
          │ 3. GET /video.mp4?Expires=...&Signature=...&Key-Pair-Id=...
          ▼
   ┌──────────────────────────────────────────────────────┐
   │  CloudFront  — verifies signature with the PUBLIC key │
   │  in the trusted key group attached to the behavior    │
   └──────────────────────────────────────────────────────┘
          │ valid & unexpired → serve      invalid/expired → 403
          ▼
      S3 / origin
```

#### Signed URLs vs signed cookies

| | Signed URL | Signed cookie |
|---|---|---|
| Grants access to | **One object** | **Many objects** matching a path pattern |
| Where the token lives | Query string | `CloudFront-Policy`, `CloudFront-Signature`, `CloudFront-Key-Pair-Id` cookies |
| Works with RTMP-style players / native clients | ✅ | Sometimes not |
| Good for | A single download link, a share link | An entire subscription library, HLS/DASH streams with many segments |
| Cache impact | Query strings excluded from cache key by default → still cacheable ✔ | Cookies excluded from cache key by default → still cacheable ✔ |

#### Canned vs custom policy

| | Canned policy | Custom policy |
|---|---|---|
| Expiry time | ✅ | ✅ |
| Start time (not valid before) | ❌ | ✅ |
| Source IP / CIDR restriction | ❌ | ✅ |
| Wildcard resource path | ❌ (one exact URL) | ✅ |
| URL length | Shorter | Longer |

#### Setup steps

```
1. Generate an RSA key pair (2048-bit)
     openssl genrsa -out private_key.pem 2048
     openssl rsa -pubout -in private_key.pem -out public_key.pem
2. Upload the PUBLIC key to CloudFront   → returns a Key-Pair-Id (e.g. K2JCJMDEHXQW5F)
3. Create a KEY GROUP containing that public key
4. On the cache behavior: Restrict viewer access = Yes, trusted key group = <group>
5. Store the PRIVATE key in Secrets Manager / Parameter Store. Never commit it.
6. Your app signs URLs/cookies with the private key.
```

> **Legacy warning:** the old mechanism used "CloudFront key pairs" created by the AWS account root
> user. That is deprecated. Use **public keys + key groups**, which any IAM principal can manage and
> which support rotation (put two keys in a group, rotate, remove the old one).

#### Signing in code

```python
# pip install cryptography botocore
from botocore.signers import CloudFrontSigner
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
from datetime import datetime, timedelta

def rsa_signer(message):
    with open('private_key.pem', 'rb') as f:
        private_key = serialization.load_pem_private_key(f.read(), password=None,
                                                         backend=default_backend())
    return private_key.sign(message, padding.PKCS1v15(), hashes.SHA1())

signer = CloudFrontSigner('K2JCJMDEHXQW5F', rsa_signer)
url = signer.generate_presigned_url(
    'https://cdn.example.com/private/video.mp4',
    date_less_than=datetime.utcnow() + timedelta(hours=2))
print(url)
```

```javascript
// npm i @aws-sdk/cloudfront-signer
import { getSignedUrl, getSignedCookies } from "@aws-sdk/cloudfront-signer";

const url = getSignedUrl({
  url: "https://cdn.example.com/private/video.mp4",
  keyPairId: "K2JCJMDEHXQW5F",
  privateKey: process.env.CF_PRIVATE_KEY,
  dateLessThan: new Date(Date.now() + 2 * 3600 * 1000).toISOString(),
});

const cookies = getSignedCookies({
  url: "https://cdn.example.com/premium/*",
  keyPairId: "K2JCJMDEHXQW5F",
  privateKey: process.env.CF_PRIVATE_KEY,
  dateLessThan: new Date(Date.now() + 24 * 3600 * 1000).toISOString(),
});
```

---

### 7.12 Geo Restriction

Two modes, both based on a **MaxMind GeoIP** lookup of the viewer's IP:

| Mode | Behaviour |
|---|---|
| **Allow-list** (whitelist) | Only listed ISO 3166-1 alpha-2 countries can access |
| **Block-list** (blacklist) | Listed countries get **403** |

**Limitations you must communicate to stakeholders:**
- Country-level only. No state/city granularity (use Lambda@Edge with `CloudFront-Viewer-Country-Region`
  or a GeoIP database for finer control).
- Trivially bypassed with a VPN or proxy. This is a **compliance/licensing tool, not a security
  control**.
- Applies to the whole distribution, not per behavior. For per-path rules, use **AWS WAF geo match
  rules**, which *are* per-rule and can be combined with other conditions.
- Accuracy is ~99.8% but not perfect.

**CloudFront geo headers available to your origin and edge functions:**
`CloudFront-Viewer-Country`, `-Country-Name`, `-Country-Region`, `-Country-Region-Name`,
`-City`, `-Postal-Code`, `-Latitude`, `-Longitude`, `-Time-Zone`, `-Metro-Code`, `-Address`,
`-ASN`, `-Http-Version`, `-TLS`, `-JA3-Fingerprint`.

---

### 7.13 Field-Level Encryption

Encrypts **specific form fields** with your own public key at the edge, so sensitive data (card
numbers, national IDs) stays encrypted through every downstream hop and only your designated
application — holding the private key — can decrypt it.

```
   Viewer POSTs  { name: "Asha", card: "4111111111111111" }
                                  │
                                  ▼
   CloudFront edge encrypts ONLY the `card` field with your RSA public key
                                  │
                                  ▼
   Origin / load balancer / logs / middleware see:
                 { name: "Asha", card: "AXsgh8Kj...base64 ciphertext..." }
                                  │
                                  ▼
   Only the payment microservice with the private key can decrypt
```

- Works on `POST` requests with `application/x-www-form-urlencoded`.
- Up to 10 fields per configuration, RSA 2048.
- Requires a **field-level encryption profile** (public key + field patterns) and a **configuration**
  (maps content types to profiles), attached to a cache behavior.
- Priced per 10,000 requests.
- Reduces PCI DSS scope meaningfully. Genuinely useful, rarely used because people don't know it exists.

---

### 7.14 Origin Shield

An **additional, single, designated caching layer** in a region you choose, placed directly in front
of your origin.

```
   WITHOUT Origin Shield                      WITH Origin Shield
   ─────────────────────                      ──────────────────
   PoP Mumbai   ─┐                            PoP Mumbai   ─┐
   PoP London   ─┼─► 13 RECs ─► 13 requests   PoP London   ─┼─► RECs ─► SHIELD ─► 1 request
   PoP Tokyo    ─┤     to your origin          PoP Tokyo    ─┤              (ap-south-1)
   PoP São Paulo─┘                             PoP São Paulo─┘
```

**Benefits:**
- **Request collapsing across all RECs** — a viral object causes one origin fetch, not thirteen.
- Better cache hit ratio for long-tail content.
- Lower origin load and lower origin egress cost.
- Reduced origin operational load during invalidation storms and cache-cold events.

**Costs and caveats:**
- Charged per 10,000 requests that reach the shield.
- Adds a hop → marginally higher latency on a cold miss.
- **Choose the region closest to your origin**, not to your users.
- Not worth it if your hit ratio is already very high or your origin is trivially scalable (S3 with
  low traffic).
- Bad fit for genuinely uncacheable, per-user dynamic content — you're paying for a hop that never
  serves a hit.

---

### 7.15 Custom Error Pages

Map an origin/CloudFront error status to your own page, and control how long the error is cached.

| Field | Meaning |
|---|---|
| HTTP error code | 400, 403, 404, 405, 414, 416, 500, 501, 502, 503, 504 |
| Response page path | e.g. `/errors/404.html` — must be reachable through the distribution |
| HTTP response code | What the viewer actually sees (can differ from the origin's) |
| Error caching min TTL | Default 10 s. Set to 0 while debugging; raise to protect a struggling origin. |

**The SPA routing pattern** (React/Vue/Angular with client-side routes):

```
403 → /index.html → response code 200 → min TTL 10
404 → /index.html → response code 200 → min TTL 10
```

S3 REST origins return **403** (not 404) for missing keys when the bucket is private, which is why
you must map both. Mapping to `200` matters: returning `index.html` with a 404 status breaks SEO and
some routers.

> A cleaner alternative for SPAs: a **CloudFront Function** that rewrites any non-file URI to
> `/index.html`. It avoids masking genuine errors from your API paths.

---

### 7.16 Continuous Deployment (Staging Distributions)

Test a configuration change against real production traffic before committing to it.

```
                          ┌──────────────────────────────┐
        production   ┌───►│  PRIMARY distribution (live) │
        traffic ─────┤    └──────────────────────────────┘
                     │
                     │    ┌──────────────────────────────┐
                     └───►│  STAGING distribution        │  ◄── new config under test
                          └──────────────────────────────┘
                Split by:  • WEIGHT (up to 15%, optional session stickiness)
                           • HEADER (Aws-Cf-Cd-<name>: <value>) for targeted testing
```

Workflow:
1. `create-distribution --staging` (or "Create staging distribution" in the console) — clones the
   primary.
2. Change whatever you want on the staging distribution.
3. Create a **continuous deployment policy** with either a header-based or weight-based traffic config.
4. Validate with real traffic and CloudWatch metrics.
5. **Promote** — copies the staging config onto the primary and detaches the policy.

**Quotas & constraints:**
- 20 staging distributions and 20 continuous deployment policies per account.
- Maximum **15%** of traffic to staging in weight-based mode.
- Session stickiness idle duration 300–3600 s.
- Staging distributions cannot have their own alternate domain names.
- If the primary uses OAC to S3, **update the bucket policy to also allow the staging distribution**
  — otherwise the staging traffic all 403s.

---

### 7.17 Multi-Tenant Distributions (SaaS Manager)

For SaaS platforms that serve **hundreds or thousands of customer domains** and previously had to
create one distribution per customer (bumping into the distributions-per-account quota).

```
   ┌─────────────────────────────────────────────────────────┐
   │  DISTRIBUTION TEMPLATE  (shared config, cache behaviors) │
   │  with parameters:  {{tenantName}}, {{originPath}}         │
   └───────────────┬─────────────────────────────────────────┘
                   │
     ┌─────────────┼─────────────┬─────────────┐
     ▼             ▼             ▼             ▼
  TENANT A      TENANT B      TENANT C      TENANT D
  acme.com      globex.io     initech.dev   umbrella.co
  own cert      own cert      own cert      own cert
  own WAF opt.  own params    own params    own params
```

- One template, many **distribution tenants**, each with its own domain, certificate, and parameter
  values.
- Certificate management and validation are handled per tenant.
- Pricing is tiered per tenant, with a free allowance for the first several tenants.
- Dramatically simpler than managing thousands of distributions.

---

### 7.18 Anycast Static IPs

By default CloudFront's IP ranges change over time — fine for the public internet, a problem when a
partner's firewall requires a small, permanent IP allow-list.

An **Anycast static IP list** gives you a fixed set of IPs, globally anycast, that you attach to your
distributions. It is an **enterprise-priced feature** (four figures per month per list). Use it only
when a partner or regulator demands a static allow-list.

For most people the correct answer is instead: use the `com.amazonaws.global.cloudfront.origin-facing`
managed prefix list on the *origin* side, and let viewers connect to anycast DNS normally.

---

### 7.19 Price Classes

| Price class | Edge locations used | When |
|---|---|---|
| **All** | Every PoP worldwide, including South America, Australia, India, Middle East, Africa | Global audience, best performance |
| **200** | Excludes the most expensive regions (South America, Australia/NZ) | Mostly NA/EU/Asia audience |
| **100** | North America and Europe only | Regional audience, cheapest |

If a viewer is outside your price class, they're still served — just from a farther PoP with higher
latency. Nothing breaks. This is purely a cost/performance dial.

---

### 7.20 Everything Else Worth Knowing

| Feature | What it does |
|---|---|
| **Default root object** | Serves `/index.html` when the viewer requests `/`. **Only applies to the root** — `/blog/` still needs a rewrite. |
| **Range requests / byte-range** | Supported and cached in segments. Essential for video seeking and resumable downloads. CloudFront can fetch just the needed range from the origin. |
| **Chunked transfer encoding** | Supported for origin responses. |
| **WebSockets** | Supported end-to-end (`ws://`/`wss://`). Requires the origin to support them; connections bypass the cache. |
| **gRPC** | Supported per cache behavior over HTTP/2. |
| **IPv6** | Toggle per distribution. Enable it — it's free and some mobile networks are IPv6-only. |
| **Smooth Streaming** | Legacy Microsoft Smooth Streaming support on a cache behavior. |
| **`CloudFront-*` request headers** | Rich viewer metadata (device type, country, TLS version, JA3 fingerprint) forwarded to your origin without you doing GeoIP yourself. |
| **Managed prefix list** | `com.amazonaws.global.cloudfront.origin-facing` — use in security groups so only CloudFront edge IPs can reach your origin. |
| **CloudFront Savings Bundle** | Commit to a monthly spend for up to ~30% discount on CloudFront + WAF + Shield. |
| **Flat-rate pricing plans** | Free / Pro / Business / Premium tiers bundling CDN + WAF + DDoS + DNS + logging into one predictable monthly price per distribution, with no overage charges. |
| **AI activity dashboard** | Visibility into AI crawler and agent traffic, so you can distinguish legitimate agents from scrapers. |

---

## 8. Step-by-Step Configuration & Implementation Guide

This is the "read once, then follow" version. The full click-by-click and command-by-command
walkthroughs live in **[hands-on-labs.md](./hands-on-labs.md)**.

### Phase 0 — Decide before you click anything

Answer these five questions first. Ninety percent of CloudFront pain comes from skipping them.

```
1. WHAT IS THE ORIGIN?
   S3 (private + OAC) │ ALB (public or VPC origin) │ API Gateway │ Lambda URL │ external

2. WHAT VARIES THE RESPONSE?
   nothing → cache aggressively
   query strings → allow-list exactly which ones
   headers → allow-list exactly which ones (never User-Agent)
   cookies → allow-list, or accept a low hit ratio
   user identity → don't cache; use CachingDisabled

3. HOW LONG CAN CONTENT BE STALE?
   forever (hashed assets) │ minutes (product pages) │ never (checkout, APIs)
   → decide the Cache-Control values your ORIGIN will send

4. WHO IS ALLOWED TO SEE IT?
   everyone │ specific countries │ signed URL holders │ WAF-filtered │ mTLS clients

5. DOES ANY LOGIC NEED TO RUN AT THE EDGE?
   no │ URI rewrite/redirect → CloudFront Function │ heavy transform → Lambda@Edge
```

### Phase 1 — Prepare the origin

**If S3:**
```
□ Create the bucket in the region nearest your build pipeline
□ Block ALL public access (leave the default on — OAC does not need public access)
□ Upload content with CORRECT Content-Type on every object
□ Set Cache-Control at upload time:
     hashed assets → public, max-age=31536000, immutable
     index.html    → no-cache
□ Enable versioning if you want rollback
```

**If ALB/EC2:**
```
□ Health checks green
□ TLS certificate on the ALB (or plan to use a VPC origin over HTTP internally)
□ Decide how you'll block direct access:  VPC origin (best) │ shared header │ prefix list
□ Confirm the app doesn't depend on the viewer's Host header, or plan to forward it
```

**If API Gateway / Lambda URL:**
```
□ Note the stage name → becomes the origin path
□ Plan to use the AllViewerExceptHostHeader origin request policy
□ For Lambda URLs: auth type AWS_IAM + OAC with signing service `lambda`
```

### Phase 2 — Certificate and DNS (only if using a custom domain)

```
□ Request an ACM certificate IN us-east-1
□ Validate via DNS (add the CNAME record; Route 53 can do this automatically)
□ Wait for status = ISSUED   (usually minutes; can take longer)
```

### Phase 3 — Create the distribution

```
□ Origin domain, origin path, origin access (OAC), origin custom headers
□ Default cache behavior:
     viewer protocol policy = Redirect HTTP to HTTPS
     allowed methods        = GET,HEAD  (or add OPTIONS/POST as needed)
     cache policy           = CachingOptimized (static) or CachingDisabled (dynamic)
     origin request policy  = matched to the origin type
     response headers policy= SecurityHeadersPolicy (or your own)
     compress objects       = yes
□ Additional behaviors for /api/*, /static/*, etc. — ORDER MATTERS
□ Alternate domain names + ACM certificate + TLSv1.2_2021
□ Price class
□ Default root object = index.html
□ Custom error responses (SPA: 403/404 → /index.html → 200)
□ Standard logging → S3 or CloudWatch Logs
□ WAF web ACL (scope = CLOUDFRONT)
□ Geo restriction if required
```

### Phase 4 — Wire up permissions

```
□ S3 bucket policy allowing cloudfront.amazonaws.com with AWS:SourceArn condition
□ KMS key policy if the bucket uses SSE-KMS
□ ALB listener rule checking the shared secret header (if applicable)
□ Security group referencing the CloudFront managed prefix list (if applicable)
```

### Phase 5 — Point DNS

```
□ Route 53: A + AAAA alias records → distribution (zone Z2FDTNDATAQYW2)
□ Other DNS: CNAME → dxxxx.cloudfront.net
□ Verify: dig cdn.example.com  → should resolve to CloudFront IPs
```

### Phase 6 — Verify (do not skip)

```bash
# 1. It responds and TLS is valid
curl -sSI https://cdn.example.com/

# 2. Second request is a HIT
curl -sSI https://cdn.example.com/ | grep -i x-cache      # expect: Hit from cloudfront

# 3. HTTP redirects to HTTPS
curl -sSI http://cdn.example.com/ | head -1               # expect: 301

# 4. Compression is applied
curl -sSI -H 'Accept-Encoding: br,gzip' https://cdn.example.com/app.js | grep -i content-encoding

# 5. Origin is NOT reachable directly
curl -sSI https://my-bucket.s3.ap-south-1.amazonaws.com/index.html   # expect: 403

# 6. Security headers present
curl -sSI https://cdn.example.com/ | grep -iE 'strict-transport|x-content-type|x-frame'

# 7. Which PoP served it, and the request id for support
curl -sSI https://cdn.example.com/ | grep -iE 'x-amz-cf-pop|x-amz-cf-id'
```

### Phase 7 — Operationalize

```
□ CloudWatch alarms: 5xxErrorRate, OriginLatency, CacheHitRate
□ Enable additional metrics if you need cache-hit-rate and per-status-code breakdowns
□ Athena table over the access logs
□ Deployment pipeline: sync → invalidate /index.html only
□ Document the distribution ID, ARN, and OAC ID in your runbook
□ Tag the distribution (Environment, Owner, CostCenter)
```

---

## 9. Where to Use CloudFront — Target Use Cases

### 9.1 Static website / SPA hosting  ⭐ most common

```
   Route 53 ──► CloudFront ──[OAC]──► S3 (private)
                    │
                    ├─ CF Function: URI rewrite for directory indexes
                    ├─ Custom errors: 403/404 → /index.html (200)
                    └─ Response headers policy: HSTS, CSP, nosniff
```
**Why:** cheapest possible hosting, no servers, global performance, free TLS, near-infinite scale.
**Watch for:** the `/about/` → 404 problem, and forgetting `Content-Type` on uploads.

### 9.2 Accelerating a dynamic web application

```
   Viewers ──► CloudFront ──► ALB ──► ECS/EKS/EC2
                   │
                   ├─ /static/*  CachingOptimized  (long TTL)
                   ├─ /api/*     CachingDisabled   (AllViewerExceptHostHeader)
                   └─ *          short TTL or origin-driven Cache-Control
```
**Why:** even with zero caching you win on TLS termination, connection reuse, and the AWS backbone.
Typical improvement for dynamic content: 20–40% lower TTFB for distant users.

### 9.3 API acceleration and protection

```
   Clients ──► CloudFront ──► API Gateway / ALB
                   │
                   ├─ AWS WAF: rate limiting, SQLi/XSS rules, bot control
                   ├─ CachingDisabled for writes; short TTL for read-heavy GETs
                   └─ CF Function: reject malformed requests before they cost you anything
```
**Bonus:** caching even 30 seconds of a hot `GET /products` endpoint can cut origin load by 95%
during a traffic spike.

### 9.4 Video streaming (VOD and live)

```
   VOD:   S3 (HLS/DASH segments) ──► CloudFront ──► players
   Live:  MediaLive ──► MediaPackage ──► CloudFront ──► players
                            │
                            ├─ Signed cookies for subscriber access
                            ├─ Range requests for seeking
                            └─ Long TTL on segments, short TTL on manifests
```
**Key detail:** manifests (`.m3u8`, `.mpd`) need a short TTL; segments (`.ts`, `.m4s`) can be cached
for a long time. Use separate cache behaviors by extension.

### 9.5 Large file and software distribution

```
   S3 ──► CloudFront ──► global users
             │
             ├─ Signed URLs with expiry for licensed downloads
             ├─ Range requests → resumable downloads
             └─ Price Class All for worldwide reach
```

### 9.6 Security perimeter for a legacy or on-prem origin

```
   Internet ──► CloudFront (WAF + Shield + TLS 1.3) ──► on-prem origin over HTTPS
                                                          (firewall allows only
                                                           CloudFront prefix list)
```
**Why:** you get a modern TLS front end, a WAF, and DDoS absorption in front of a system you can't
easily modernize.

### 9.7 Multi-origin single-domain architecture

```
                          ┌── /            → S3 (marketing site)
                          ├── /app/*       → S3 (React SPA)
   cdn.example.com ──────►├── /api/*       → ALB (backend)
                          ├── /auth/*      → Cognito / API Gateway
                          └── /docs/*      → S3 (docs bucket)
```
**Why:** no CORS problems, one certificate, one domain, one WAF, one log stream. This is one of
CloudFront's most underrated capabilities.

### 9.8 Blue/green and canary releases

Use continuous deployment (staging distribution) for CDN-config changes, and origin-side weighted
target groups for application changes.

### 9.9 SaaS multi-tenant delivery

Distribution tenants (SaaS Manager) — one template, thousands of customer domains, per-tenant certs.

### 9.10 When **not** to use CloudFront

| Situation | Better choice |
|---|---|
| Purely internal app, all users in one region, no internet exposure | Internal ALB directly |
| You need TCP/UDP acceleration for non-HTTP protocols (gaming, VoIP, MQTT) | **AWS Global Accelerator** |
| You need static anycast IPs and instant regional failover for a TCP service | **AWS Global Accelerator** |
| You only need faster *uploads* to a single S3 bucket | **S3 Transfer Acceleration** |
| Every response is unique per user, tiny, and your origin is already at the edge of your users | Probably still worth it for TLS + WAF, but measure |

---

## 10. Reference Architectures

### 10.1 Production static site (the canonical setup)

```
                    ┌──────────────┐
                    │  Route 53    │  cdn.example.com → A/AAAA ALIAS
                    └──────┬───────┘
                           ▼
   ┌───────────────────────────────────────────────────────────────┐
   │                       CLOUDFRONT                               │
   │  ACM cert (us-east-1) • TLSv1.2_2021 • HTTP/2 + HTTP/3        │
   │  ┌──────────────────────────────────────────────────────────┐ │
   │  │ AWS WAF  (CLOUDFRONT scope)                              │ │
   │  │   Core rule set • rate limit 2000/5min • bot control      │ │
   │  └──────────────────────────────────────────────────────────┘ │
   │  Behaviors:                                                    │
   │   [0] /assets/*  CachingOptimized  1y immutable                │
   │   [*] default    CachingOptimized  + CF Function (URI rewrite) │
   │  Response headers policy: HSTS + CSP + nosniff + frame DENY    │
   │  Custom errors: 403,404 → /index.html (200, TTL 10)            │
   │  Standard logs → S3 (partitioned) → Athena                     │
   └───────────────────────────┬───────────────────────────────────┘
                               │ OAC (SigV4)
                               ▼
                    ┌──────────────────────┐
                    │  S3 bucket (PRIVATE) │  versioning on
                    │  Block Public Access │  SSE-S3 or SSE-KMS
                    └──────────────────────┘
```

### 10.2 Full-stack app with private origins

```
   Viewers
      │
      ▼
   CloudFront ──[WAF]
      │
      ├── /static/*  ──[OAC]──────────────► S3 (private)
      │
      └── /*, /api/* ──[VPC Origin]──────► Internal ALB (private subnets)
                                              │
                                              ├─► ECS Fargate tasks
                                              └─► targets in private subnets,
                                                  NO internet gateway route
```
Origin is completely unreachable from the internet. There is no bypass path to close.

### 10.3 Global multi-region active-active

```
                       CloudFront (origin group)
                          │              │
              primary ────┘              └──── secondary
                 │                              │
        ┌────────▼─────────┐          ┌─────────▼────────┐
        │ ap-south-1 ALB   │          │ eu-west-1 ALB    │
        │ ECS + Aurora     │◄────────►│ ECS + Aurora     │
        │                  │  Global   │                  │
        └──────────────────┘  Database └──────────────────┘

   Failover status codes: 500, 502, 503, 504
   + Route 53 health checks for DNS-level failover of the origin hostnames
```

### 10.4 Video on demand with subscriber access

```
   Subscriber logs in ──► App ──► issues SIGNED COOKIES for /premium/*
                                        │
   Player requests manifest + segments ─┘
                    │
                    ▼
   CloudFront  (behavior /premium/*: restrict viewer access = trusted key group)
     ├── *.m3u8 / *.mpd  → TTL 2 s   (manifest changes)
     └── *.ts / *.m4s    → TTL 1 year (segments are immutable)
                    │
                    ▼
              S3 (private, OAC)  or  MediaPackage
```

---

## 11. Security Model

### 11.1 Defense in depth — the layers, in order

```
  1. VIEWER TLS        TLSv1.2_2021 minimum, HSTS with preload, optional mTLS
  2. GEO RESTRICTION   country allow/block (licensing, not security)
  3. AWS SHIELD        Standard is free & always on; Advanced adds DRT + cost protection
  4. AWS WAF           managed rule groups, rate limiting, bot control, CAPTCHA, custom rules
  5. SIGNED URL/COOKIE cryptographic per-object or per-path access control
  6. EDGE FUNCTIONS    auth header checks, token validation, request sanitization
  7. FIELD ENCRYPTION  end-to-end protection of specific sensitive fields
  8. ORIGIN ACCESS     OAC (SigV4) / VPC origins / shared secret / prefix list
  9. RESPONSE HEADERS  CSP, HSTS, X-Frame-Options, nosniff, Referrer-Policy
 10. LOGGING & AUDIT   access logs, real-time logs, CloudTrail for config changes
```

### 11.2 AWS WAF with CloudFront

```
□ Web ACL scope MUST be CLOUDFRONT (created in us-east-1)
□ Recommended baseline managed rule groups:
     AWSManagedRulesCommonRuleSet
     AWSManagedRulesKnownBadInputsRuleSet
     AWSManagedRulesAmazonIpReputationList
     AWSManagedRulesSQLiRuleSet          (if you have a database)
     AWSManagedRulesLinuxRuleSet / UnixRuleSet / PHPRuleSet as applicable
□ Add a rate-based rule (e.g. 2000 requests / 5 min / IP)
□ Deploy in COUNT mode first, review the logs, THEN switch to BLOCK
□ Enable WAF logging to Kinesis Firehose / S3 / CloudWatch Logs
```

**WAF geo match vs CloudFront geo restriction:** WAF rules are per-rule and combinable
(`country = X AND uri starts with /admin`), and can CAPTCHA rather than hard-block. CloudFront's
geo restriction is distribution-wide and free. Use CloudFront's for blunt licensing rules and WAF's
for anything nuanced.

### 11.3 Least-privilege IAM for a CI/CD deploy role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SyncSiteAssets",
      "Effect": "Allow",
      "Action": ["s3:PutObject","s3:DeleteObject","s3:ListBucket","s3:GetObject"],
      "Resource": ["arn:aws:s3:::my-site-bucket","arn:aws:s3:::my-site-bucket/*"]
    },
    {
      "Sid": "InvalidateOnlyThisDistribution",
      "Effect": "Allow",
      "Action": ["cloudfront:CreateInvalidation","cloudfront:GetInvalidation",
                 "cloudfront:ListInvalidations"],
      "Resource": "arn:aws:cloudfront::111122223333:distribution/E1234ABCDEFGH"
    }
  ]
}
```

Note that the deploy role does **not** get `UpdateDistribution`. Config changes go through IaC with
a separate, reviewed role.

### 11.4 Security checklist

```
□ S3 bucket: Block Public Access ON, OAC configured, SourceArn condition present
□ No S3 website endpoint used as origin (it can't be private)
□ Viewer protocol policy = Redirect HTTP to HTTPS (or HTTPS Only)
□ Origin protocol policy = HTTPS Only for public origins
□ Minimum TLS = TLSv1.2_2021
□ HSTS enabled with a long max-age (and preload only when you're sure)
□ WAF attached with managed rules + rate limiting
□ Direct origin access blocked (VPC origin, or prefix list + secret header)
□ Signed URLs/cookies for any non-public content
□ Private signing keys stored in Secrets Manager, rotated, never in git
□ Standard logging enabled and retained per your compliance policy
□ CloudTrail capturing CloudFront API calls
□ Lambda@Edge / CF Function code reviewed — they see every request
□ Server / X-Powered-By headers removed via response headers policy
□ Distributions tagged and owned
```

---

## 12. Observability, Logging & Monitoring

### 12.1 CloudWatch metrics (namespace `AWS/CloudFront`, region **us-east-1**)

**Default (free) metrics:**

| Metric | Meaning | Alarm on |
|---|---|---|
| `Requests` | Total requests | Sudden drops (DNS/config break) or spikes (attack) |
| `BytesDownloaded` / `BytesUploaded` | Traffic volume | Cost anomalies |
| `TotalErrorRate` | % of 4xx + 5xx | > 5% |
| `4xxErrorRate` | % of 4xx | > 10% (may be normal for APIs) |
| `5xxErrorRate` | % of 5xx | **> 1% — page someone** |

**Additional metrics (opt-in, small monthly charge per distribution):**

| Metric | Meaning |
|---|---|
| `CacheHitRate` | % of requests served from cache — **your single most useful optimization metric** |
| `OriginLatency` | Time from origin request to last byte — origin health |
| `ErrorRate by status code` | 401, 403, 404, 502, 503, 504 broken out individually |

```bash
aws cloudfront create-monitoring-subscription \
  --distribution-id E1234ABCDEFGH \
  --monitoring-subscription RealtimeMetricsSubscriptionConfig={RealtimeMetricsSubscriptionStatus=Enabled}
```

### 12.2 Standard access logs

Standard logging (v2) can deliver to **S3, CloudWatch Logs, or Amazon Data Firehose**, with a
selectable field list and output format (plain, w3c, JSON, Parquet).

**Characteristics:**
- Delivered on a best-effort basis, typically within minutes (not real time).
- ~30+ fields including timestamp, edge location, bytes, IP, method, host, URI, status, referer,
  user agent, query string, cookies, edge result type, request id, TLS details, and timings.
- `x-edge-result-type` is the field you'll live in: `Hit`, `Miss`, `RefreshHit`, `LimitExceeded`,
  `CapacityExceeded`, `Error`, `Redirect`, `OriginShieldHit`.

**Athena table for logs in S3 (v2 plain format):**

```sql
CREATE EXTERNAL TABLE cf_logs (
  `date` DATE, time STRING, x_edge_location STRING, sc_bytes BIGINT,
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
LOCATION 's3://my-cf-logs/cloudfront/'
TBLPROPERTIES ('skip.header.line.count'='2');
```

**The three queries you'll actually run:**

```sql
-- 1. Cache hit ratio over the last day
SELECT x_edge_result_type, COUNT(*) AS n,
       ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) AS pct
FROM cf_logs WHERE "date" >= current_date - interval '1' day
GROUP BY x_edge_result_type ORDER BY n DESC;

-- 2. Which URLs are missing the cache most (your optimization worklist)
SELECT cs_uri_stem, COUNT(*) AS misses
FROM cf_logs WHERE x_edge_result_type = 'Miss' AND "date" >= current_date - interval '1' day
GROUP BY cs_uri_stem ORDER BY misses DESC LIMIT 50;

-- 3. Slowest responses (origin problems)
SELECT cs_uri_stem, AVG(time_taken) AS avg_s, MAX(time_taken) AS max_s, COUNT(*) AS n
FROM cf_logs WHERE "date" >= current_date - interval '1' day
GROUP BY cs_uri_stem HAVING COUNT(*) > 100 ORDER BY avg_s DESC LIMIT 25;
```

### 12.3 Real-time logs

Delivered to **Kinesis Data Streams** within seconds, with a configurable sampling rate (1–100%) and
field selection. Use for live dashboards, real-time attack detection, and immediate error alerting.

**Cost note:** priced per million log lines *plus* Kinesis costs. Sample at 1–10% for high-traffic
distributions; 100% only for short investigations.

### 12.4 CloudFront console reports

Free, built-in, and often forgotten: **Cache statistics**, **Popular objects**, **Top referrers**,
**Usage**, and **Viewers** (devices, browsers, OS, locations). "Popular objects" shows per-object hit
and miss counts — a fast way to find your cache-ratio problem without writing any Athena SQL.

### 12.5 The three headers that debug 80% of problems

```
X-Cache:      Hit from cloudfront | Miss from cloudfront | RefreshHit from cloudfront
              | Error from cloudfront | LimitExceeded from cloudfront
Age:          seconds this object has been in the cache (absent on a miss)
X-Amz-Cf-Pop: BOM51-P2   ← the edge location that served you
X-Amz-Cf-Id:  opaque request id — ALWAYS include this in an AWS support ticket
```

---

## 13. Pricing & Cost Optimization

### 13.1 What you're billed for (pay-as-you-go)

| Dimension | Notes |
|---|---|
| **Data transfer out to internet** | The dominant cost. Priced per GB, **varies by region** — India/South America/Australia cost several times more than North America. |
| **Data transfer out to origin** | Small; only for request bodies (uploads). |
| **HTTP/HTTPS requests** | Per 10,000. HTTPS costs meaningfully more than HTTP. |
| **Invalidations** | First 1,000 paths/month free, then per path. |
| **Field-level encryption** | Per 10,000 requests. |
| **Origin Shield** | Per 10,000 requests reaching the shield. |
| **Real-time logs** | Per million log lines (plus Kinesis). |
| **CloudFront Functions** | Per million invocations (2 M/month permanently free). |
| **Lambda@Edge** | Per million requests + GB-seconds. |
| **Dedicated IP SSL** | Flat monthly per distribution — avoid. |
| **Anycast static IPs** | Enterprise flat monthly per list — avoid unless required. |
| **Distribution tenants** | Free allowance, then tiered. |

**Free by design:**
- **Origin → CloudFront data transfer from AWS origins (S3, EC2, ALB, API Gateway) is $0.** This
  alone often makes CloudFront cheaper than serving directly from an ALB.
- Regional Edge Caches.
- Shield Standard.
- ACM certificates for CloudFront.

**Always-free tier:** 1 TB data transfer out, 10 million HTTP/HTTPS requests, and 2 million
CloudFront Functions invocations per month — permanently, not just for 12 months.

### 13.2 Flat-rate pricing plans

Introduced in late 2025, these bundle CloudFront + AWS WAF + DDoS protection + Route 53 DNS +
CloudWatch Logs ingestion + a TLS certificate into **one fixed monthly price per distribution with
no overage charges**.

| Tier | Positioning |
|---|---|
| **Free** | One distribution, small allowance, WAF + DDoS + DNS + cert included. Hobby projects. |
| **Pro** | Small sites and blogs with generous allowance and built-in security. |
| **Business** | Business apps: uptime SLA, custom caching rules, advanced WAF, VPC origins, S3 storage credit. |
| **Premium** | High-traffic/mission-critical: origin failover, Origin Shield, mTLS, bot management, configurable usage levels scaling to billions of requests and hundreds of TB per month. |

**Decision guide:**
```
Predictable, security-heavy workload with steady traffic?   → flat-rate is usually cheaper & simpler
Highly variable traffic, or heavy edge compute?             → pay-as-you-go
Multi-tenant SaaS with many custom domains?                 → pay-as-you-go / distribution tenants
Large steady spend and want a discount without a fixed cap? → CloudFront Security Savings Bundle
```

> Prices and tier contents change. Always confirm against the official
> [CloudFront pricing page](https://aws.amazon.com/cloudfront/pricing/) before committing budget.

### 13.3 Ten concrete cost-reduction tactics

1. **Raise your cache hit ratio.** Every 1% improvement is 1% fewer origin requests and less origin
   compute. Start with the "top misses" Athena query.
2. **Trim the cache key.** Remove `utm_*`/`fbclid` query strings and unnecessary headers.
3. **Enable compression.** 60–80% smaller text payloads = 60–80% less egress cost on those objects.
4. **Choose the right price class.** If 95% of your users are in India and Europe, Price Class 200
   can cut costs noticeably.
5. **Stop invalidating.** Use content-hashed filenames. Invalidate only `/index.html`.
6. **Set long TTLs with `immutable`** on versioned assets.
7. **Serve WebP/AVIF images and modern video codecs.** The cheapest byte is the one you don't send.
8. **Use Origin Shield only where it pays for itself** — measure origin request reduction against
   the per-request shield cost.
9. **Sample real-time logs** rather than logging 100%.
10. **Evaluate the Savings Bundle or a flat-rate plan** once your monthly spend is predictable.

### 13.4 The comparison people forget

Serving 10 TB/month directly from an ALB vs. through CloudFront: the ALB path pays full EC2/ALB
internet egress rates on all 10 TB. Through CloudFront, origin→CDN transfer is free and viewer-facing
transfer is at CDN rates — and with a 90% hit ratio your ALB only handles 1 TB of origin fetches.
**CloudFront frequently reduces the bill even before you count the performance benefit.**

---

## 14. Quotas & Hard Limits

> Quotas change. Verify with `aws service-quotas list-service-quotas --service-code cloudfront`
> or the [official quotas page](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-limits.html).

### General

| Entity | Default | Adjustable |
|---|---|---|
| Distributions per account | 500 | ✅ |
| Data transfer rate per distribution | 150 Gbps | ✅ |
| Requests per second per distribution | 250,000 | ✅ |
| Files served per distribution | No quota | — |
| Max request/origin-response length (headers + query string, excl. body) | 32,768 bytes | ❌ |
| Max URL length | 8,192 bytes | ❌ |
| Tags per distribution | 50 | ✅ |
| Real-time log configurations per account | 150 | — |
| Web ACL associations per web ACL | 100 | ✅ |

### Per distribution

| Entity | Default | Adjustable |
|---|---|---|
| Alternate domain names (CNAMEs) | 100 | ✅ |
| Cache behaviors | 75 | ✅ |
| Origins | 25 | ✅ |
| Origin groups | 10 | ✅ |
| Custom error responses | 25 | — |
| Custom headers per origin | 10 | ✅ |
| Connection attempts per origin | 1–3 (default 3) | ❌ |
| Connection timeout per origin | 1–10 s (default 10) | ❌ |
| Response (read) timeout per origin | 1–120 s (default 30) | ✅ |
| Keep-alive timeout per origin | 1–120 s (default 5) | ✅ |
| Countries in a geo restriction | 255 | — |

### Policies

| Entity | Default |
|---|---|
| Cache policies per account | 20 |
| Origin request policies per account | 20 |
| Response headers policies per account | 20 |
| Headers / cookies / query strings per policy | 10 each (adjustable) |
| Custom headers per response headers policy | 10 |
| Key groups per account | 10 |
| Public keys per key group | 5 |
| Key groups per distribution | 4 |

### Invalidations

| Entity | Default |
|---|---|
| Paths per invalidation request (non-wildcard) | 3,000 |
| In-progress invalidation requests with wildcards | 15 |
| Free invalidation paths per month | 1,000 |

### Edge compute

| Entity | CloudFront Functions | Lambda@Edge |
|---|---|---|
| Max code/package size | 10 KB | 1 MB (viewer) / 50 MB (origin) |
| Max execution time | < 1 ms | 5 s (viewer) / 30 s (origin) |
| Memory | ~2 MB | 128 MB – 10 GB |
| Distributions per function | 100 | — |
| Functions per distribution per event type | 1 | 1 |
| KeyValueStore size | 5 MB per store | — |

### Continuous deployment

| Entity | Default |
|---|---|
| Staging distributions per account | 20 |
| Continuous deployment policies per account | 20 |
| Max traffic weight to staging | 15% |
| Session stickiness idle duration | 300–3600 s |

---

## 15. Best Practices & Anti-Patterns

### ✅ Do

| Practice | Why |
|---|---|
| Control TTL with origin `Cache-Control` headers | Deploys change caching without touching CloudFront |
| Use content-hashed filenames + `immutable` | Free, instant, atomic cache busting |
| Use OAC (not OAI, not public buckets) | Modern, supports KMS and all methods |
| Use VPC origins for ALB/NLB/EC2 | Eliminates the bypass path entirely |
| Use managed cache/origin-request policies first | AWS has already thought about the edge cases |
| Order cache behaviors most-specific first | First match wins; a `*` in position 0 shadows everything |
| Enable compression + HTTP/3 + IPv6 | Free performance |
| Attach a response headers policy for security headers | Free, no code |
| Deploy WAF rules in COUNT mode first | You will block real users otherwise |
| Enable additional CloudWatch metrics on production | `CacheHitRate` and `OriginLatency` are worth the few dollars |
| Manage distributions in IaC (Terraform/CDK/CloudFormation) | The console config drifts and is hard to review |
| Tag everything | Cost allocation, ownership, cleanup |
| Test config changes with a staging distribution | 15% of real traffic beats 100% of synthetic traffic |
| Keep `X-Amz-Cf-Id` in your error logs | It's what AWS Support asks for first |

### ❌ Don't

| Anti-pattern | What goes wrong |
|---|---|
| Forwarding **all** headers (especially `User-Agent`) to the cache key | Hit ratio collapses to near zero; you've built an expensive proxy |
| Forwarding the `Host` header to an S3 REST origin | SigV4/OAC breaks → 403 on everything |
| Using an S3 **website** endpoint when you need a private bucket | Website endpoints are public-only and HTTP-only |
| Invalidating `/*` on every deploy | Slow, costs money, and cold-starts your whole cache |
| Requesting the ACM certificate outside us-east-1 | It simply won't appear in the dropdown |
| Setting `MinTTL` high "to be safe" | It overrides `no-cache` and serves stale content |
| Associating `$LATEST` or an alias for Lambda@Edge | Only numbered versions are allowed |
| Deleting a Lambda@Edge function before replicas are removed | Delete fails for hours with a confusing error |
| Leaving WAF in BLOCK with untested custom rules | Outage of your own making |
| Assuming geo restriction is a security control | A VPN defeats it in five seconds |
| Caching responses containing `Set-Cookie` without thinking | You will serve one user's session to another |
| Hardcoding CloudFront IP ranges in firewalls | They change; use the managed prefix list |
| Enabling Anycast static IPs "to see what it does" | Four-figure monthly bill |
| Testing cache behaviour in a browser with devtools open and cache disabled | You're testing the browser, not CloudFront. Use `curl`. |

---

## 16. CloudFront vs. Everything Else

| Service | Layer | Caches? | Static IPs | Best for |
|---|---|---|---|---|
| **CloudFront** | L7 (HTTP/HTTPS) | ✅ | Optional (Anycast IP lists) | Web content, APIs, streaming, WAF perimeter |
| **AWS Global Accelerator** | L4 (TCP/UDP) | ❌ | ✅ Two static anycast IPs | Gaming, VoIP, IoT/MQTT, non-HTTP protocols, fast regional failover |
| **S3 Transfer Acceleration** | Uploads to S3 | ❌ | ❌ | Fast long-distance uploads to a single bucket |
| **ALB / NLB** | L7 / L4, regional | ❌ | NLB: ✅ | In-region load balancing behind CloudFront |
| **API Gateway edge-optimized** | L7 | Limited | ❌ | It literally uses CloudFront under the hood |
| **Route 53** | DNS | ❌ | — | Resolution, health checks, latency/geo routing — complements CloudFront |
| **Cloudflare / Akamai / Fastly** | L7 CDN | ✅ | Varies | Comparable CDNs; CloudFront wins on AWS-native integration and free origin egress |

**Can you use CloudFront *and* Global Accelerator?** Yes — Global Accelerator can front a
CloudFront distribution when you need static IPs for HTTP traffic, though Anycast static IP lists now
address that directly.

---

## 17. Interview-Grade Q&A

**Q: A user reports seeing another user's data on a cached page. What happened?**
The cache key didn't include whatever distinguishes the two users — most often a query string, a
cookie, or the `Authorization` header — so both requests hashed to the same key. Immediate fix:
switch that behavior to `CachingDisabled` and invalidate. Proper fix: either don't cache
authenticated responses (send `Cache-Control: private, no-store` from the origin) or include the
distinguishing input in the cache key.

**Q: Why is my cache hit ratio 3%?**
In order of likelihood: (1) the cache policy forwards `User-Agent` or all headers/cookies;
(2) the origin sends `Cache-Control: no-cache`/`no-store`/`private`; (3) unbounded query strings
(tracking params) are in the cache key; (4) TTLs are effectively zero; (5) the content genuinely is
per-user. Diagnose with the "Popular objects" report or the Athena top-misses query.

**Q: Difference between cache policy and origin request policy?**
The cache policy defines the **cache key** (what makes a request unique). The origin request policy
defines what is **forwarded to the origin**, which is a superset. This lets the origin see the
viewer's country and user agent for logging without those values fragmenting the cache.

**Q: OAI vs OAC?**
OAI is the legacy CloudFront-specific identity referenced in an S3 bucket policy; it can't do
SSE-KMS, can't handle non-GET methods, and isn't supported in newer regions. OAC uses SigV4 signing
with the CloudFront service principal and a `AWS:SourceArn` condition, supports KMS, all HTTP
methods, and MediaStore/Lambda Function URL origins. Use OAC.

**Q: Where do you request the certificate for a CloudFront custom domain?**
ACM in **us-east-1**, always, regardless of where anything else lives.

**Q: When do you choose Lambda@Edge over CloudFront Functions?**
When you need an origin request/response trigger, network access, the request or response body,
more than 10 KB of code, or more than ~1 ms of execution time. Otherwise CloudFront Functions is
faster and roughly 6× cheaper.

**Q: How do you stop people bypassing CloudFront and hitting the ALB directly?**
Best: **VPC origins** — put the ALB in private subnets with no public route. Otherwise: a secret
custom header injected by CloudFront and validated by an ALB listener rule, combined with a security
group referencing the `com.amazonaws.global.cloudfront.origin-facing` managed prefix list.

**Q: Invalidation vs versioned objects?**
Invalidation is a manual, eventually-consistent purge that costs money past 1,000 paths/month.
Versioned (content-hashed) filenames change the cache key so the new object is a cache miss and the
old one simply ages out. Versioning is atomic, free, and instant. Use invalidation only for
entry-point documents like `index.html`.

**Q: Explain CloudFront's caching layers.**
Edge locations (750+, small hot cache), Regional Edge Caches (larger second tier, free and
automatic, where Lambda@Edge runs), and optionally Origin Shield (one designated region, collapses
requests globally). Each layer reduces load on the next.

**Q: What does a 502 from CloudFront usually mean?**
An SSL/TLS problem between CloudFront and the origin: certificate expired, hostname mismatch,
untrusted CA, self-signed cert, or no shared cipher/protocol. Also possible: the origin returned a
malformed response or closed the connection.

**Q: What does a 504 mean?**
The origin didn't respond within the response timeout (default 30 s), or a security group / NACL /
firewall silently dropped the connection. Check reachability first, then raise the timeout if the
origin is legitimately slow.

**Q: How do you serve a React SPA with client-side routing?**
Either custom error responses mapping 403 and 404 to `/index.html` with a 200 status, or — cleaner —
a CloudFront Function on viewer request that rewrites any extension-less URI to `/index.html`. Keep
`/api/*` on a separate behavior so real API errors aren't masked.

**Q: How would you do a canary deployment of a CloudFront config change?**
Continuous deployment: create a staging distribution, apply the change there, attach a continuous
deployment policy sending up to 15% of traffic (or header-targeted traffic) to staging, watch
metrics, then promote.

---

## 18. Glossary

| Term | Definition |
|---|---|
| **Anycast** | One IP address announced from many locations; the network routes you to the nearest. How CloudFront gets you to a close PoP. |
| **Cache key** | The fingerprint CloudFront uses to identify a cached object. |
| **Canned policy** | A simple signed-URL policy with expiry only. |
| **CNAME (alternate domain name)** | Your own hostname attached to a distribution. |
| **Cold cache** | A cache with no stored objects; every request is a miss. |
| **Collapsing** | Merging simultaneous identical cache misses into a single origin request. |
| **Custom policy** | A signed-URL/cookie policy supporting start time, IP restriction, and wildcards. |
| **Distribution tenant** | A per-customer instance of a distribution template (SaaS Manager). |
| **Edge location / PoP** | A CloudFront data centre serving viewers. |
| **ETag** | Optimistic-concurrency token for distribution updates (also an HTTP validator). |
| **Field-level encryption** | Encrypting specific POST form fields at the edge with your public key. |
| **Invalidation** | Manual removal of an object from all caches. |
| **JA3 fingerprint** | A TLS client fingerprint CloudFront can expose for bot detection. |
| **Key group** | A set of public keys trusted for signed URLs/cookies. |
| **Lambda@Edge** | Full Lambda functions running at Regional Edge Caches on all four triggers. |
| **Managed prefix list** | AWS-maintained list of CloudFront origin-facing IP ranges for security groups. |
| **mTLS** | Mutual TLS — the client also presents a certificate. |
| **OAC / OAI** | Origin Access Control (current) / Origin Access Identity (legacy). |
| **Origin group** | Primary + secondary origin with failover status codes. |
| **Origin Shield** | Optional single caching layer in front of the origin. |
| **Path pattern** | The glob a cache behavior matches, e.g. `/api/*`. |
| **Price class** | Which set of edge locations to use (All / 200 / 100). |
| **QUIC / HTTP/3** | UDP-based transport with faster handshakes and better loss recovery. |
| **Regional Edge Cache** | Second-tier cache between PoPs and the origin. |
| **Request collapsing** | See *Collapsing*. |
| **Response headers policy** | Reusable config for headers CloudFront adds/removes on responses. |
| **SigV4** | AWS Signature Version 4 — used by OAC to sign origin requests. |
| **Signed cookie** | Time-limited credential granting access to many objects. |
| **Signed URL** | Time-limited credential granting access to one object. |
| **SNI** | Server Name Indication — lets many certs share an IP. Free on CloudFront. |
| **Stale-while-revalidate** | Serve slightly-stale content while refreshing in the background. |
| **TTL** | Time To Live — cache freshness duration. |
| **VPC origin** | An origin in a private subnet reachable only by CloudFront. |
| **X-Cache** | Response header telling you Hit / Miss / RefreshHit / Error. |
| **X-Amz-Cf-Id** | Opaque per-request identifier for AWS Support. |
| **X-Amz-Cf-Pop** | Which edge location served the response. |

---

## Where to Next

1. **[hands-on-labs.md](./hands-on-labs.md)** — build all of this, lab by lab, starting from an
   empty AWS account.
2. **[commands-cheatsheet.md](./commands-cheatsheet.md)** — the CLI reference you'll keep open.
3. **[troubleshooting.md](./troubleshooting.md)** — bookmark this one before you need it.

---

## Contributing

Corrections and additions are welcome. AWS ships CloudFront features constantly — if something here
has gone stale, open an issue or a PR with a link to the current AWS documentation.

## Disclaimer

Quotas, pricing, feature availability, and managed-policy IDs change over time. This repository is a
learning resource; always verify against the
[official AWS CloudFront documentation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)
before making production or budget decisions.
