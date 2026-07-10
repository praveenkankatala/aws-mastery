# AWS VPC — Hands-On Labs

> Guided, buildable labs that take you from a basic VPC to a full production-style hybrid network. Each lab includes an objective, prerequisites, step-by-step instructions, a validation check, and a cleanup command set. Labs are ordered by increasing complexity and build on concepts from earlier labs.

## Prerequisites (All Labs)

- An AWS account with permissions for EC2/VPC actions.
- AWS CLI v2 configured (`aws configure`) **or** access to the AWS Console.
- Basic familiarity with SSH (or use **AWS Systems Manager Session Manager** — recommended, avoids opening port 22).
- **Cost warning:** NAT Gateways, Elastic IPs (if unattached), Transit Gateway attachments, and VPN connections are billed hourly. Always run the cleanup commands at the end of each lab.

---

## Lab 1 — Build a Basic VPC with Public & Private Subnets

**Objective:** Understand the minimal building blocks — VPC, subnet, IGW, route table — and confirm a public instance is reachable while a private one is not.

**Steps:**

1. Create a VPC with CIDR `10.0.0.0/16`.
2. Create two subnets: `public-a` (`10.0.1.0/24`) and `private-a` (`10.0.2.0/24`), both in the same AZ.
3. Create and attach an Internet Gateway to the VPC.
4. Create a route table `public-rt` with a route `0.0.0.0/0 → igw-xxxx`; associate it with `public-a`.
5. Leave `private-a` on the VPC's default (main) route table — **do not** add an internet route.
6. Enable "auto-assign public IPv4" on `public-a`.
7. Launch a `t3.micro` EC2 instance in `public-a` and one in `private-a`, both using the default Security Group (adjust to allow ICMP/SSH for testing, ideally via SSM instead of an open port 22).

**Validation:**

- From your laptop, ping/SSH the public instance — should succeed.
- Attempt to reach the private instance directly from the internet — should fail (no route, no public IP).
- From inside the public instance, `curl` the private instance's private IP — should succeed (same VPC, local route).

**Cleanup:**

```bash
aws ec2 terminate-instances --instance-ids i-public i-private
aws ec2 delete-subnet --subnet-id subnet-public-a
aws ec2 delete-subnet --subnet-id subnet-private-a
aws ec2 detach-internet-gateway --vpc-id vpc-xxxx --internet-gateway-id igw-xxxx
aws ec2 delete-internet-gateway --internet-gateway-id igw-xxxx
aws ec2 delete-vpc --vpc-id vpc-xxxx
```

---

## Lab 2 — Full 3-Tier Architecture (Web / App / DB)

**Objective:** Build the production reference architecture from the main `readme.md`, across two Availability Zones.

**Steps:**

1. Create VPC `10.0.0.0/16`.
2. Create 6 subnets across 2 AZs:
   - `pub-a` `10.0.101.0/24`, `pub-b` `10.0.102.0/24`
   - `app-a` `10.0.1.0/24`, `app-b` `10.0.2.0/24`
   - `db-a` `10.0.201.0/24`, `db-b` `10.0.202.0/24`
3. Attach an IGW; create `public-rt` (`0.0.0.0/0 → igw`) and associate with both `pub-*` subnets.
4. Allocate 2 Elastic IPs; create a **NAT Gateway per AZ** (`nat-a` in `pub-a`, `nat-b` in `pub-b`).
5. Create two private route tables (`private-rt-a → nat-a`, `private-rt-b → nat-b`); associate each with the matching-AZ app subnet.
6. Create `db-rt` with **no internet route at all**; associate with both `db-*` subnets (fully isolated tier).
7. Create Security Groups per Section 2.6 of the README: `web-sg` (443 from `0.0.0.0/0`), `app-sg` (8080 from `web-sg`), `db-sg` (5432 from `app-sg`).
8. Deploy an Application Load Balancer in the `pub-*` subnets, an Auto Scaling Group of app instances in `app-*`, and an RDS Multi-AZ instance in a DB Subnet Group spanning `db-a`/`db-b`.

**Validation:**

- ALB DNS name resolves and serves traffic from the internet.
- App instances have no public IP but can reach the internet (test with `curl` from an instance via SSM Session Manager) through NAT.
- RDS is unreachable from the internet and unreachable even from `pub-*` — only reachable from `app-sg`.
- Kill one AZ's NAT Gateway (simulate failure) — confirm the **other AZ's app tier is unaffected**, but the affected AZ's app tier loses internet access (demonstrates why per-AZ NAT matters).

**Cleanup:** Follow the teardown order in `commands-cheatsheet.md` (Cleanup section) — delete ALB/ASG/RDS first, then NAT Gateways, EIPs, route tables, IGW, subnets, VPC.

---

## Lab 3 — NAT Gateway Failure & Cost Trade-off Test

**Objective:** Empirically compare a single shared NAT Gateway vs. one-per-AZ, and observe the blast radius of a NAT Gateway outage.

**Steps:**

1. Start from Lab 2's architecture but temporarily point **both** private route tables at a single NAT Gateway (`nat-a` only).
2. From an instance in `app-b` (different AZ than `nat-a`), confirm outbound internet access still works (cross-AZ, at the cost of a small latency/data-transfer charge).
3. Delete `nat-a`.
4. Attempt outbound internet access from **both** `app-a` and `app-b` — both should now fail, proving the single point of failure.
5. Recreate NAT Gateways per-AZ and re-point the route tables to restore Lab 2's HA design.

**Validation:** Document the observed downtime window and reason about the cost difference (1 NAT GW vs. 2) versus the availability risk.

**Cleanup:** Same as Lab 2.

---

## Lab 4 — Security Groups vs. NACLs in Practice

**Objective:** Directly observe the stateful vs. stateless behavior difference.

**Steps:**

1. Using the Lab 1 VPC (or a fresh minimal VPC), launch an instance in a public subnet with a Security Group allowing inbound TCP 80 only (no explicit outbound rule needed — SG default allows all outbound).
2. Confirm `curl` to the instance's port 80 from your laptop succeeds, and the instance can also reach the internet outbound (SG is stateful; the return leg of any connection it initiates is automatically allowed).
3. Now create a **custom NACL** on the subnet with only an **inbound** allow rule for port 80 — do **not** add an outbound rule for ephemeral ports.
4. Retest `curl` to port 80 — it will **fail or hang**, because the response packet (destined to a high ephemeral port) is blocked by the stateless NACL on the way out.
5. Add an outbound NACL rule allowing TCP `1024–65535` to `0.0.0.0/0`.
6. Retest — now it succeeds.
7. Add an explicit **NACL Deny rule** for your own IP with a lower rule number than the allow rule, and confirm you get blocked at the subnet level even though the Security Group would have allowed you — demonstrating NACLs evaluate in numeric order and support Deny, which SGs cannot do.

**Cleanup:**

```bash
aws ec2 delete-network-acl --network-acl-id acl-xxxx
aws ec2 terminate-instances --instance-ids i-xxxx
```

---

## Lab 5 — VPC Peering Between Two VPCs

**Objective:** Connect two isolated VPCs privately and observe the non-transitive routing limitation.

**Steps:**

1. Create `VPC-A` (`10.0.0.0/16`) and `VPC-B` (`10.1.0.0/16`) — **non-overlapping CIDRs**.
2. Launch a test instance in each.
3. Create a peering connection from `VPC-A` to `VPC-B`; accept it.
4. Add a route in `VPC-A`'s route table: `10.1.0.0/16 → pcx-xxxx`. Add the mirror route in `VPC-B`: `10.0.0.0/16 → pcx-xxxx`.
5. Update both instances' Security Groups to allow ICMP/relevant ports from the peer VPC's CIDR.
6. Validate: ping/curl between the two instances using **private IPs** — should succeed.
7. **(Optional, demonstrates non-transitivity):** Create a third `VPC-C`, peer it with `VPC-B` only. Attempt to reach `VPC-C` from `VPC-A` — it will **fail**, because peering is not transitive, even though `A↔B` and `B↔C` both work.

**Cleanup:**

```bash
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id pcx-xxxx
# then delete route entries, instances, subnets, VPCs for A, B, (C)
```

---

## Lab 6 — VPC Endpoint for S3 (Eliminate NAT Dependency)

**Objective:** Show that private-subnet traffic to S3 can bypass the NAT Gateway entirely, improving both security posture and cost.

**Steps:**

1. Using Lab 2's `app-a` subnet, launch an instance with an IAM role granting `s3:GetObject`/`s3:ListBucket` on a test bucket.
2. From the instance, run `aws s3 ls s3://your-test-bucket` — this currently routes through the NAT Gateway (verify via Flow Logs: destination will show AWS's public S3 IP ranges routed via `nat-xxxx`).
3. Create a **Gateway VPC Endpoint** for S3, targeting `private-rt-a`.
4. Re-run the same `aws s3 ls` command — traffic now uses the endpoint's route (a prefix-list target), never leaving the AWS network or touching the NAT Gateway.
5. Confirm by temporarily removing the NAT route (`0.0.0.0/0`) from `private-rt-a` entirely — S3 access **still works** via the endpoint, while general internet access (e.g., `curl https://example.com`) now fails, proving the endpoint operates independently of NAT.

**Cleanup:**

```bash
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-xxxx
```

---

## Lab 7 — Simulated Site-to-Site VPN (Two VPCs as "Office" and "Cloud")

**Objective:** Because a real on-premises device isn't available in a lab, simulate one using a second VPC with a software VPN endpoint (e.g., StrongSwan on an EC2 instance) acting as the "Customer Gateway," connecting to AWS's managed Site-to-Site VPN.

**Steps:**

1. Treat `VPC-Office` (`192.168.0.0/16`) as your simulated on-premises network; launch an EC2 instance in its public subnet running an IPsec daemon (e.g., StrongSwan), assign it an Elastic IP — this is your **Customer Gateway device**.
2. In `VPC-Cloud` (`10.0.0.0/16`), create a **Virtual Private Gateway** and attach it.
3. Register the office instance's Elastic IP as an AWS **Customer Gateway** resource.
4. Create the **Site-to-Site VPN connection** between the Customer Gateway and the Virtual Private Gateway; download the configuration file AWS generates.
5. Apply the relevant tunnel configuration (pre-shared keys, tunnel IPs) to the StrongSwan instance.
6. Enable **route propagation** on `VPC-Cloud`'s private route table so it learns `192.168.0.0/16` automatically.
7. Add a static route on the office instance pointing `10.0.0.0/16` back through the tunnel.

**Validation:**

- Check tunnel status: `aws ec2 describe-vpn-connections --vpn-connection-ids vpn-xxxx` — look for `UP` status on at least one of the two tunnels.
- Ping a private instance in `VPC-Cloud` from the "office" instance using its private IP, traversing the encrypted tunnel.

**Cleanup:** Delete the VPN connection, detach/delete the Virtual Private Gateway, delete the Customer Gateway resource, terminate the office instance, delete both VPCs.

---

## Lab 8 — Transit Gateway Hub-and-Spoke with 3 VPCs

**Objective:** Demonstrate transitive routing, which Lab 5 (peering) proved is impossible without a hub.

**Steps:**

1. Create three VPCs: `VPC-Prod` (`10.0.0.0/16`), `VPC-Shared` (`10.1.0.0/16`), `VPC-Security` (`10.2.0.0/16`) — all non-overlapping.
2. Create a Transit Gateway.
3. Attach all three VPCs to the TGW (one attachment each, in a designated subnet per VPC).
4. Add a route in each VPC's route table: e.g., in `VPC-Prod`, add `10.1.0.0/16 → tgw-xxxx` and `10.2.0.0/16 → tgw-xxxx` (and the mirrored routes in the other VPCs).
5. Launch a test instance in each VPC; adjust Security Groups to allow ICMP/relevant ports from the other VPCs' CIDRs.
6. Validate transitive routing: from `VPC-Prod`, reach an instance in `VPC-Security` — this works through the TGW hub, unlike the equivalent peering scenario in Lab 5.
7. **(Advanced extension):** Create a second TGW route table, associate only `VPC-Security`'s attachment with it, and use it to demonstrate **route table segmentation** — e.g., allow `VPC-Prod` and `VPC-Shared` to reach each other but force all of their traffic to `VPC-Security` (simulating a centralized inspection VPC pattern) by using separate association/propagation route tables per attachment.

**Cleanup:**

```bash
aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id tgw-attach-xxxx
# repeat for all 3 attachments, then:
aws ec2 delete-transit-gateway --transit-gateway-id tgw-xxxx
# then delete routes, instances, subnets, and all 3 VPCs
```

---

## Lab 9 — VPC Flow Logs Investigation

**Objective:** Use Flow Logs to diagnose a deliberately broken configuration — a core real-world troubleshooting skill.

**Steps:**

1. In Lab 2's architecture, enable VPC Flow Logs (traffic type `ALL`) to a CloudWatch Logs group.
2. Deliberately misconfigure `app-sg` to remove the rule allowing traffic from `web-sg`.
3. Generate traffic from the ALB to the app tier (or simulate with `curl` from a `pub-*` instance to an `app-*` instance's private IP).
4. Open CloudWatch Logs Insights and query for `REJECT` actions on the app instance's ENI:

```
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| sort @timestamp desc
| limit 20
```

5. Identify the rejected flow, correlate it to the missing Security Group rule, fix it, and confirm subsequent flows show `ACCEPT`.

**Cleanup:** Disable Flow Logs, delete the CloudWatch Logs group if no longer needed.

---

## Suggested Progression Summary

| Lab | Concept Proven | Builds On |
|---|---|---|
| 1 | Public vs private subnet basics | — |
| 2 | Full 3-tier HA architecture | Lab 1 |
| 3 | NAT Gateway HA trade-offs | Lab 2 |
| 4 | Stateful (SG) vs stateless (NACL) | Lab 1 |
| 5 | VPC Peering & non-transitivity | Lab 1 |
| 6 | VPC Endpoints reduce NAT dependency | Lab 2 |
| 7 | Site-to-Site VPN hybrid connectivity | Lab 2 |
| 8 | Transit Gateway solves non-transitivity | Labs 5, 7 |
| 9 | Flow Logs for real diagnostics | Lab 2 |

> See `troubleshooting.md` for the diagnostic playbooks referenced throughout these labs.
