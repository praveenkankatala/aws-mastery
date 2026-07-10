# Commands Cheatsheet — ELB & Auto Scaling

Copy-paste ready commands: Terraform, AWS CLI, curl, and bash. Pairs with [`README.md`](./README.md) (concepts) and [`hands-on-labs.md`](./hands-on-labs.md) (full walkthroughs).

## Contents
- [Terraform: Launch Template + ASG](#terraform-launch-template--asg)
- [Terraform: Lifecycle Hook](#terraform-lifecycle-hook)
- [AWS CLI: Auto Scaling Group](#aws-cli-auto-scaling-group)
- [AWS CLI: Lifecycle Hooks](#aws-cli-lifecycle-hooks)
- [AWS CLI: Instance Refresh](#aws-cli-instance-refresh)
- [AWS CLI: Suspend/Resume Processes](#aws-cli-suspendresume-processes)
- [AWS CLI: Target Groups & ELB](#aws-cli-target-groups--elb)
- [User Data Bootstrap Scripts](#user-data-bootstrap-scripts)
- [curl: Sticky Session Proof](#curl-sticky-session-proof)
- [Load Testing CPU (for scale-out demo)](#load-testing-cpu-for-scale-out-demo)

---

## Terraform: Launch Template + ASG

```hcl
# 1. The Blueprint
resource "aws_launch_template" "app_blueprint" {
  name_prefix   = "app-v2-template-"
  image_id      = "ami-0c55b159cbfafe1f0" # Amazon Linux 2
  instance_type = "t3.medium"

  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.app_sg.id]
  }
}

# 2. The Auto Scaling Group
resource "aws_autoscaling_group" "app_asg" {
  desired_capacity    = 3
  max_size            = 6
  min_size            = 2
  target_group_arns   = [aws_lb_target_group.alb_tg.arn] # Connect to Load Balancer
  vpc_zone_identifier = [aws_subnet.private_az_a.id, aws_subnet.private_az_b.id]

  launch_template {
    id      = aws_launch_template.app_blueprint.id
    version = "$Latest"
  }

  health_check_type         = "ELB" # Use application health, not just EC2 liveness
  health_check_grace_period = 300   # Give instances 5 mins to boot before checking health
}
```

## Terraform: Lifecycle Hook

```hcl
resource "aws_autoscaling_lifecycle_hook" "deployment_warmup_hook" {
  name                   = "app-deployment-warmup"
  autoscaling_group_name = aws_autoscaling_group.app_asg.name

  # Trigger when launching new servers
  lifecycle_transition   = "autoscaling:EC2_INSTANCE_LAUNCHING"

  # Wait 10 minutes (600 seconds) for bootstrap scripts to finish
  heartbeat_timeout      = 600

  # If the hook times out or fails, destroy the instance and try again
  default_result         = "ABANDON"

  # Optional: metadata payload passed directly to your target Lambda function
  notification_metadata = jsonencode({
    env  = "production"
    role = "api-worker"
  })
}
```

---

## AWS CLI: Auto Scaling Group

```bash
# Create an ASG from an existing launch template
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name production-web-asg \
  --launch-template "LaunchTemplateName=web-app-template,Version=\$Latest" \
  --min-size 2 --max-size 5 --desired-capacity 2 \
  --vpc-zone-identifier "subnet-aaaa,subnet-bbbb" \
  --target-group-arns arn:aws:elasticloadbalancing:region:acct:targetgroup/tg-web-production/abc123 \
  --health-check-type ELB \
  --health-check-grace-period 300

# Attach a target-tracking scaling policy (average CPU at 60%)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name production-web-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ASGAverageCPUUtilization"},
    "TargetValue": 60.0
  }'

# Manually change desired capacity
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name production-web-asg \
  --desired-capacity 4 \
  --honor-cooldown

# View current instances & lifecycle state
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names production-web-asg

# View recent scaling activity (proof of self-healing / scale-out events)
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name production-web-asg \
  --max-items 10

# Terminate a specific instance and let the ASG decide whether to replace it
aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-id i-0123456789abcdef0 \
  --should-decrement-desired-capacity
```

## AWS CLI: Lifecycle Hooks

```bash
# Create a scale-out lifecycle hook
aws autoscaling put-lifecycle-hook \
  --lifecycle-hook-name app-deployment-warmup \
  --auto-scaling-group-name production-web-asg \
  --lifecycle-transition autoscaling:EC2_INSTANCE_LAUNCHING \
  --heartbeat-timeout 600 \
  --default-result ABANDON

# Reset the countdown for a long-running setup task
aws autoscaling record-lifecycle-action-heartbeat \
  --lifecycle-hook-name app-deployment-warmup \
  --auto-scaling-group-name production-web-asg \
  --instance-id i-0123456789abcdef0 \
  --lifecycle-action-token <token-from-notification>

# Tell AWS the hook finished successfully
aws autoscaling complete-lifecycle-action \
  --lifecycle-hook-name app-deployment-warmup \
  --auto-scaling-group-name production-web-asg \
  --instance-id i-0123456789abcdef0 \
  --lifecycle-action-token <token-from-notification> \
  --lifecycle-action-result CONTINUE
```

## AWS CLI: Instance Refresh

```bash
# Roll the fleet onto a new Launch Template version, 20% at a time
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name production-web-asg \
  --preferences '{"MinHealthyPercentage": 80, "InstanceWarmup": 300}'

# Check refresh progress
aws autoscaling describe-instance-refreshes \
  --auto-scaling-group-name production-web-asg

# Cancel a refresh in progress
aws autoscaling cancel-instance-refresh \
  --auto-scaling-group-name production-web-asg
```

## AWS CLI: Suspend/Resume Processes

```bash
# Suspend health-check replacement (e.g., to SSH into a "degraded" node)
aws autoscaling suspend-processes \
  --auto-scaling-group-name production-web-asg \
  --scaling-processes HealthCheck

# Suspend AZ rebalancing during a known localized network issue
aws autoscaling suspend-processes \
  --auto-scaling-group-name production-web-asg \
  --scaling-processes AZRebalance

# Resume everything
aws autoscaling resume-processes \
  --auto-scaling-group-name production-web-asg
```

## AWS CLI: Target Groups & ELB

```bash
# Create a target group with health check thresholds
aws elbv2 create-target-group \
  --name tg-web-production \
  --protocol HTTP --port 80 \
  --vpc-id vpc-0123456789abcdef0 \
  --health-check-path / \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 \
  --health-check-timeout-seconds 5 \
  --health-check-interval-seconds 30

# Enable stickiness on a target group (duration-based, AWSALB cookie)
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:region:acct:targetgroup/tg-web-production/abc123 \
  --attributes Key=stickiness.enabled,Value=true \
               Key=stickiness.type,Value=lb_cookie \
               Key=stickiness.lb_cookie.duration_seconds,Value=86400

# Create the ALB itself
aws elbv2 create-load-balancer \
  --name alb-public-web \
  --subnets subnet-public-a subnet-public-b \
  --security-groups sg-0123456789abcdef0 \
  --scheme internet-facing --type application

# Create a listener forwarding to the target group
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:region:acct:loadbalancer/app/alb-public-web/xyz \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:region:acct:targetgroup/tg-web-production/abc123

# Check target health right now
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:region:acct:targetgroup/tg-web-production/abc123
```

---

## User Data Bootstrap Scripts

**Basic Apache install with instance ID (from the ASG demo):**
```bash
#!/bin/bash
sudo dnf update -y
sudo dnf install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
# Grabs the local instance ID using IMDSv2
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/instance-id)
echo "<h1>Hello from ASG Instance: $INSTANCE_ID</h1>" > /var/www/html/index.html
```

---

## curl: Sticky Session Proof

```bash
# Step 1: first request — save the AWSALB cookie the ALB issues
curl -c cookies.txt http://my-alb-123456789.us-east-1.elb.amazonaws.com/

# Inspect the saved cookie (should show an AWSALB entry)
cat cookies.txt

# Step 2: replay the cookie on every subsequent request — should keep hitting the same backend
curl -b cookies.txt http://my-alb-123456789.us-east-1.elb.amazonaws.com/
curl -b cookies.txt http://my-alb-123456789.us-east-1.elb.amazonaws.com/
curl -b cookies.txt http://my-alb-123456789.us-east-1.elb.amazonaws.com/

# Compare: WITHOUT any cookie jar, plain round-robin behavior returns
curl http://my-alb-123456789.us-east-1.elb.amazonaws.com/
curl http://my-alb-123456789.us-east-1.elb.amazonaws.com/
curl http://my-alb-123456789.us-east-1.elb.amazonaws.com/
```

---

## Load Testing CPU (for scale-out demo)

```bash
# SSH into a running ASG instance, then run a lightweight CPU-pegging loop
sha1sum /dev/zero &

# Stop it afterward
kill %1
```
