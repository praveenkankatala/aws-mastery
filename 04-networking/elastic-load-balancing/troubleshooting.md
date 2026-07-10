# Troubleshooting — ELB & Auto Scaling

Common failure modes, timing gotchas, and guardrails to know before they bite you in production. Cross-references [`README.md`](./README.md) for the underlying concepts.

## Contents
- [Scaling Thrashing / Over-Provisioning](#scaling-thrashing--over-provisioning)
- [Instance Terminated Faster Than Expected During Scale-In](#instance-terminated-faster-than-expected-during-scale-in)
- [Health Check Doesn't Seem to Respect Cooldown](#health-check-doesnt-seem-to-respect-cooldown)
- [Sticky Sessions Aren't Sticking](#sticky-sessions-arent-sticking)
- [Instance Stuck in Pending:Wait / Terminating:Wait](#instance-stuck-in-pendingwait--terminatingwait)
- [ASG Won't Launch Any Instances](#asg-wont-launch-any-instances)
- [Target Group Shows Unhealthy Hosts](#target-group-shows-unhealthy-hosts)
- [New AMI Deployed but Old Instances Still Running](#new-ami-deployed-but-old-instances-still-running)
- [Session/Cart Data Disappears on Scale-In](#sessioncart-data-disappears-on-scale-in)
- [Launch Configuration Won't Let Me Edit Something](#launch-configuration-wont-let-me-edit-something)

---

## Scaling Thrashing / Over-Provisioning

**Symptom**: The ASG keeps adding instances every minute even though load hasn't actually grown 5x.

**Cause**: No cooldown period (or one set too short) between scaling activities. CloudWatch keeps sampling the *same* CPU spike while new instances are still booting, and the ASG interprets that stale metric as "still need more capacity."

**Fix**:
- Confirm the Default Group Cooldown is set (AWS default: 300s) for Simple/Step Scaling policies.
- If using Target Tracking, check the **Instance Warmup** value — instances still warming up are excluded from the aggregate metric, which should prevent double-counting. If warmup is set too low, raise it to roughly match your actual boot + app-ready time.

---

## Instance Terminated Faster Than Expected During Scale-In

**Symptom**: You expected a slow, safe scale-in but instances vanish quickly, or conversely, taking longer than expected to leave the fleet.

**Cause**: Two separate timers stack when an instance is behind an ALB:
1. **ALB Deregistration Delay** (connection draining) — e.g., 300s
2. **ASG scale-in cooldown** — starts only *after* deregistration finishes, e.g., another 300s

**Fix**: Do the math — `Deregistration Delay + Scale-in Cooldown` = total time to fully cycle a node out (e.g., 300 + 300 = **600 seconds**). If your on-call runbook assumes instances disappear the moment a scale-in alarm fires, update those assumptions.

---

## Health Check Doesn't Seem to Respect Cooldown

**Symptom**: An unhealthy instance gets replaced immediately even though a cooldown timer is supposedly still active.

**This is expected behavior, not a bug**: Cooldown periods **never** delay or block self-healing. If an instance fails an EC2 status check or ALB health check, the ASG terminates and replaces it instantly regardless of remaining cooldown time. Cooldowns only throttle *capacity-driven* scaling actions, not health-driven replacements.

---

## Sticky Sessions Aren't Sticking

**Checklist**:
- Is stickiness actually **enabled on the Target Group** (not just assumed)? Check `stickiness.enabled = true`.
- Is the client actually **sending the cookie back**? Test with `curl -c cookies.txt ...` then `curl -b cookies.txt ...` — a plain `curl` with no cookie jar will never look sticky because it never stores or resends `AWSALB`.
- Is the cookie **expired**? Check the stickiness duration (`stickiness.lb_cookie.duration_seconds`) against how long your test took.
- In a browser, check DevTools → Application → Cookies for `AWSALB` under your site's domain — if it's missing, stickiness likely isn't enabled on the target group the listener is actually forwarding to.
- Remember: **NLB does not support cookie-based stickiness** the way ALB does (it uses a different, IP/flow-based approach) — verify you're testing against an ALB, not an NLB, if you were expecting `AWSALB` cookies specifically.

---

## Instance Stuck in Pending:Wait / Terminating:Wait

**Symptom**: New instances never reach `InService`, or terminating instances never fully disappear.

**Likely causes**:
- A **Lifecycle Hook** is attached and waiting for `complete-lifecycle-action` to be called with `CONTINUE`, but the automation (Lambda/script) never fired it — check EventBridge/SNS/SQS delivery and Lambda logs.
- The **Heartbeat Timeout** hasn't been reached yet — it defaults to 1 hour, which can look like a "stuck" instance if you expected faster automation.
- If a long-running setup task legitimately needs more time, make sure it's calling `record-lifecycle-action-heartbeat` to reset the clock — otherwise it will hit the timeout and fall through to `default_result` (commonly `ABANDON`, which tears the instance down).
- Check the **global hard cap**: 48 hours or 100× the heartbeat timeout, whichever is smaller — no amount of heartbeating gets you past that.

---

## ASG Won't Launch Any Instances

**Checklist**:
- Does the **Launch Template** reference a valid, existing AMI in the current region?
- Are the referenced **subnets** and **security groups** valid and in the same VPC?
- Is **Desired Capacity** actually above 0? (An ASG with Desired = 0 will sit idle by design.)
- Check the **Activity** tab / `describe-scaling-activities` for the actual API error — most "silent" launch failures are IAM permission issues, invalid AMI IDs, or exceeding your account's EC2 service quota (vCPU limits) in that region/AZ.

---

## Target Group Shows Unhealthy Hosts

**Checklist**:
- Does the **health check path** (e.g., `/` or `/health`) actually return a `200` on that port from that instance? Test directly: `curl http://<instance-private-ip>:80/`.
- Is the **security group** on the instance allowing inbound traffic from the **load balancer's security group** on the health check port?
- Are the **Healthy/Unhealthy threshold counts** and **interval/timeout** reasonable for your app's actual boot time? An app that takes 90s to become ready but has a 30s grace period will get marked unhealthy before it's ever had a chance.
- For ASG-attached instances, confirm the **health check grace period** (e.g., 300s) is long enough to cover full boot + User Data execution before ELB health checks start counting against it.

---

## New AMI Deployed but Old Instances Still Running

**This is expected behavior**: Updating a Launch Template (new version) does **not** retroactively touch already-running instances. You must explicitly trigger an **Instance Refresh** to roll the fleet onto the new version. Without it, old and new AMI versions can run side-by-side indefinitely.

---

## Session/Cart Data Disappears on Scale-In

**Cause**: ASGs are designed for **stateless** applications. If session/cart data is stored only in local server memory (not an external store like Redis/ElastiCache or a database like RDS), that data is permanently lost the moment the instance holding it is terminated during scale-in or a health-check replacement.

**Fix**: Externalize session state (Redis/Memcached/DynamoDB) so any backend instance can serve any user's session, or enable ALB **sticky sessions** as a partial mitigation — though stickiness alone doesn't protect against the *replacement* of the very instance holding the data; it only keeps the same client on the same instance while that instance is alive.

---

## Launch Configuration Won't Let Me Edit Something

**This is expected behavior, not a bug**: Launch Configurations are strictly immutable — you cannot modify User Data, AMI, or any other field after creation. You must create an entirely new Launch Configuration and repoint the ASG to it. This is one of the core reasons AWS recommends migrating to **Launch Templates**, which support in-place versioning (`$Latest`/`$Default`) instead.
