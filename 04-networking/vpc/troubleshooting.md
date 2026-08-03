# AWS VPC — Troubleshooting Guide

> Symptom → root cause → fix. Plus the verbatim error messages AWS gives you and what they actually mean.
> Written for the moment when something is broken and you need an answer, not a lecture.

---

## Table of Contents

1. [The Universal Diagnostic Method](#1-the-universal-diagnostic-method)
2. [Quick Symptom Index](#2-quick-symptom-index)
3. [Connectivity: Internet Access](#3-connectivity-internet-access)
4. [Connectivity: Inside the VPC](#4-connectivity-inside-the-vpc)
5. [Security Groups & NACLs](#5-security-groups--nacls)
6. [NAT Gateway Problems](#6-nat-gateway-problems)
7. [DNS Problems](#7-dns-problems)
8. [VPC Endpoint Problems](#8-vpc-endpoint-problems)
9. [Peering Problems](#9-peering-problems)
10. [Transit Gateway Problems](#10-transit-gateway-problems)
11. [VPN & Direct Connect Problems](#11-vpn--direct-connect-problems)
12. [IP Address Exhaustion & CIDR Problems](#12-ip-address-exhaustion--cidr-problems)
13. [Deletion & Dependency Errors](#13-deletion--dependency-errors)
14. [Performance & MTU Problems](#14-performance--mtu-problems)
15. [Session Manager / SSH Access Problems](#15-session-manager--ssh-access-problems)
16. [Load Balancer & Target Health Problems](#16-load-balancer--target-health-problems)
17. [IPv6 Problems](#17-ipv6-problems)
18. [Cost Surprises](#18-cost-surprises)
19. [Verbatim Error Message Reference](#19-verbatim-error-message-reference)
20. [Diagnostic Command Toolkit](#20-diagnostic-command-toolkit)

---

## 1. The Universal Diagnostic Method

Before touching anything, work through these five layers **in order**. Ninety percent of VPC problems are found by layer 3.

```
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 1 — ROUTING                                               │
│   Is there a route from source to destination, and back?        │
│   → aws ec2 describe-route-tables                               │
│   → Symptom if broken: total silence, timeouts, no log entries   │
├─────────────────────────────────────────────────────────────────┤
│ LAYER 2 — NACL (subnet boundary, BOTH directions, stateless)    │
│   Inbound rules on the destination subnet?                      │
│   Outbound rules on the source subnet?                          │
│   Ephemeral ports allowed for the return path?                  │
│   → Symptom if broken: timeouts, REJECT in Flow Logs            │
├─────────────────────────────────────────────────────────────────┤
│ LAYER 3 — SECURITY GROUP (ENI, stateful)                        │
│   Destination SG inbound allows source SG/CIDR on that port?    │
│   Source SG outbound allows it?                                 │
│   → Symptom if broken: timeouts, REJECT in Flow Logs            │
├─────────────────────────────────────────────────────────────────┤
│ LAYER 4 — HOST                                                  │
│   Is the app listening? On the right interface (0.0.0.0 not     │
│   127.0.0.1)? Is iptables/firewalld blocking it?                │
│   → ss -tulnp ; sudo iptables -L -n                             │
│   → Symptom if broken: "connection refused" (fast, not a hang)  │
├─────────────────────────────────────────────────────────────────┤
│ LAYER 5 — APPLICATION / IAM                                     │
│   Wrong port, wrong protocol, TLS mismatch, IAM denial          │
│   → Symptom: connects then errors, HTTP 4xx/5xx, AccessDenied   │
└─────────────────────────────────────────────────────────────────┘
```

### The single most useful distinction

| Behaviour | Meaning |
|---|---|
| **Connection hangs / times out** | Packets are being **dropped silently** → routing, security group, or NACL |
| **Connection refused (instant)** | Packets **arrived**; nothing is listening on that port → application or host firewall |
| **Connection resets mid-stream** | MTU, idle timeout, asymmetric routing, or the app crashed |
| **Intermittent failure** | NACL ephemeral ports, DNS rate limit, NAT port exhaustion, or one AZ misconfigured |
| **Works from one instance, not another** | Different subnet → different route table or NACL. Compare them. |

### Start with the tools, not with guesses

```bash
# 1. Ask AWS to analyse the configuration (no packets sent — safe in prod)
aws ec2 create-network-insights-path --source i-0src --destination i-0dst \
  --destination-port 443 --protocol tcp
aws ec2 start-network-insights-analysis --network-insights-path-id nip-0abc

# 2. Are packets even arriving? Check Flow Logs
fields @timestamp, srcaddr, dstaddr, dstport, action, tcp_flags
| filter dstaddr = "10.0.10.42"
| sort @timestamp desc | limit 50

# 3. From the instance
ss -tulnp | grep :8080
sudo tcpdump -i any -nn port 8080
```

---

## 2. Quick Symptom Index

| Symptom | Jump to |
|---|---|
| Instance can't reach the internet | [§3.1](#31-instance-cannot-reach-the-internet) |
| Can't SSH / connect to a public instance | [§3.3](#33-cannot-connect-to-a-public-instance-from-outside) |
| Private instance has no internet despite NAT | [§6.1](#61-private-instances-have-no-internet-despite-a-nat-gateway) |
| Two instances in the same VPC can't talk | [§4.1](#41-two-instances-in-the-same-vpc-cannot-communicate) |
| Works sometimes, fails sometimes | [§5.3](#53-intermittent-failures--the-ephemeral-port-trap) |
| DNS resolution failing / SERVFAIL | [§7](#7-dns-problems) |
| Endpoint created but nothing works | [§8.1](#81-interface-endpoint-created-but-connections-time-out) |
| Peering shows active but no traffic | [§9.1](#91-peering-is-active-but-traffic-does-not-flow) |
| TGW attached but VPCs can't reach each other | [§10.1](#101-tgw-attachment-exists-but-no-connectivity) |
| VPN tunnel DOWN | [§11.1](#111-vpn-tunnel-shows-down) |
| `InsufficientFreeAddressesInSubnet` | [§12.1](#121-insufficientfreeaddressesinsubnet) |
| `DependencyViolation` on delete | [§13.1](#131-dependencyviolation-when-deleting-a-subnet-or-vpc) |
| SSH connects then hangs on large output | [§14.1](#141-ssh-connects-then-hangs--mtu--path-mtu-discovery) |
| Session Manager: `TargetNotConnected` | [§15.1](#151-targetnotconnected) |
| ALB targets unhealthy | [§16.1](#161-target-group-shows-unhealthy) |
| Unexpected AWS bill | [§18](#18-cost-surprises) |
| A specific error string | [§19](#19-verbatim-error-message-reference) |

---

## 3. Connectivity: Internet Access

### 3.1 Instance cannot reach the internet

**The four-point checklist.** All four must be true; any one missing causes silent failure.

```bash
INSTANCE=i-0abc
SUBNET=$(aws ec2 describe-instances --instance-ids $INSTANCE \
  --query 'Reservations[0].Instances[0].SubnetId' --output text)
VPC=$(aws ec2 describe-instances --instance-ids $INSTANCE \
  --query 'Reservations[0].Instances[0].VpcId' --output text)

# ── CHECK 1: Is an IGW attached to the VPC? ──
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC" \
  --query 'InternetGateways[].[InternetGatewayId,Attachments[0].State]' --output table
# Expect: one row, state "available". Empty = no IGW.

# ── CHECK 2: Does the subnet's route table have 0.0.0.0/0 → igw? ──
RT=$(aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$SUBNET" \
     --query 'RouteTables[0].RouteTableId' --output text)
# If that returns None, the subnet uses the MAIN route table:
[ "$RT" = "None" ] && RT=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC" "Name=association.main,Values=true" \
  --query 'RouteTables[0].RouteTableId' --output text)
aws ec2 describe-route-tables --route-table-ids $RT \
  --query 'RouteTables[].Routes[].[DestinationCidrBlock,GatewayId,NatGatewayId,State]' --output table

# ── CHECK 3: Does the instance have a public IP or EIP? ──
aws ec2 describe-instances --instance-ids $INSTANCE \
  --query 'Reservations[0].Instances[0].[PublicIpAddress,PrivateIpAddress]'

# ── CHECK 4: Do SG and NACL allow the traffic? ──
aws ec2 describe-security-groups --group-ids $(aws ec2 describe-instances \
  --instance-ids $INSTANCE --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' --output text)
aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=$SUBNET" \
  --query 'NetworkAcls[].Entries'
```

| Which check failed | Fix |
|---|---|
| 1 — no IGW | `aws ec2 create-internet-gateway` then `attach-internet-gateway` |
| 2 — no default route | `aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxx` |
| 2 — subnet on the main route table unintentionally | `aws ec2 associate-route-table --route-table-id rtb-public --subnet-id $SUBNET` |
| 3 — no public IP | Assign an EIP, or set `--map-public-ip-on-launch` and relaunch (you can't add an auto-assigned public IP to a running instance) |
| 4 — SG blocks egress | `aws ec2 authorize-security-group-egress --group-id sg-xxx --protocol -1 --port -1 --cidr 0.0.0.0/0` |
| 4 — NACL blocks return traffic | Add outbound allow for `1024-65535` |

> **Common trap:** the instance *does* have a public IP but is in a **private** subnet (routed to a NAT Gateway). A public IP with no IGW route is useless — and you're paying for it.

### 3.2 Instance loses its public IP after stop/start

**Cause:** auto-assigned public IPv4 addresses are released on stop and a new one is allocated on start. They are not persistent by design.

**Fix:** allocate an **Elastic IP** and associate it.

```bash
ALLOC=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
aws ec2 associate-address --allocation-id $ALLOC --instance-id i-0abc
```

Or, better, put the instance behind a load balancer or use DNS instead of depending on an IP.

### 3.3 Cannot connect to a public instance from outside

Work through this in order:

```bash
# Is the instance actually running and initialised?
aws ec2 describe-instance-status --instance-ids i-0abc \
  --query 'InstanceStatuses[].[InstanceState.Name,SystemStatus.Status,InstanceStatus.Status]'

# Is the SG rule really there, for your IP?
MYIP=$(curl -s https://checkip.amazonaws.com); echo "My IP: $MYIP"
aws ec2 describe-security-groups --group-ids sg-0abc --query 'SecurityGroups[].IpPermissions'

# Does the NACL allow inbound AND the ephemeral return?
aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=subnet-0abc"

# Is the app listening on all interfaces, not just loopback?
# (from Session Manager)
ss -tulnp | grep :80
#   0.0.0.0:80   ✅ good
#   127.0.0.1:80 ❌ only reachable from the instance itself
```

**Other causes people miss:**
- Corporate firewall or ISP blocking outbound 22 from your side.
- Your public IP changed (home broadband, VPN toggled) and the SG rule is stale.
- The instance is behind a NAT Gateway route, not an IGW route.
- Host firewall (`firewalld`, `ufw`, `iptables`, Windows Firewall) blocking it.

### 3.4 `curl` works but the browser doesn't (or vice versa)

- Browser is forcing **HTTPS** and only port 80 is open. Check the URL scheme.
- HSTS cached from a previous HTTPS site on the same host.
- Browser proxy or corporate MITM inspection.
- Test explicitly: `curl -v http://IP` vs `curl -vk https://IP`.

---

## 4. Connectivity: Inside the VPC

### 4.1 Two instances in the same VPC cannot communicate

Remember: the `local` route always exists, so **routing is never the problem within a single VPC** (unless you've added a more specific route pointing elsewhere). It's almost always security groups.

**Four checkpoints in each direction:**

```
Source SG (outbound) → Source subnet NACL (outbound) →
Destination subnet NACL (inbound) → Destination SG (inbound)
```

```bash
# The fastest possible answer:
aws ec2 create-network-insights-path --source i-0src --destination i-0dst \
  --destination-port 3306 --protocol tcp
aws ec2 start-network-insights-analysis --network-insights-path-id nip-0abc
aws ec2 describe-network-insights-analyses --network-insights-analysis-ids nia-0abc \
  --query 'NetworkInsightsAnalyses[0].{Reachable:NetworkPathFound,Why:Explanations}'
```

**Manual check from the source instance:**

```bash
nc -zv 10.0.20.15 3306
#   succeeded          → network is fine, look at the app
#   Connection refused → packets arrive, nothing listening → app/host firewall
#   timed out          → dropped → SG or NACL
```

**Most common causes:**

| Cause | Fix |
|---|---|
| Destination SG doesn't allow the source SG | `authorize-security-group-ingress` with `UserIdGroupPairs` |
| Rule uses a CIDR that no longer matches (instance re-IP'd) | Switch to SG referencing |
| Custom NACL on one subnet missing ephemeral outbound | Allow `1024-65535` both ways |
| App bound to `127.0.0.1` | Bind to `0.0.0.0` |
| Wrong port (e.g. app on 8080, SG allows 80) | Fix the rule |
| Instance in a different VPC than you assumed | `describe-instances --query '...VpcId'` |

> **ICMP is not TCP.** `ping` failing while the app works is normal — most SGs don't allow ICMP. Test the real port with `nc`, not `ping`.

### 4.2 Instance can reach some subnets but not others

Different subnets can have **different NACLs** and **different route tables**.

```bash
# Compare route tables across all subnets in the VPC
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC" --query \
 'RouteTables[].{RTB:RouteTableId,Subnets:Associations[].SubnetId,Routes:Routes[].[DestinationCidrBlock,GatewayId,NatGatewayId,TransitGatewayId]}'

# Compare NACLs
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC" --query \
 'NetworkAcls[].{NACL:NetworkAclId,Subnets:Associations[].SubnetId,Entries:Entries[].[RuleNumber,Egress,CidrBlock,RuleAction]}'
```

Look for a subnet that isn't explicitly associated with a route table — it silently uses the **main** route table, which is often not what you intended.

### 4.3 Cross-AZ traffic works but is slow or expensive

That's expected: cross-AZ traffic is charged at $0.01/GB **each direction** and adds ~1ms latency. If it's unexpected, check:
- Are private subnets routing to a NAT Gateway in a **different** AZ?
- Is an ALB with cross-zone load balancing sending traffic to distant targets?
- Is a database replica in another AZ being read from synchronously?

---

## 5. Security Groups & NACLs

### 5.1 "I added the rule but it still doesn't work"

```bash
# Verify the rule actually exists on the right SG
aws ec2 describe-security-group-rules --filters "Name=group-id,Values=sg-0abc" --output table

# Verify the instance actually uses that SG
aws ec2 describe-instances --instance-ids i-0abc \
  --query 'Reservations[0].Instances[0].SecurityGroups'

# Verify you edited the right direction (ingress vs egress)
```

**Very common mistakes:**
- Added the rule to the *source* SG's inbound instead of the *destination* SG's inbound.
- Edited a security group in a different VPC with the same name.
- Added it to the ALB's SG when the problem is between ALB and targets.
- The resource has multiple SGs and another one isn't the issue — SGs are additive/permissive, so a second SG never *blocks*; if it still fails, the block is a NACL or routing.

> **Security group changes take effect immediately** — no restart needed. If a change appeared to "need a reboot", the real fix was something else.

### 5.2 SG rule with a CIDR source doesn't match

If the source is behind a NAT Gateway, its packets arrive with the **NAT Gateway's private IP** as source (within the VPC) or the NAT's EIP (from outside). Allow the NAT's address, or better, restructure so you can reference a security group.

If traffic comes through an **ALB**, the source is the ALB's ENI private IPs — reference the ALB's security group, not client IPs.

If traffic comes through an **NLB**:
- With `preserve_client_ip` enabled (default for instance targets), the source is the **original client IP** — SG rules must allow client CIDRs, not the NLB.
- With IP targets, the source is the NLB node's private IP.

### 5.3 Intermittent failures — the ephemeral port trap

**Symptom:** some connections work, others fail, seemingly at random. Different clients behave differently.

**Cause:** a custom NACL allows a narrow ephemeral port range. Return traffic goes to a port chosen randomly by the client OS.

| Client | Ephemeral range |
|---|---|
| Linux | 32768–60999 |
| Windows (Vista+) | 49152–65535 |
| NLB / Lambda / ELB health checks | 1024–65535 |

**Fix:** allow `1024–65535` in the direction of the return traffic.

```bash
aws ec2 create-network-acl-entry --network-acl-id acl-0abc --rule-number 200 \
  --protocol 6 --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow --egress
```

**Rule of thumb:** if a problem is *intermittent* and *client-dependent*, suspect NACLs first.

### 5.4 NACL deny rule isn't working

NACL rules are evaluated **in ascending rule number order, and evaluation stops at the first match.**

```
Rule 100: ALLOW  TCP 80  from 0.0.0.0/0
Rule 200: DENY   ALL     from 198.51.100.5/32     ← never reached for port 80 traffic
```

**Fix:** give the deny rule a **lower** number than the allow.

```bash
aws ec2 delete-network-acl-entry --network-acl-id acl-0abc --rule-number 200 --ingress
aws ec2 create-network-acl-entry --network-acl-id acl-0abc --rule-number 50 \
  --protocol -1 --cidr-block 198.51.100.5/32 --rule-action deny --ingress
```

### 5.5 Everything broke after attaching a custom NACL

A newly created custom NACL **denies all traffic in both directions** until you add rules. (The *default* NACL allows everything — that's the difference.)

```bash
# See what you actually have
aws ec2 describe-network-acls --network-acl-ids acl-0abc --query 'NetworkAcls[].Entries'

# Emergency rollback: reassociate the default NACL
DEFAULT=$(aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC" \
  "Name=default,Values=true" --query 'NetworkAcls[0].NetworkAclId' --output text)
ASSOC=$(aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=subnet-0abc" \
  --query "NetworkAcls[0].Associations[?SubnetId=='subnet-0abc'].NetworkAclAssociationId" --output text)
aws ec2 replace-network-acl-association --association-id $ASSOC --network-acl-id $DEFAULT
```

### 5.6 Hit the security group rule limit

```
RulesPerSecurityGroupLimitExceeded
```

Default is 60 inbound + 60 outbound per SG, 5 SGs per ENI, and a hard ceiling of ~1,000 rules per ENI.

**Fixes, in order of preference:**
1. **Use a managed prefix list** — one rule covers many CIDRs.
2. **Reference security groups** instead of listing IPs.
3. Consolidate overlapping CIDRs (`10.0.1.0/24` + `10.0.2.0/24` → `10.0.0.0/22` where appropriate).
4. Request a quota increase (rules per SG, or SGs per ENI up to 16).

---

## 6. NAT Gateway Problems

### 6.1 Private instances have no internet despite a NAT Gateway

**Run through this exact list:**

```bash
NAT=nat-0abc

# 1. Is the NAT Gateway available?
aws ec2 describe-nat-gateways --nat-gateway-ids $NAT \
  --query 'NatGateways[].{State:State,Subnet:SubnetId,IP:NatGatewayAddresses[0].PublicIp,Fail:FailureMessage}'

# 2. Is it in a PUBLIC subnet? (the #1 mistake)
NAT_SUBNET=$(aws ec2 describe-nat-gateways --nat-gateway-ids $NAT \
  --query 'NatGateways[0].SubnetId' --output text)
NAT_RT=$(aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$NAT_SUBNET" \
  --query 'RouteTables[0].RouteTableId' --output text)
aws ec2 describe-route-tables --route-table-ids $NAT_RT \
  --query 'RouteTables[].Routes[].[DestinationCidrBlock,GatewayId]' --output table
# MUST show 0.0.0.0/0 → igw-xxxx

# 3. Does the PRIVATE subnet route to the NAT?
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=subnet-private" \
  --query 'RouteTables[].Routes[].[DestinationCidrBlock,NatGatewayId]' --output table

# 4. Does the NAT have an Elastic IP? (public NAT only)
# 5. Do the private subnet's NACLs allow outbound + ephemeral inbound?
```

| Finding | Fix |
|---|---|
| NAT is in a private subnet | Delete it and recreate in a public subnet. You cannot move it. |
| NAT's subnet has no IGW route | Add `0.0.0.0/0 → igw-xxx` to that route table |
| Private RT points at the IGW instead of the NAT | Private subnets must point at the NAT, not the IGW |
| NAT state is `failed` | Check `FailureMessage` — usually EIP or subnet IP exhaustion. Recreate. |
| `--connectivity-type private` used by mistake | Private NAT has no internet path. Recreate as public. |

### 6.2 `ErrorPortAllocation` — port exhaustion

**Symptom:** connections start failing under load; the CloudWatch metric `ErrorPortAllocation` is non-zero.

**Cause:** a NAT Gateway supports ~55,000 simultaneous connections **to each unique destination IP:port**. Hammering one endpoint (a single API host, one S3 IP) exhausts the port pool.

**Fixes:**
1. **Add secondary Elastic IPs** to the NAT Gateway — each adds another 55k ports per destination.
   ```bash
   aws ec2 associate-nat-gateway-address --nat-gateway-id nat-0abc --allocation-ids eipalloc-0new
   ```
2. **Use VPC endpoints** for AWS services so that traffic never touches the NAT at all.
3. **Fix the application** — enable HTTP keep-alive and connection pooling instead of opening a new connection per request.
4. **Distribute across more NAT Gateways** (one per AZ minimum).

### 6.3 Long-lived connections drop after ~350 seconds

**Cause:** NAT Gateway idle timeout is 350 seconds. An idle connection is dropped, and the NAT sends no RST — the client just hangs until its own timeout.

**Fixes:**
- Enable **TCP keepalive** on the client with an interval below 350s.
  ```bash
  sudo sysctl -w net.ipv4.tcp_keepalive_time=240
  sudo sysctl -w net.ipv4.tcp_keepalive_intvl=30
  sudo sysctl -w net.ipv4.tcp_keepalive_probes=5
  ```
- Configure application-level keepalive/heartbeats (common fix for database connection pools and long-poll APIs).
- Watch the `IdleTimeoutCount` metric to confirm.

### 6.4 NAT Gateway costs more than expected

See [§18.1](#181-the-nat-gateway-bill). Short version: add an S3 gateway endpoint, route in-AZ, and check Flow Logs for the top destinations.

### 6.5 NAT instance (not gateway) doesn't forward traffic

```bash
# Source/destination check MUST be disabled
aws ec2 modify-instance-attribute --instance-id i-0nat --no-source-dest-check

# IP forwarding and masquerade must be configured in the OS
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Also ensure the NAT instance's security group allows inbound traffic from the private subnet CIDRs — unlike a NAT Gateway, a NAT instance **has** a security group.

---

## 7. DNS Problems

### 7.1 Instance cannot resolve any hostnames

```bash
# From the instance
cat /etc/resolv.conf              # should point to VPC CIDR base + 2, e.g. 10.0.0.2
dig @10.0.0.2 amazon.com
dig @169.254.169.253 amazon.com   # link-local resolver — works regardless of CIDR
```

**Checks:**

```bash
aws ec2 describe-vpc-attribute --vpc-id $VPC --attribute enableDnsSupport
aws ec2 describe-vpc-attribute --vpc-id $VPC --attribute enableDnsHostnames
```

Both should be `true`. If `enableDnsSupport` is false, the `.2` resolver simply doesn't answer.

**Also check:**
- A custom **DHCP option set** pointing at DNS servers that are unreachable or down. Revert with:
  ```bash
  aws ec2 associate-dhcp-options --dhcp-options-id default --vpc-id $VPC
  ```
- NACLs blocking **UDP 53** (and TCP 53 for large responses).
- Security group egress blocking UDP 53.

### 7.2 Intermittent `SERVFAIL` under load

**Cause:** the Route 53 Resolver enforces a hard limit of **1,024 packets per second per ENI**. Containerised, high-concurrency workloads exceed this easily.

**Fixes:**

1. **Cache locally on the host:**
   ```bash
   sudo dnf install -y dnsmasq   # or nscd, or systemd-resolved caching
   ```
2. **On EKS, deploy NodeLocal DNSCache.**
3. **Reduce `ndots`** — the default `ndots:5` in Kubernetes turns one lookup into five.
   ```yaml
   dnsConfig:
     options:
       - name: ndots
         value: "2"
   ```
4. **Use fully-qualified names with a trailing dot** (`s3.amazonaws.com.`) to skip search-domain expansion.
5. Spread load across more ENIs / more nodes.

### 7.3 Private hosted zone records don't resolve

- The private hosted zone must be **associated with the VPC**.
  ```bash
  aws route53 list-hosted-zones-by-vpc --vpc-id $VPC --vpc-region us-east-1
  aws route53 associate-vpc-with-hosted-zone --hosted-zone-id Z123 \
    --vpc VPCRegion=us-east-1,VPCId=$VPC
  ```
- `enableDnsHostnames` **and** `enableDnsSupport` must both be true.
- For cross-account association, the owner must authorise first (`create-vpc-association-authorization`).
- Over a **peering** connection, enable DNS resolution options on both sides:
  ```bash
  aws ec2 modify-vpc-peering-connection-options --vpc-peering-connection-id pcx-0abc \
    --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
    --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true
  ```

### 7.4 On-prem can't resolve AWS names (or vice versa)

You need **Route 53 Resolver endpoints**:

| Direction | Endpoint type |
|---|---|
| On-prem servers resolving AWS private zones | **Inbound** resolver endpoint |
| AWS instances resolving `corp.internal` | **Outbound** resolver endpoint + forwarding rules |

Check the resolver endpoint's security group allows **TCP and UDP 53** from the appropriate source.

### 7.5 A service name resolves to a public IP instead of the endpoint

You created an interface endpoint but didn't enable private DNS, or DNS support is off.

```bash
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-0abc --private-dns-enabled
```

Verify from the instance — the answer should be a private VPC address:

```bash
dig +short secretsmanager.us-east-1.amazonaws.com
# 10.0.10.55   ✅ endpoint working
# 52.94.x.x    ❌ still going public
```

---

## 8. VPC Endpoint Problems

### 8.1 Interface endpoint created but connections time out

**The overwhelmingly common cause: the endpoint's security group.**

An interface endpoint is an ENI, and that ENI has a security group. If it doesn't allow inbound **TCP 443** from your workload subnets, every call hangs with no useful error.

```bash
# Which SGs are on the endpoint?
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids vpce-0abc \
  --query 'VpcEndpoints[].Groups'

# Fix
aws ec2 authorize-security-group-ingress --group-id sg-endpoints \
  --ip-permissions 'IpProtocol=tcp,FromPort=443,ToPort=443,IpRanges=[{CidrIp=10.0.0.0/16,Description="VPC HTTPS"}]'
```

**Other causes:**

| Cause | Check |
|---|---|
| Endpoint not deployed in the AZ where your workload runs | `describe-vpc-endpoints --query 'VpcEndpoints[].SubnetIds'` |
| Private DNS disabled | `--query 'VpcEndpoints[].PrivateDnsEnabled'` |
| Endpoint state not `available` | `--query 'VpcEndpoints[].State'` |
| Endpoint policy too restrictive | `--query 'VpcEndpoints[].PolicyDocument'` |
| Wrong service name (region mismatch) | `com.amazonaws.us-east-1.ssm` not `us-west-2` |

### 8.2 Gateway endpoint (S3) not being used

```bash
# The route must exist in the route table for the subnet the workload is in
aws ec2 describe-route-tables --route-table-ids rtb-0abc \
  --query 'RouteTables[].Routes[?DestinationPrefixListId!=null]'
```

**Causes:**
- The gateway endpoint was associated with a *different* route table than the one serving your subnet.
- You're trying to use it from **on-premises over VPN/DX** — gateway endpoints don't work from outside the VPC. Use an interface endpoint for S3 instead.
- You're accessing S3 in a **different region** — the endpoint is region-specific.
- Using an S3 **access point** or a **dualstack/FIPS** hostname not covered by the prefix list.

### 8.3 `AccessDenied` only when going through the endpoint

Three policies stack, and **all** must allow:

```
Endpoint policy  →  Bucket / resource policy  →  IAM identity policy
```

```bash
# Endpoint policy
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids vpce-0abc --query 'VpcEndpoints[].PolicyDocument'

# Bucket policy — look for aws:SourceVpce / aws:SourceVpc conditions
aws s3api get-bucket-policy --bucket my-bucket
```

A bucket policy with `"Condition": {"StringNotEquals": {"aws:SourceVpce": "vpce-OLD"}}` will deny access from a **new** endpoint after you recreate one. Update the condition with the new endpoint ID.

### 8.4 SSM / Session Manager works in one AZ but not another

The interface endpoints were only created in some subnets. Add the missing ones:

```bash
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-0abc --add-subnet-ids subnet-0other
```

---

## 9. Peering Problems

### 9.1 Peering is active but traffic does not flow

**Checklist:**

```bash
PCX=pcx-0abc

# 1. Status must be "active", not "pending-acceptance"
aws ec2 describe-vpc-peering-connections --vpc-peering-connection-ids $PCX \
  --query 'VpcPeeringConnections[].Status'

# 2. Routes must exist on BOTH sides — this is the usual failure
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_A" \
  --query 'RouteTables[].Routes[?VpcPeeringConnectionId!=null]'
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_B" \
  --query 'RouteTables[].Routes[?VpcPeeringConnectionId!=null]'

# 3. Security groups on the destination must allow the source CIDR
```

| Problem | Fix |
|---|---|
| Route only on one side | Add the mirror route in the peer VPC |
| Route added to the wrong route table | Add it to the table serving the *workload's* subnet |
| Destination SG only allows the local VPC CIDR | Add the peer CIDR (or reference the peer SG — same-region only) |
| NACL blocking the peer CIDR | Allow it both directions |

### 9.2 Cannot create the peering — CIDR overlap

```
InvalidVpcPeeringConnection: The requester and accepter VPCs have overlapping CIDR blocks
```

There is no way around this. Your options:
1. Re-IP one VPC (correct, painful).
2. Use **PrivateLink** — the consumer reaches your NLB through an ENI in its own space; overlap is irrelevant.
3. Use a **private NAT Gateway** to translate one side into a non-overlapping transit range.
4. Add a **secondary, non-overlapping CIDR** to each VPC and put the interconnected resources there.

### 9.3 Can't reach the peer VPC's internet, NAT, or endpoints

This is **edge-to-edge routing** and is explicitly unsupported. Over a peering you cannot use the other VPC's:
- Internet Gateway
- NAT Gateway or NAT instance
- Gateway VPC endpoints
- VPN or Direct Connect connections

**Fix:** give the VPC its own path, or use a **Transit Gateway** (which does support transitive routing) with a centralised egress VPC.

### 9.4 Peering works for some subnets only

Each subnet uses its own route table. Add the peering route to **every** route table that needs it — including the main route table if some subnets fall back to it.

### 9.5 Cannot resolve the peer VPC's private DNS names

```bash
aws ec2 modify-vpc-peering-connection-options --vpc-peering-connection-id $PCX \
  --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
  --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true
```

---

## 10. Transit Gateway Problems

### 10.1 TGW attachment exists but no connectivity

**There are three routing layers with a TGW. All three must be right.**

```
1. VPC subnet route table  → destination CIDR → tgw-xxx
2. TGW route table (of the SENDING attachment) → destination CIDR → target attachment
3. VPC subnet route table on the RECEIVING side → return CIDR → tgw-xxx
```

```bash
# Layer 1 & 3 — VPC route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC" \
  --query 'RouteTables[].Routes[?TransitGatewayId!=null]'

# Layer 2 — TGW route table for the sending attachment
ASSOC_RTB=$(aws ec2 get-transit-gateway-route-table-associations \
  --transit-gateway-route-table-id tgw-rtb-0abc \
  --query 'Associations[].TransitGatewayAttachmentId' --output text)
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id tgw-rtb-0abc \
  --filters "Name=state,Values=active" --output table
```

| Symptom | Cause | Fix |
|---|---|---|
| Attachment shows `available` but no routes learned | Propagation not enabled | `enable-transit-gateway-route-table-propagation` |
| Routes exist in one TGW route table but the attachment uses another | Wrong association | `associate-transit-gateway-route-table` |
| VPC route table missing the TGW route | — | `create-route --transit-gateway-id tgw-xxx` |
| Route shows `blackhole` | Target attachment was deleted | Remove the stale static route, re-create the attachment |
| Traffic to a specific CIDR fails | A more specific peering route is winning | Check longest-prefix match; remove the peering route |

### 10.2 TGW attachment subnets

The TGW places an ENI in **one subnet per AZ** you select. If you didn't select a subnet in an AZ, resources in that AZ **cannot use the TGW at all**.

```bash
aws ec2 describe-transit-gateway-vpc-attachments --transit-gateway-attachment-ids tgw-attach-0abc \
  --query 'TransitGatewayVpcAttachments[].SubnetIds'

aws ec2 modify-transit-gateway-vpc-attachment --transit-gateway-attachment-id tgw-attach-0abc \
  --add-subnet-ids subnet-0missing
```

> **Best practice:** create dedicated tiny `/28` subnets in each AZ purely for TGW attachments. Keeps ENIs out of workload subnets and makes NACLs simpler.

### 10.3 Asymmetric routing breaks a stateful firewall

**Symptom:** traffic through an inspection VPC works one way, gets dropped the other way; the firewall logs show "no matching session".

**Cause:** without appliance mode, the TGW may choose a different AZ for the return path, and the firewall in that AZ has no state for the flow.

**Fix:**

```bash
aws ec2 modify-transit-gateway-vpc-attachment \
  --transit-gateway-attachment-id tgw-attach-0firewall \
  --options ApplianceModeSupport=enable
```

### 10.4 Cross-account attachment stuck in `pendingAcceptance`

```bash
aws ec2 accept-transit-gateway-vpc-attachment --transit-gateway-attachment-id tgw-attach-0abc
```

Or set `AutoAcceptSharedAttachments=enable` on the TGW when you trust the sharing scope.

### 10.5 TGW peering doesn't propagate routes automatically

TGW **peering** attachments do not support route propagation. You must add **static routes** in the TGW route tables on both sides.

```bash
aws ec2 create-transit-gateway-route --transit-gateway-route-table-id tgw-rtb-0local \
  --destination-cidr-block 10.16.0.0/12 --transit-gateway-attachment-id tgw-attach-0peering
```

---

## 11. VPN & Direct Connect Problems

### 11.1 VPN tunnel shows DOWN

```bash
aws ec2 describe-vpn-connections --vpn-connection-ids vpn-0abc --query \
 'VpnConnections[].VgwTelemetry[].{Outside:OutsideIpAddress,Status:Status,Msg:StatusMessage,Routes:AcceptedRouteCount}' \
 --output table
```

The `StatusMessage` is the most useful field. Common values and meanings:

| Message | Meaning | Fix |
|---|---|---|
| `IPSEC IS DOWN` | No IKE/IPsec established | Check the customer gateway public IP, PSK, and that UDP 500/4500 + ESP are permitted |
| `No route to host` / no IKE packets | Traffic isn't reaching AWS | On-prem firewall blocking UDP 500/4500 |
| Phase 1 up, Phase 2 down | Mismatched encryption domain | Proxy IDs / traffic selectors must match on both ends |
| Tunnel up, `AcceptedRouteCount: 0` | BGP session not established or no prefixes advertised | Check ASNs, BGP neighbour config, advertised prefixes |
| Flaps every few minutes | DPD timeouts, or asymmetric NAT on your side | Adjust DPD, check for a NAT device in front of the CGW |

**Also check:**
- **Both tunnels configured.** AWS gives you two; many outages are "we only ever configured tunnel 1".
- **Route propagation enabled** on the VPC route table:
  ```bash
  aws ec2 enable-vgw-route-propagation --route-table-id rtb-0abc --gateway-id vgw-0abc
  ```
- For **static** VPNs, the routes must be added manually with `create-vpn-connection-route`.
- Idle tunnels can go down: generate keepalive traffic or enable DPD with a suitable timeout action.

### 11.2 Tunnel is up but traffic doesn't flow

```
1. VPC route table → on-prem CIDR → vgw-xxx or tgw-xxx     ← check propagation
2. Security groups allow the on-prem CIDR
3. NACLs allow it both directions
4. On-prem firewall allows the AWS CIDR
5. On-prem routing sends AWS CIDRs to the VPN device
6. No overlapping/conflicting routes preferring another path
```

### 11.3 Direct Connect VIF stuck in `down`

| Check | Command / action |
|---|---|
| Physical link | `aws directconnect describe-connections` — `state` should be `available` |
| VLAN mismatch | Confirm the 802.1Q VLAN tag matches your router config |
| BGP peer IPs | `/30` addresses must match exactly on both sides |
| BGP ASN | Your ASN and Amazon side ASN configured correctly |
| BGP MD5 key | Must match |
| Router config | Re-download from the console and re-apply |

### 11.4 Traffic prefers VPN over Direct Connect (or vice versa)

AWS route preference order:
1. Longest prefix match wins first.
2. Static routes beat propagated routes.
3. For equal prefixes: **Direct Connect > VPN**.

To influence path selection, advertise more specific prefixes over the preferred path, or use BGP AS-path prepending on the less preferred one.

### 11.5 Client VPN connects but can't reach anything

```bash
# Authorization rules — required, separate from SGs
aws ec2 describe-client-vpn-authorization-rules --client-vpn-endpoint-id cvpn-endpoint-0abc

# Routes from the endpoint
aws ec2 describe-client-vpn-routes --client-vpn-endpoint-id cvpn-endpoint-0abc
```

| Issue | Fix |
|---|---|
| No authorization rule for the target CIDR | `authorize-client-vpn-ingress --target-network-cidr 10.0.0.0/16 --authorize-all-groups` |
| No route for the destination | `create-client-vpn-route` |
| Split-tunnel enabled but the CIDR isn't pushed | Add the route, or disable split-tunnel |
| Client CIDR overlaps the VPC CIDR | Must not overlap — pick a distinct `/22` |
| Target SG doesn't allow the Client VPN SG | Add the rule |

---

## 12. IP Address Exhaustion & CIDR Problems

### 12.1 `InsufficientFreeAddressesInSubnet`

```bash
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC" \
  --query 'Subnets[].[SubnetId,CidrBlock,AvailableIpAddressCount]' --output table
```

**Remember AWS reserves 5 IPs per subnet.** A `/28` gives you 11, not 16.

**Immediate options:**

| Option | Detail |
|---|---|
| Free up ENIs | `describe-network-interfaces --filters "Name=status,Values=available"` then delete |
| Launch in another subnet/AZ | Fastest workaround |
| Add a **secondary CIDR** to the VPC and create larger subnets | `associate-vpc-cidr-block --cidr-block 10.1.0.0/16` |
| Resize the subnet | **Not possible.** Subnet CIDRs are immutable. |

**Hidden IP consumers people forget:**
- Interface VPC endpoints (one ENI per AZ per endpoint)
- NAT Gateways
- ALB/NLB nodes (8+ IPs per AZ, and they scale)
- Lambda functions in a VPC (Hyperplane ENIs)
- RDS, ElastiCache, Redshift, EKS control plane ENIs
- Transit Gateway attachments
- **EKS pods with the VPC CNI — one VPC IP per pod**

### 12.2 EKS: pods exhausting the subnet

The VPC CNI gives each pod a real VPC IP. A `/24` subnet supports ~250 pods total, across all nodes.

**Fixes:**
1. **Prefix delegation** — assign `/28` blocks per ENI, massively increasing density.
   ```bash
   kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
   ```
2. **Secondary CIDR for pods** from CGNAT space `100.64.0.0/10`, keeping routable space free.
3. Use **larger subnets** (`/20` or bigger) for node groups from the start.

### 12.3 Cannot add a secondary CIDR

```
InvalidVpcRange / CidrConflict
```

**Rules for secondary CIDRs:**
- Must not overlap the existing VPC CIDRs, or any route in the VPC's route tables.
- If the primary is in `10.0.0.0/8`, secondaries generally must also be from RFC 1918 ranges — and specific restricted combinations exist (e.g. you can't add a `172.31.x` block that conflicts with defaults).
- Size between `/16` and `/28`.
- Default limit: 5 CIDRs per VPC (raisable to 50).

### 12.4 CIDR chosen conflicts with something later

Once the **primary** CIDR is set, it can never be changed. If it conflicts with on-prem or a future acquisition, your options are: rebuild the VPC, use PrivateLink to sidestep routing, or use private NAT to translate.

**Prevention:** use **AWS IPAM** from day one and allocate every VPC from a governed pool.

---

## 13. Deletion & Dependency Errors

### 13.1 `DependencyViolation` when deleting a subnet or VPC

Something still holds a network interface. Find it:

```bash
aws ec2 describe-network-interfaces --filters "Name=subnet-id,Values=subnet-0abc" \
  --query 'NetworkInterfaces[].[NetworkInterfaceId,InterfaceType,Description,Status,Attachment.InstanceId]' \
  --output table
```

The `Description` field names the owner:

| Description contains | Owner | Delete via |
|---|---|---|
| `ELB app/...` / `ELB net/...` | Load balancer | `aws elbv2 delete-load-balancer` |
| `VPC Endpoint Interface vpce-...` | Interface endpoint | `aws ec2 delete-vpc-endpoints` |
| `Interface for NAT Gateway nat-...` | NAT Gateway | `aws ec2 delete-nat-gateway` (takes minutes) |
| `AWS Lambda VPC ENI ...` | Lambda in VPC | Delete/unVPC the function; ENIs clear in ~20–40 min |
| `RDSNetworkInterface` | RDS instance | Delete the DB instance |
| `ElastiCache ...` | ElastiCache | Delete the cluster |
| `Amazon EKS ...` | EKS control plane | Delete the cluster |
| `arn:aws:ecs:...` | ECS/Fargate task | Stop the tasks/service |
| `Transit Gateway Attachment` | TGW | `delete-transit-gateway-vpc-attachment` |
| `EFS mount target` | EFS | Delete mount targets |
| `DMS`, `Redshift`, `Workspaces`, `SageMaker` | That service | Delete the resource |
| *(blank)* + status `available` | Orphan | `aws ec2 delete-network-interface` |

### 13.2 Correct deletion order

```
1.  EC2 instances (terminate, wait)
2.  Load balancers, ECS services, EKS clusters
3.  RDS / ElastiCache / Redshift / EFS mount targets
4.  Lambda VPC configuration (ENIs take up to 40 min to clear)
5.  VPC endpoints
6.  NAT Gateways (wait for deleted) → then release EIPs
7.  Transit Gateway attachments → TGW route tables → TGW
8.  VPN connections → VGW detach → VGW delete
9.  Peering connections
10. Flow logs
11. Detach + delete Internet Gateway / Egress-only IGW
12. Orphaned ENIs
13. Subnets
14. Non-main route tables
15. Non-default security groups (may need two passes — cross-references)
16. Non-default NACLs
17. The VPC
```

### 13.3 Cannot delete a security group

```
DependencyViolation: resource sg-0abc has a dependent object
```

**Causes:**
1. It's attached to an ENI → find with:
   ```bash
   aws ec2 describe-network-interfaces --filters "Name=group-id,Values=sg-0abc" \
     --query 'NetworkInterfaces[].[NetworkInterfaceId,Description]'
   ```
2. **Another security group references it** in a rule. Find the referencing groups:
   ```bash
   aws ec2 describe-security-groups \
     --filters "Name=ip-permission.group-id,Values=sg-0abc" \
     --query 'SecurityGroups[].[GroupId,GroupName]'
   ```
   Remove those rules first, or delete groups in dependency order (running the delete loop twice usually resolves circular references).
3. It's the **default** security group — cannot be deleted, only emptied.

### 13.4 Cannot detach the Internet Gateway

```
DependencyViolation: Network vpc-0abc has some mapped public address(es)
```

Something in the VPC still has a public IP or Elastic IP. Find and remove them:

```bash
aws ec2 describe-instances --filters "Name=vpc-id,Values=$VPC" \
  --query 'Reservations[].Instances[?PublicIpAddress!=null].[InstanceId,PublicIpAddress]'
aws ec2 describe-addresses --filters "Name=domain,Values=vpc" \
  --query 'Addresses[].[PublicIp,InstanceId,NetworkInterfaceId]'
```

NAT Gateways also count — delete them first.

### 13.5 Deleted the VPC but resources still cost money

Elastic IPs are **not** released when a NAT Gateway or instance is deleted.

```bash
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]' --output table
# then
aws ec2 release-address --allocation-id eipalloc-0abc
```

---

## 14. Performance & MTU Problems

### 14.1 SSH connects then hangs — MTU / Path MTU Discovery

**Classic symptom:** SSH login works, and then the session freezes as soon as output is large (`ls -la` on a big directory, `cat` of a long file). Small commands work fine.

**Cause:** a packet larger than the path MTU needs fragmentation. The router sends ICMP type 3 code 4 ("Fragmentation Needed"), but a firewall or NACL is blocking ICMP, so the sender never learns and keeps retransmitting.

**Fix:**

```bash
# Allow ICMP destination-unreachable in the NACL
aws ec2 create-network-acl-entry --network-acl-id acl-0abc --rule-number 150 \
  --protocol 1 --icmp-type-code Type=3,Code=4 \
  --cidr-block 0.0.0.0/0 --rule-action allow --ingress

# Allow ICMP in the security group
aws ec2 authorize-security-group-ingress --group-id sg-0abc \
  --protocol icmp --port -1 --cidr 0.0.0.0/0
```

**Workaround** — lower the MTU on the instance:

```bash
sudo ip link set dev eth0 mtu 1500
```

**MTU reference:**

| Path | Max MTU |
|---|---|
| Within a VPC | 9001 (jumbo) |
| Over an Internet Gateway | 1500 |
| Over VPC peering (same region) | 9001 |
| Over VPC peering (cross-region) | 1500 |
| Over Transit Gateway | 8500 |
| Over Site-to-Site VPN | 1500 (often 1436 after overhead) |
| Over Direct Connect | 1500 or 9001 (if enabled on the VIF) |

Test the actual path MTU:

```bash
ping -M do -s 8972 10.0.20.15    # 8972 + 28 = 9000
ping -M do -s 1472 8.8.8.8       # 1472 + 28 = 1500
```

### 14.2 Throughput far below the instance's rated bandwidth

| Cause | Detail / fix |
|---|---|
| **Single-flow 5 Gbps cap** | One TCP flow to a destination outside a cluster placement group is capped at 5 Gbps. Use multiple parallel connections. |
| Burstable instance credits exhausted | T-family instances have baseline + burst network. Check `CPUCreditBalance` and network baselines; move to M/C family. |
| Enhanced networking not enabled | `aws ec2 describe-instances --query '...EnaSupport'` should be `true` |
| Cross-AZ or cross-region path | Adds latency and cost; keep hot paths in-AZ |
| MTU 1500 where 9001 is possible | Enable jumbo frames within the VPC |
| Traffic hairpinning through a NAT Gateway | Add VPC endpoints so it doesn't |
| Instance too small | Network performance scales with instance size |

Measure it properly:

```bash
# On the server
iperf3 -s
# On the client — parallel streams to bypass the single-flow cap
iperf3 -c 10.0.20.15 -P 8 -t 30
```

### 14.3 High latency between instances

- Different AZs adds ~1ms. Different regions adds tens of ms.
- Use a **cluster placement group** for latency-sensitive workloads in the same AZ.
- Enable **ENA Express** (SRD) for lower tail latency on supported instance types.
- Check for a NAT Gateway or firewall appliance in the path that shouldn't be there.

### 14.4 Connections reset after a period of inactivity

| Component | Idle timeout | Fix |
|---|---|---|
| NAT Gateway | 350 s | TCP keepalive < 350 s |
| ALB | 60 s (configurable) | Raise the idle timeout attribute |
| NLB | 350 s (TCP) | TCP keepalive |
| Network Firewall | Configurable | Adjust the stateful engine settings |

---

## 15. Session Manager / SSH Access Problems

### 15.1 `TargetNotConnected`

The SSM Agent can't reach the SSM service. All of the following must be true:

```bash
# 1. Instance profile with the right policy
aws ec2 describe-instances --instance-ids i-0abc \
  --query 'Reservations[0].Instances[0].IamInstanceProfile'
# → must have AmazonSSMManagedInstanceCore attached to the role

# 2. Network path to SSM: either NAT/IGW, or all three interface endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC" \
  --query 'VpcEndpoints[].ServiceName'
# → need ssm, ssmmessages, AND ec2messages

# 3. Endpoint security group allows 443 from the instance subnets

# 4. Is the instance registered?
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=i-0abc"

# 5. Is the agent running? (check via EC2 console → instance screenshot, or user-data logs)
sudo systemctl status amazon-ssm-agent
sudo journalctl -u amazon-ssm-agent -n 50
```

| Missing piece | Fix |
|---|---|
| No IAM role | Attach `AmazonSSMManagedInstanceCore` |
| Only `ssm` endpoint created | You need **all three**: `ssm`, `ssmmessages`, `ec2messages` |
| Endpoint SG blocks 443 | Allow inbound 443 from the VPC CIDR |
| Private DNS not enabled on endpoints | `modify-vpc-endpoint --private-dns-enabled` |
| Agent outdated | `sudo dnf update -y amazon-ssm-agent && sudo systemctl restart amazon-ssm-agent` |
| Instance just launched | Registration takes 2–5 minutes |
| KMS endpoint missing (with session encryption on) | Add the `kms` interface endpoint |

### 15.2 SSH: `Permission denied (publickey)`

Not a networking problem, but it's what people hit next:
- Wrong username: `ec2-user` (Amazon Linux/RHEL), `ubuntu` (Ubuntu), `admin` (Debian), `centos`, `fedora`.
- Wrong key pair, or key permissions too open → `chmod 400 key.pem`.
- The AMI doesn't have the key baked in (custom AMI).

### 15.3 SSH: `Connection timed out`

That **is** a networking problem — go to [§3.3](#33-cannot-connect-to-a-public-instance-from-outside).

---

## 16. Load Balancer & Target Health Problems

### 16.1 Target group shows `unhealthy`

```bash
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:...
```

| `Reason` | Meaning | Fix |
|---|---|---|
| `Target.Timeout` | Health check got no response | Target SG must allow the **ALB's SG** on the health check port |
| `Target.FailedHealthChecks` | Response received, wrong status code | Check the health check path and expected matcher (e.g. 200 vs 302) |
| `Target.NotRegistered` | Target isn't in the group | Register it |
| `Target.NotInUse` | Target isn't in an AZ enabled on the LB | Enable that subnet/AZ on the load balancer |
| `Elb.RegistrationInProgress` | Just wait | — |
| `Target.ResponseCodeMismatch` | App returns e.g. 403 | Fix the app or adjust the matcher |

**The most common root cause:** the target's security group allows the ALB's *subnet CIDR* instead of the ALB's *security group*, and the ALB scaled to new IPs. Always reference the SG.

### 16.2 ALB itself is unreachable

- ALB must be in **at least two subnets in two AZs**, and those subnets must be **public** (IGW route) for an internet-facing ALB.
- Each ALB subnet needs at least **8 free IP addresses** (`/27` minimum, `/26` recommended).
- ALB security group must allow inbound from clients.
- Check the ALB's DNS name resolves: `dig my-alb-123.us-east-1.elb.amazonaws.com`.

### 16.3 NLB client IP confusion

- NLB with **instance targets** preserves the client IP by default. Your target SG must allow the **client** CIDRs, not the NLB.
- NLB with **IP targets** does not preserve it by default.
- Because NLB traffic bypasses SGs in some configurations, always verify with `describe-target-health` and Flow Logs.

---

## 17. IPv6 Problems

### 17.1 IPv6 assigned but no connectivity

```bash
# Does the subnet have an IPv6 CIDR?
aws ec2 describe-subnets --subnet-ids subnet-0abc --query 'Subnets[].Ipv6CidrBlockAssociationSet'

# Is there an IPv6 default route?
aws ec2 describe-route-tables --route-table-ids rtb-0abc \
  --query 'RouteTables[].Routes[?DestinationIpv6CidrBlock!=null]'
```

| Missing | Fix |
|---|---|
| No `::/0` route | Add `--destination-ipv6-cidr-block ::/0 --gateway-id igw-xxx` (public) or `--egress-only-internet-gateway-id eigw-xxx` (private) |
| Interface has no IPv6 address | `modify-subnet-attribute --assign-ipv6-address-on-creation`, then relaunch, or `assign-ipv6-addresses` on the ENI |
| OS not configured | Enable DHCPv6/SLAAC in the guest |

### 17.2 Security groups allow IPv4 but block IPv6

**This is both a connectivity bug and a security bug.** `0.0.0.0/0` does **not** cover `::/0`. They are entirely separate rule sets.

```bash
# Audit: anything open to the IPv6 world
aws ec2 describe-security-groups \
  --filters "Name=ip-permission.ipv6-cidr,Values=::/0" \
  --query 'SecurityGroups[].[GroupId,GroupName]' --output table

# Add the matching IPv6 rule
aws ec2 authorize-security-group-ingress --group-id sg-0abc \
  --ip-permissions 'IpProtocol=tcp,FromPort=443,ToPort=443,Ipv6Ranges=[{CidrIpv6=::/0}]'
```

Same for NACLs — IPv6 entries are separate from IPv4 entries.

### 17.3 Instance reachable from the internet unexpectedly over IPv6

**All AWS IPv6 addresses are globally routable.** There is no "private IPv6" equivalent to RFC 1918. If your subnet has an IPv6 CIDR and a `::/0 → igw` route, that instance is on the public internet — regardless of its IPv4 configuration.

**Fix:** use an **Egress-Only Internet Gateway** for private tiers, and audit IPv6 SG/NACL rules separately.

---

## 18. Cost Surprises

### 18.1 The NAT Gateway bill

Typically the largest VPC line item. Roughly $0.045/hr + $0.045/GB processed (US regions).

**Diagnose where the traffic goes:**

```sql
-- CloudWatch Logs Insights on Flow Logs
fields @timestamp, srcaddr, dstaddr, bytes, pkt_srcaddr
| filter srcaddr like /^10\./ and not (dstaddr like /^10\./)
| stats sum(bytes)/1024/1024/1024 as GB by dstaddr, pkt_srcaddr
| sort GB desc
| limit 30
```

`pkt_srcaddr` gives you the original instance behind the NAT — that's how you find the culprit.

**Fixes, in impact order:**

| Fix | Typical saving |
|---|---|
| S3 + DynamoDB **gateway endpoints** (free) | Often 40–70 % of NAT data |
| Interface endpoints for ECR, CloudWatch Logs, SSM | Large for container workloads |
| Route each private subnet to the NAT GW **in its own AZ** | Removes double-charged cross-AZ transfer |
| Centralised egress VPC for many accounts | Consolidates hourly charges |
| Cache container images / packages locally | Removes repeated pulls |

### 18.2 Public IPv4 charges

Since **1 February 2024**, every public IPv4 address is billed ~$0.005/hr — attached or not, EIP or auto-assigned.

```bash
# Count public IPs on instances
aws ec2 describe-instances --query \
  'Reservations[].Instances[?PublicIpAddress!=null].[InstanceId,PublicIpAddress]' --output table

# Unattached EIPs — pure waste
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]' --output table
```

**Fixes:** release unused EIPs, disable `MapPublicIpOnLaunch` on subnets that don't need it, put workloads behind a shared load balancer, and adopt IPv6 where possible (IPv6 addresses are free).

### 18.3 Cross-AZ data transfer

$0.01/GB **each direction**, so $0.02/GB round trip. Silent and cumulative.

**Common sources:**
- Private subnets routed to a NAT Gateway in another AZ
- ALB cross-zone load balancing to distant targets
- Database replicas in another AZ
- Chatty microservices spread across AZs without zone-aware routing

### 18.4 Interface endpoint sprawl

~$0.01/hr per endpoint **per AZ**. Ten services × three AZs ≈ $220/month before any data.

```bash
aws ec2 describe-vpc-endpoints --filters "Name=vpc-endpoint-type,Values=Interface" \
  --query 'VpcEndpoints[].[VpcEndpointId,ServiceName,length(SubnetIds)]' --output table
```

Deploy only in AZs where workloads actually run, and remove endpoints for services you no longer call.

### 18.5 Idle Transit Gateway attachments

~$0.05/hr each, forever, whether traffic flows or not.

```bash
aws ec2 describe-transit-gateway-attachments \
  --query 'TransitGatewayAttachments[?State==`available`].[TransitGatewayAttachmentId,ResourceType,ResourceId]' \
  --output table
```

Delete attachments for decommissioned VPCs.

---

## 19. Verbatim Error Message Reference

| Error | Meaning | Fix |
|---|---|---|
| `InvalidVpcID.NotFound` | VPC doesn't exist, or wrong region | Check `--region` and the ID |
| `InvalidSubnetID.NotFound` | Same, for subnets | — |
| `InvalidParameterValue: CIDR block is malformed` | Bad CIDR syntax or size | VPC must be /16–/28; subnet within the VPC range |
| `InvalidSubnet.Conflict` | Subnet CIDR overlaps an existing subnet | Pick a non-overlapping range |
| `InvalidSubnet.Range` | Subnet CIDR isn't inside the VPC CIDR | Fix the range |
| `InsufficientFreeAddressesInSubnet` | No IPs left | §12.1 |
| `NetworkAclEntryAlreadyExists` | That rule number is taken | Use a different number or `replace-network-acl-entry` |
| `InvalidPermission.Duplicate` | SG rule already exists | Nothing to do |
| `InvalidPermission.NotFound` | Trying to revoke a rule that isn't there | Check the exact protocol/port/source |
| `RulesPerSecurityGroupLimitExceeded` | 60-rule limit | §5.6 |
| `DependencyViolation` | Something still references the resource | §13 |
| `InvalidGroup.InUse` | SG attached to an ENI or referenced by another SG | §13.3 |
| `Gateway.NotAttached` | IGW not attached to the VPC | `attach-internet-gateway` |
| `Route already exists` / `RouteAlreadyExists` | Duplicate destination in the table | Use `replace-route` |
| `InvalidRoute.NotFound` | Deleting a route that isn't there | Check the exact destination CIDR |
| `InvalidNatGatewayID.NotFound` | Wrong ID or already deleted | — |
| `NatGatewayNotFound` in a route | NAT deleted while route remains | Delete/replace the route |
| `Resource.AlreadyAssociated` | EIP already associated | Use `--allow-reassociation` or disassociate first |
| `AddressLimitExceeded` | EIP quota (default 5) | Release unused, or request an increase |
| `InvalidVpcPeeringConnection: overlapping CIDR blocks` | Peering CIDR conflict | §9.2 |
| `VpcPeeringConnectionAlreadyExists` | Duplicate peering | Reuse the existing one |
| `InvalidTransitGatewayID.NotFound` | Wrong ID/region | — |
| `IncorrectState` on TGW | Attachment or TGW not yet `available` | Wait and retry |
| `Blackhole` route state on TGW | Target attachment deleted | Remove the stale static route |
| `TargetNotConnected` (SSM) | Agent can't reach SSM | §15.1 |
| `AccessDeniedException` on an S3 call via endpoint | Endpoint/bucket/IAM policy denies it | §8.3 |
| `UnauthorizedOperation` | IAM lacks the EC2 permission | Add the specific `ec2:*` action |
| `RequestLimitExceeded` | API throttling | Exponential backoff; batch calls |
| `VcpuLimitExceeded` | EC2 quota, not networking | Request an increase |
| `InvalidClientVpnEndpointId.NotFound` | Wrong ID | — |
| `Network vpc-xxx has some mapped public address(es)` | Public IPs/EIPs still in the VPC | §13.4 |
| `The maximum number of internet gateways has been reached` | One IGW per VPC | Reuse the existing IGW |
| `InvalidParameterCombination: Cannot specify both a NAT gateway and gateway` | Two targets in one route | Pick one |

---

## 20. Diagnostic Command Toolkit

### The 60-second triage script

```bash
#!/usr/bin/env bash
# Usage: ./triage.sh i-0abc123
INSTANCE=$1

echo "=== INSTANCE ==="
aws ec2 describe-instances --instance-ids $INSTANCE --query \
 'Reservations[0].Instances[0].{State:State.Name,AZ:Placement.AvailabilityZone,Subnet:SubnetId,VPC:VpcId,PrivIP:PrivateIpAddress,PubIP:PublicIpAddress,SGs:SecurityGroups[].GroupId,Profile:IamInstanceProfile.Arn}' \
 --output json

SUBNET=$(aws ec2 describe-instances --instance-ids $INSTANCE --query 'Reservations[0].Instances[0].SubnetId' --output text)
VPC=$(aws ec2 describe-instances --instance-ids $INSTANCE --query 'Reservations[0].Instances[0].VpcId' --output text)

echo "=== ROUTE TABLE ==="
RT=$(aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$SUBNET" \
     --query 'RouteTables[0].RouteTableId' --output text)
if [ "$RT" = "None" ]; then
  echo "(no explicit association — using MAIN route table)"
  RT=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC" "Name=association.main,Values=true" \
       --query 'RouteTables[0].RouteTableId' --output text)
fi
aws ec2 describe-route-tables --route-table-ids $RT --query \
 'RouteTables[].Routes[].[DestinationCidrBlock,GatewayId,NatGatewayId,TransitGatewayId,VpcPeeringConnectionId,State]' \
 --output table

echo "=== SECURITY GROUPS ==="
for SG in $(aws ec2 describe-instances --instance-ids $INSTANCE \
            --query 'Reservations[0].Instances[0].SecurityGroups[].GroupId' --output text); do
  echo "--- $SG ---"
  aws ec2 describe-security-group-rules --filters "Name=group-id,Values=$SG" --query \
   'SecurityGroupRules[].[IsEgress,IpProtocol,FromPort,ToPort,CidrIpv4,ReferencedGroupInfo.GroupId,Description]' \
   --output table
done

echo "=== NACL ==="
aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=$SUBNET" --query \
 'NetworkAcls[].Entries[].[RuleNumber,Egress,Protocol,PortRange.From,PortRange.To,CidrBlock,RuleAction]' \
 --output table

echo "=== IGW / NAT ==="
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC" \
  --query 'InternetGateways[].[InternetGatewayId,Attachments[0].State]' --output table
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC" \
  --query 'NatGateways[].[NatGatewayId,State,SubnetId,NatGatewayAddresses[0].PublicIp]' --output table

echo "=== ENDPOINTS ==="
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC" \
  --query 'VpcEndpoints[].[VpcEndpointId,ServiceName,VpcEndpointType,State]' --output table
```

### From inside the instance

```bash
# Address & routing
ip addr show ; ip route show ; ip -6 route show

# What am I seen as externally?
curl -s https://checkip.amazonaws.com

# Port test — the single most useful command
nc -zv 10.0.20.15 3306
timeout 5 bash -c '</dev/tcp/10.0.20.15/3306' && echo OPEN || echo CLOSED

# What's listening locally?
ss -tulnp

# DNS
dig @10.0.0.2 amazonaws.com
dig +short ssm.us-east-1.amazonaws.com   # should be 10.x if endpoint works
cat /etc/resolv.conf

# Path
traceroute -n 8.8.8.8
mtr -rw 10.0.20.15

# MTU
ping -M do -s 1472 8.8.8.8

# Capture
sudo tcpdump -i any -nn port 3306 -c 50

# Host firewall — the layer people forget
sudo iptables -L -n -v
sudo systemctl status firewalld

# Metadata (IMDSv2)
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4
```

### Flow Log queries for diagnosis

```sql
-- Everything to/from one instance
fields @timestamp, srcaddr, dstaddr, srcport, dstport, protocol, action, tcp_flags
| filter srcaddr = "10.0.10.42" or dstaddr = "10.0.10.42"
| sort @timestamp desc | limit 100

-- Only rejects, grouped
fields srcaddr, dstaddr, dstport
| filter action = "REJECT"
| stats count(*) as hits by srcaddr, dstaddr, dstport
| sort hits desc | limit 25

-- Half-open connections (SYN with no reply) = firewall block signature
fields srcaddr, dstaddr, dstport, tcp_flags
| filter tcp_flags = 2
| stats count(*) as syns by srcaddr, dstaddr, dstport
| sort syns desc

-- Top egress destinations (NAT cost driver)
fields dstaddr, bytes, pkt_srcaddr
| filter flow_direction = "egress"
| stats sum(bytes)/1024/1024/1024 as GB by dstaddr, pkt_srcaddr
| sort GB desc | limit 30
```

---

## Prevention Checklist

Most of the problems in this document are avoidable:

- [ ] Plan CIDRs org-wide before building; use **IPAM**
- [ ] Size subnets with 2–3× headroom, especially for EKS
- [ ] **One NAT Gateway per AZ**, each private subnet routed in-AZ
- [ ] Reference **security groups**, never hardcoded IPs
- [ ] Leave NACLs permissive unless compliance requires otherwise; if you use them, always allow ephemeral `1024–65535`
- [ ] Add **S3 and DynamoDB gateway endpoints** on day one — free, immediate savings
- [ ] Create the **three SSM endpoints** so you never need a bastion
- [ ] Enable **VPC Flow Logs** at VPC level with `pkt-srcaddr` and `tcp-flags`
- [ ] Alarm on NAT `ErrorPortAllocation` and VPN `TunnelState` (per tunnel)
- [ ] Audit **IPv6 rules separately** from IPv4
- [ ] Run **Network Access Analyzer** periodically
- [ ] Save **Reachability Analyzer** paths for critical flows as living documentation
- [ ] Define everything in **Terraform/CloudFormation** — a hand-built VPC is an undocumented VPC
- [ ] Tag everything: `Name`, `Env`, `Tier`, `Owner`, `CostCenter`

---

**Theory:** [`README.md`](README.md) · **Commands:** [`commands-cheatsheet.md`](commands-cheatsheet.md) · **Practice:** [`hands-on-labs.md`](hands-on-labs.md)
