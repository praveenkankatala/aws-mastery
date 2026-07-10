# Hands-On Labs — ELB & Auto Scaling

Step-by-step labs you can actually run in an AWS account (free-tier friendly with `t3.micro`). Concepts are explained in [`README.md`](./README.md); every command used here is also in [`commands-cheatsheet.md`](./commands-cheatsheet.md).

## Lab Index
- [Lab 1 — Deploy an ALB + Target Group + Two Backends (Round-Robin)](#lab-1--deploy-an-alb--target-group--two-backends-round-robin)
- [Lab 2 — Prove Sticky Sessions Work](#lab-2--prove-sticky-sessions-work)
- [Lab 3 — Build a Launch Template](#lab-3--build-a-launch-template)
- [Lab 4 — Create a Self-Healing ASG Attached to the ALB](#lab-4--create-a-self-healing-asg-attached-to-the-alb)
- [Lab 5 — Prove Self-Healing (Kill an Instance)](#lab-5--prove-self-healing-kill-an-instance)
- [Lab 6 — Prove Dynamic Scale-Out (CPU Stress)](#lab-6--prove-dynamic-scale-out-cpu-stress)
- [Lab 7 — Add a Lifecycle Hook](#lab-7--add-a-lifecycle-hook)
- [Lab 8 — Roll Out a New AMI with Instance Refresh](#lab-8--roll-out-a-new-ami-with-instance-refresh)
- [Lab 9 — Suspend/Resume ASG Processes](#lab-9--suspendresume-asg-processes)

---

## Lab 1 — Deploy an ALB + Target Group + Two Backends (Round-Robin)

**Goal**: see default round-robin routing in action.

1. Launch two EC2 instances in **private subnets** across two AZs (e.g. `us-east-1a`, `us-east-1b`).
2. Attach a Security Group allowing inbound traffic **only from the ALB's security group**, port 80.
3. Give each instance User Data that serves distinct text:
   - Instance 1 → `"Welcome to Web Server 01"`
   - Instance 2 → `"Welcome to Web Server 02"`
4. **Create the Target Group**: EC2 Console → Target Groups → Create target group.
   - Target type: **Instances**
   - Name: `tg-web-production`
   - Protocol/Port: `HTTP` / `80`
   - Health check path: `/`
   - Advanced settings: Healthy threshold `2`, Unhealthy threshold `2`, Timeout `5s`, Interval `30s`
   - Register both EC2 instances → Create.
5. **Create the ALB**: EC2 Console → Load Balancers → Create → Application Load Balancer.
   - Name: `alb-public-web`, Scheme: **Internet-facing**, IP type: IPv4
   - Network mapping: select your VPC + at least 2 AZs + their **public subnets**
   - Security group: allow inbound HTTP (80) from `0.0.0.0/0`
   - Listener: `HTTP:80` → Default action → Forward to `tg-web-production`
   - Create load balancer, wait for state `Provisioning` → `Active`.
6. **Test it** — copy the ALB's DNS name and fire repeated requests:
   ```bash
   curl http://alb-public-web-123456789.us-east-1.elb.amazonaws.com
   ```
   Expected round-robin output:
   ```
   Response 1: Welcome to Web Server 01
   Response 2: Welcome to Web Server 02
   Response 3: Welcome to Web Server 01
   ```
7. **Bonus — test health checks**: stop Web Server 01 from the EC2 console. Within about a minute the target group flags it unhealthy, and the ALB seamlessly sends 100% of traffic to Web Server 02 with zero user-facing downtime.

---

## Lab 2 — Prove Sticky Sessions Work

**Goal**: see the `AWSALB` cookie override round-robin.

1. On `tg-web-production`, enable **stickiness** (duration-based) — see the CLI command in the cheatsheet, or via Console: Target Group → Attributes → Edit → enable "Stickiness".
2. **Without cookies** (baseline — should still round-robin):
   ```bash
   curl http://my-alb-123456789.us-east-1.elb.amazonaws.com/
   curl http://my-alb-123456789.us-east-1.elb.amazonaws.com/
   curl http://my-alb-123456789.us-east-1.elb.amazonaws.com/
   ```
3. **With a cookie jar** — first request saves the cookie:
   ```bash
   curl -c cookies.txt http://my-alb-123456789.us-east-1.elb.amazonaws.com/
   ```
   Inspect `cookies.txt` — you'll see an `AWSALB` entry with a long hash value.
4. Replay the cookie on every subsequent request:
   ```bash
   curl -b cookies.txt http://my-alb-123456789.us-east-1.elb.amazonaws.com/
   curl -b cookies.txt http://my-alb-123456789.us-east-1.elb.amazonaws.com/
   curl -b cookies.txt http://my-alb-123456789.us-east-1.elb.amazonaws.com/
   ```
   Expected — every response comes from the **same** server this time.
5. **Verify visually in a browser**: open the ALB URL in Chrome → F12 → **Application tab** → Cookies → your site's URL. You'll see `AWSALB` listed with its expiry. As long as it's present, refreshing keeps you on the same backend.

---

## Lab 3 — Build a Launch Template

**Goal**: create the reusable blueprint the ASG will launch from.

1. EC2 Console → Launch Templates → Create launch template.
2. Name: `web-app-template`, check "Provide guidance to help me set up a template".
3. AMI: **Amazon Linux 2023**
4. Instance type: **t3.micro**
5. Security group: allow inbound HTTP (80) from your ALB's security group.
6. Under **Advanced details → User data**, paste:
   ```bash
   #!/bin/bash
   sudo dnf update -y
   sudo dnf install -y httpd
   sudo systemctl start httpd
   sudo systemctl enable httpd
   TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
   INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/instance-id)
   echo "<h1>Hello from ASG Instance: $INSTANCE_ID</h1>" > /var/www/html/index.html
   ```
7. Create the template.

---

## Lab 4 — Create a Self-Healing ASG Attached to the ALB

1. EC2 Console → Auto Scaling Groups → Create Auto Scaling group.
2. Name: `production-web-asg`, select `web-app-template` from Lab 3.
3. **Network**: your VPC, at least 2–3 **private subnets** across different AZs.
4. **Advanced options**:
   - Load balancing: attach to your existing ALB's target group (`tg-web-production`)
   - Health checks: enable **ELB** health checks alongside default EC2 checks
   - Health check grace period: **300 seconds**
5. **Group size and scaling**:
   - Minimum: **2**, Maximum: **5**, Desired: **2**
   - Scaling policy: **Target tracking**, metric = Average CPU Utilization, target = **60**, instance warmup = **300s**
6. Skip notifications/tags, review, and create.

---

## Lab 5 — Prove Self-Healing (Kill an Instance)

1. Confirm the ASG has launched 2 instances (matching Desired Capacity).
2. EC2 Console → select one instance → **Instance State → Terminate**.
3. Go to `production-web-asg` → **Activity** tab.
4. Within a minute you should see:
   ```
   "An instance was taken out of service in response to an EC2 health check."
   "Launching a new EC2 instance"
   ```
5. The ASG detected the fleet dropped below the minimum of 2 and self-healed automatically — no human intervention.

---

## Lab 6 — Prove Dynamic Scale-Out (CPU Stress)

1. SSH into one of your running ASG instances.
2. Peg the CPU:
   ```bash
   sha1sum /dev/zero &
   ```
3. Within ~5 minutes, CloudWatch's average-CPU alarm crosses the 60% target-tracking threshold.
4. Check the ASG's **Activity** tab — Desired Capacity shifts from `2` → `3` (or higher), and a new instance is provisioned to absorb load.
5. Clean up: `kill %1` on the stressed instance so it scales back in once CPU normalizes.

---

## Lab 7 — Add a Lifecycle Hook

**Goal**: pause new instances in `Pending:Wait` to simulate a warm-up/config step before they take traffic.

1. Apply the Terraform lifecycle hook block from the cheatsheet (or use the CLI `put-lifecycle-hook` command) against `production-web-asg`:
   - Transition: `autoscaling:EC2_INSTANCE_LAUNCHING`
   - Heartbeat timeout: `600` seconds
   - Default result: `ABANDON`
2. Trigger a scale-out (e.g., manually raise Desired Capacity, or repeat Lab 6).
3. Watch the new instance sit in `Pending:Wait` in the Activity tab instead of going straight to `InService`.
4. Simulate the automation step completing by calling `complete-lifecycle-action` with `CONTINUE` (see cheatsheet) — the instance then proceeds to `InService`.
5. **Try the failure path**: let the heartbeat timeout expire without completing the action — with `default_result = ABANDON`, the ASG tears down that instance and launches a fresh one automatically.

---

## Lab 8 — Roll Out a New AMI with Instance Refresh

1. Edit `web-app-template` → create a **new version** (e.g., patch the User Data or point to an updated AMI).
2. Point the ASG at the new version (or `$Latest`).
3. Start an Instance Refresh (see cheatsheet `start-instance-refresh`), e.g. `MinHealthyPercentage: 80`.
4. Watch the ASG terminate ~20% of instances at a time, wait for replacements to pass ALB health checks, then proceed batch by batch until the whole fleet is on the new version.
5. Check progress with `describe-instance-refreshes`.

---

## Lab 9 — Suspend/Resume ASG Processes

**Goal**: safely inspect a "degraded" instance without the ASG killing it out from under you.

1. Suspend health-check-driven replacement:
   ```bash
   aws autoscaling suspend-processes \
     --auto-scaling-group-name production-web-asg \
     --scaling-processes HealthCheck
   ```
2. Intentionally break the app on one instance (e.g., stop the httpd service) and confirm the ASG does **not** replace it while suspended.
3. SSH in, inspect logs/stack traces at your leisure.
4. Resume normal operation:
   ```bash
   aws autoscaling resume-processes \
     --auto-scaling-group-name production-web-asg
   ```
5. Confirm the now-unhealthy instance gets cycled out shortly after resuming.

---

## What's Next

These labs cover CLB/ALB fundamentals, target groups, sticky sessions, launch templates, self-healing, dynamic scaling, lifecycle hooks, instance refresh, and process suspension end-to-end. For the topics that don't have a full lab here (Gateway Load Balancer, Least Outstanding Requests, cross-zone LB, SNI, and the AWS Load Balancer Controller for EKS), see the conceptual write-ups in [`README.md`](./README.md#11-advanced-elb-topics) — they're either account/appliance-dependent (GWLB needs a real firewall appliance) or cluster-dependent (EKS) and are best understood conceptually before building.
