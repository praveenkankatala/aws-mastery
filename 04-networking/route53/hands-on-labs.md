# 🧪 Hands-On Labs — DNS & Route 53

Build every concept from `README.md` with your own hands. Each lab has an **objective**,
**prerequisites**, **steps**, **verification**, and **cleanup** so you don't leave billable
resources running.

> 💡 All CLI commands referenced here are explained in detail in
> [`commands-cheatsheet.md`](./commands-cheatsheet.md). If something breaks, check
> [`troubleshooting.md`](./troubleshooting.md).

---

## Lab Index

| # | Lab | Concept Practiced |
|---|---|---|
| 1 | [Register a Domain](#lab-1--register-a-domain) | Domain registration flow |
| 2 | [Create a Hosted Zone](#lab-2--create-a-hosted-zone) | Hosted zones, NS/SOA records |
| 3 | [Point EC2 to a Subdomain](#lab-3--point-an-ec2-instance-to-a-subdomain) | A records, Elastic IP |
| 4 | [Point S3 Static Site to a Subdomain](#lab-4--point-an-s3-static-website-to-a-subdomain) | Alias records |
| 5 | [Weighted Routing (Blue/Green)](#lab-5--weighted-routing-bluegreen-deployment) | Weighted policy |
| 6 | [Failover Routing + Health Checks](#lab-6--failover-routing-with-health-checks) | Failover, health checks |
| 7 | [Latency-Based Routing](#lab-7--latency-based-routing) | Latency policy, multi-region |
| 8 | [Geolocation Routing](#lab-8--geolocation-routing) | Geolocation policy |
| 9 | [Multi-Value Answer Routing](#lab-9--multi-value-answer-routing) | Multi-value policy |
| 10 | [Private Hosted Zone + Split-Horizon](#lab-10--private-hosted-zone--split-horizon-dns) | Private zones, split-horizon |
| 11 | [Hybrid DNS with Route 53 Resolver](#lab-11--hybrid-dns-with-route-53-resolver) | Inbound/outbound endpoints |
| 12 | [Enable DNSSEC](#lab-12--enable-dnssec) | DNSSEC signing |
| 13 | [DNS Firewall Rule Group](#lab-13--dns-firewall-rule-group) | Firewall allow/block lists |
| 14 | [Verify with DNS Checker & dig](#lab-14--verify-with-dns-checker--dig) | Global propagation testing |

---

## Lab 1 — Register a Domain

**Objective:** Purchase a domain and understand the registrant/registrar/registry flow.

**Steps:**
1. Open the **Route 53 Console → Registered Domains → Register Domain**.
2. Search for a name, e.g. `yourname-devops.com`, and add it to the cart.
3. Fill in Registrant, Admin, Technical, and Billing contact details (or accept WHOIS Privacy).
4. Complete checkout.

**Verification:**
```bash
aws route53domains get-domain-detail --domain-name yourname-devops.com
```
Confirm the domain status is `active`/`ok` and a Registry Domain ID has been assigned.

**Cleanup:** None needed — domain registrations are annual; just don't auto-renew if you don't
want to keep it.

---

## Lab 2 — Create a Hosted Zone

**Objective:** Understand how a hosted zone is the administrative container for a domain.

> If you registered the domain directly in Route 53 (Lab 1), a public hosted zone was created
> automatically — skip to verification. This lab also shows the manual path for a domain
> registered elsewhere.

**Steps:**
1. **Hosted Zones → Create Hosted Zone.**
2. Enter your domain name exactly, select **Public Hosted Zone**, click Create.
3. Note the auto-created **NS** record (4 name servers) and **SOA** record.
4. If the domain is registered with an external registrar, copy these 4 NS values into that
   registrar's "custom nameservers" setting.

**Verification:**
```bash
aws route53 get-hosted-zone --id /hostedzone/YOUR_ZONE_ID
dig yourname.com NS +short
```
The `dig` output should match the 4 NS values shown in the console.

**Cleanup:**
```bash
aws route53 delete-hosted-zone --id /hostedzone/YOUR_ZONE_ID
```
(Only after removing any custom records you added beyond the default NS/SOA.)

---

## Lab 3 — Point an EC2 Instance to a Subdomain

**Objective:** Route `app.yourname.com` to a live EC2 web server using an A record.

**Prerequisites:**
- A running EC2 instance with a web server (e.g. nginx) on port 80.
- An **Elastic IP** allocated and associated with the instance (so the IP survives restarts).

**Steps:**
1. Allocate and associate an Elastic IP: EC2 Console → Elastic IPs → Allocate → Associate.
2. In your Hosted Zone, click **Create Record**.
3. **Record name:** `app`
4. **Record type:** `A`
5. **Alias:** OFF
6. **Value:** paste the Elastic IP
7. **TTL:** `300`
8. Click **Create Records**.

**Verification:**
```bash
dig app.yourname.com A +short
curl -I http://app.yourname.com
```
You should see the Elastic IP returned and an HTTP response from your web server.

**Cleanup:** Delete the A record, release the Elastic IP, and terminate the instance if no
longer needed (Elastic IPs incur charges when not attached to a running instance).

---

## Lab 4 — Point an S3 Static Website to a Subdomain

**Objective:** Serve a static site from S3 at `web.yourname.com` using an Alias record.

**Prerequisites:**
- An S3 bucket **named exactly** `web.yourname.com` (bucket name must match the subdomain).
- Static website hosting enabled on the bucket, with public read access and an `index.html`.

**Steps:**
1. Create the bucket: `aws s3 mb s3://web.yourname.com`
2. Enable static hosting and upload `index.html` (see cheatsheet for bucket policy JSON).
3. In your Hosted Zone, click **Create Record**.
4. **Record name:** `web`
5. **Record type:** `A`
6. **Alias:** ON → **Route traffic to → Alias to S3 website endpoint**
7. Select your region, then choose the `web.yourname.com` bucket from the dropdown.
8. Click **Create Records**.

**Verification:**
```bash
dig web.yourname.com A +short
curl -I http://web.yourname.com
```
The resolved IPs will belong to AWS's regional S3 website infrastructure, not a single fixed
IP — this is expected because it's an Alias, not a plain A record.

**Cleanup:** Delete the record, empty and delete the bucket.

---

## Lab 5 — Weighted Routing (Blue/Green Deployment)

**Objective:** Send 90% of traffic to a stable version and 10% to a canary version.

**Prerequisites:** Two EC2 instances (or two ASGs) serving slightly different content, e.g.
"v1" and "v2" pages.

**Steps:**
1. Create two `A` records both named `canary.yourname.com`.
2. Record 1: **Set ID** = `stable`, **Weight** = `90`, value = instance-1 IP.
3. Record 2: **Set ID** = `canary`, **Weight** = `10`, value = instance-2 IP.
4. Set TTL low (e.g. `60`) so you can observe shifts quickly.

**Verification:** Run repeated queries and count the distribution:
```bash
for i in {1..20}; do dig canary.yourname.com +short; done | sort | uniq -c
```
Roughly 18 responses should point to the stable IP and 2 to the canary IP.

**Cleanup:** Delete both weighted records.

---

## Lab 6 — Failover Routing with Health Checks

**Objective:** Automatically fail over from a primary site to a static S3 disaster-recovery
site when the primary becomes unhealthy.

**Prerequisites:** A primary EC2 web server, and a secondary S3 static site (from Lab 4-style
setup) as the DR backup.

**Steps:**
1. Create a **Health Check** on the primary EC2's public IP/path (`/health`, HTTP, 30s
   interval, failure threshold 3).
2. Create record `site.yourname.com`, **Type A**, **Routing Policy: Failover → Primary**,
   attach the health check, value = primary EC2 IP.
3. Create a second record, same name, **Routing Policy: Failover → Secondary**, Alias to the
   S3 DR bucket (no health check needed on the secondary).

**Verification:**
1. Confirm `dig site.yourname.com` returns the primary IP while healthy.
2. Stop the web server on the primary instance (or block port 80 in its security group).
3. Wait ~90 seconds (3 failed checks × 30s interval) and query again — it should now resolve
   to the S3 DR site.

**Cleanup:** Delete both failover records and the health check.

---

## Lab 7 — Latency-Based Routing

**Objective:** Route users to whichever AWS region gives them the fastest response.

**Prerequisites:** Two EC2 instances or ALBs in two different regions (e.g. `us-east-1` and
`ap-south-1`).

**Steps:**
1. Create record `global.yourname.com`, **Type A**, **Routing Policy: Latency**, **Region:
   us-east-1**, value = us-east-1 resource IP/Alias.
2. Create a second record, same name, **Region: ap-south-1**, value = ap-south-1 resource.

**Verification:** Query from two different network locations (e.g. a VPN or an online `dig`
tool in each region) and confirm each gets routed to its geographically/network-closer
region.

**Cleanup:** Delete both latency records.

---

## Lab 8 — Geolocation Routing

**Objective:** Serve different content by continent/country, with a default fallback.

**Steps:**
1. Create record `geo.yourname.com`, **Type A**, **Routing Policy: Geolocation**,
   **Location: North America**, value = US server IP.
2. Add another record, same name, **Location: Europe**, value = EU server IP.
3. Add a third record, same name, **Location: Default**, value = a global fallback IP (handles
   any location not explicitly matched).

**Verification:** Use a VPN set to different countries, or an online geo-DNS checker, and
confirm each region resolves to its matching server, with unmatched regions falling back to
default.

**Cleanup:** Delete all three records.

---

## Lab 9 — Multi-Value Answer Routing

**Objective:** Distribute traffic across multiple redundant servers with automatic health
filtering, without paying for a full Application Load Balancer.

**Steps:**
1. Create up to 8 `A` records all named `pool.yourname.com`, **Routing Policy: Multi-Value
   Answer**, each with a unique **Set ID**, each pointing at a different server IP.
2. Attach a health check to each record.

**Verification:**
```bash
dig pool.yourname.com +short
```
Repeated queries return a shuffled subset of the healthy IPs. Stop one server and confirm its
IP disappears from the response within a few health-check intervals.

**Cleanup:** Delete all pool records and their health checks.

---

## Lab 10 — Private Hosted Zone + Split-Horizon DNS

**Objective:** Make `api.yourname.com` resolve to an internal IP inside your VPC, and a
public IP everywhere else.

**Steps:**
1. Ensure your VPC has `enableDnsSupport` and `enableDnsHostnames` set to `true`.
2. Create a **Private Hosted Zone** named `yourname.com`, associated with your VPC.
3. Inside it, create an `A` record `api.yourname.com` → internal ALB/EC2 private IP.
4. In your existing **Public Hosted Zone** for the same domain, create `api.yourname.com` →
   public ALB IP/Alias.

**Verification:**
- From an EC2 instance inside the VPC: `dig api.yourname.com +short` → internal IP.
- From your laptop on the public internet: `dig api.yourname.com +short` → public IP.

**Cleanup:** Delete both records, disassociate and delete the private hosted zone.

---

## Lab 11 — Hybrid DNS with Route 53 Resolver

**Objective:** Simulate an on-prem-to-AWS DNS bridge using inbound/outbound resolver
endpoints. (Can be simulated fully within two VPCs if you don't have real on-prem
infrastructure.)

**Steps:**
1. Create a **Resolver Inbound Endpoint** in "VPC-AWS", attached to two subnets/AZs.
2. Create a **Resolver Outbound Endpoint** in the same VPC.
3. Create a **Conditional Forwarding Rule** for `corp.internal` pointing at your simulated
   on-prem DNS server IPs (e.g. an EC2 instance running BIND in "VPC-OnPrem").
4. Associate the rule with "VPC-AWS".

**Verification:** From an EC2 instance in "VPC-AWS", resolve a `corp.internal` name and
confirm it forwards correctly to the simulated on-prem DNS server.

**Cleanup:** Delete the resolver rule association, the rule, and both endpoints (they incur
hourly charges).

---

## Lab 12 — Enable DNSSEC

**Objective:** Sign your hosted zone to protect against cache poisoning.

**Steps:**
1. Create a KMS customer-managed key configured for DNSSEC signing.
2. Create a **Key Signing Key (KSK)** referencing that KMS key.
3. Run `enable-hosted-zone-dnssec` on your hosted zone.
4. If your domain is registered elsewhere, submit the **DS record** shown in the console to
   your registrar to complete the chain of trust.

**Verification:**
```bash
aws route53 get-dnssec --hosted-zone-id YOUR_ZONE_ID
dig yourname.com DNSKEY +short
```

**Cleanup:** Disable DNSSEC before deleting the hosted zone, then delete the KSK and KMS key.

---

## Lab 13 — DNS Firewall Rule Group

**Objective:** Block outbound DNS resolution to a list of disallowed domains from your VPC.

**Steps:**
1. Create a **Firewall Domain List** and add 1–2 test domains (use harmless placeholder
   domains, not real malware domains).
2. Create a **Firewall Rule Group** with a rule: action `BLOCK`, response `NXDOMAIN`,
   priority `100`.
3. Associate the rule group with your VPC.

**Verification:** From an EC2 instance in that VPC:
```bash
dig blocked-test-domain.com
```
Expect an `NXDOMAIN` response instead of a real answer.

**Cleanup:** Disassociate the rule group from the VPC, then delete the rule, rule group, and
domain list.

---

## Lab 14 — Verify with DNS Checker & dig

**Objective:** Confirm global propagation of any record you've created.

**Steps:**
1. Go to [dnschecker.org](https://dnschecker.org).
2. Enter `app.yourname.com`, select record type `A`, click Search.
3. Confirm green checkmarks (matching IP) across most/all global locations.
4. Cross-check locally:
```bash
dig app.yourname.com A @8.8.8.8 +short
dig app.yourname.com A @1.1.1.1 +short
```

**Verification:** All sources should agree once propagation is complete (allow up to
24–48 hours for a freshly changed NS delegation; much faster, often seconds to minutes, for a
record change within an already-propagated zone).

**Cleanup:** N/A — this is a read-only verification lab.
