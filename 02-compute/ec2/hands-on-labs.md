# EC2 — Hands-On Labs

Guided, step-by-step demos you can run in your own AWS account (Free Tier eligible instance types recommended where possible). Each lab builds practical muscle memory for a concept in `README.md`.

## Table of Contents
- [Lab 1: Bake a Golden AMI from a Live Instance](#lab-1-bake-a-golden-ami-from-a-live-instance)
- [Lab 2: Decommission an AMI Without Leaving Orphaned Costs](#lab-2-decommission-an-ami-without-leaving-orphaned-costs)
- [Lab 3: Bootstrap a Web Server with User Data](#lab-3-bootstrap-a-web-server-with-user-data)
- [Lab 4: Deploy the Same Bootstrap via CloudFormation](#lab-4-deploy-the-same-bootstrap-via-cloudformation)
- [Lab 5: Query and Lock Down Instance Metadata](#lab-5-query-and-lock-down-instance-metadata)
- [Lab 6: Force a Status-Check Recovery](#lab-6-force-a-status-check-recovery)
- [Lab 7: Launch Instances Into a Placement Group](#lab-7-launch-instances-into-a-placement-group)

---

## Lab 1: Bake a Golden AMI from a Live Instance

**Goal:** capture a configured server as a reusable blueprint.

1. Launch a base Ubuntu EC2 instance and SSH in.
2. Install and configure something so there's a visible change to capture:
   ```bash
   sudo apt update && sudo apt install -y apache2
   echo "Welcome to my Golden Image App!" | sudo tee /var/www/html/index.html
   ```
3. In the AWS Console: **Instances → select instance → Actions → Image and templates → Create image**.
4. Configure:
   - **Image name**: `Webserver-Golden-Image-v1`
   - **No reboot**: leave **unchecked** (default) so AWS safely flushes the filesystem before snapshotting. Only check this if you cannot tolerate any downtime and are certain there's no in-flight disk activity.
   - Check **"Tag images and snapshots together"** and add `Environment: Production`.
5. Click **Create image**. Watch **EC2 → AMIs** — status moves `pending → available`.
6. Once available, select the AMI and click **Launch instance from AMI** to prove it spins up an identical, pre-configured server instantly.

---

## Lab 2: Decommission an AMI Without Leaving Orphaned Costs

**Goal:** clean up an obsolete AMI the *correct* way (two steps, not one).

1. **EC2 → AMIs** → select the obsolete AMI (`Webserver-Golden-Image-v1`).
2. **Actions → Deregister AMI** → confirm.
   - This immediately blocks new launches from it, but **does not stop storage billing**.
3. **EC2 → Elastic Block Store → Snapshots** → find the snapshot described as `Created by CreateImage(...) for ami-...`.
4. Select it → **Actions → Delete snapshot** → confirm.

**Why both steps matter:** deregistering only removes the *pointer*; the actual EBS snapshot data in S3 keeps costing money until you delete it explicitly.

> For fleets managed by Auto Scaling Groups, prefer **AMI Deprecation** (flag + future date) over hard deregistration, so in-flight scaling actions don't break while you migrate to the new image.

---

## Lab 3: Bootstrap a Web Server with User Data

**Goal:** launch a fully-configured web server with zero manual SSH steps.

1. **Launch Instance** → choose a base Amazon Linux/Ubuntu AMI.
2. Expand **Advanced details → User data**, and paste:
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y httpd
   systemctl start httpd
   systemctl enable httpd
   echo "Hello from User Data bootstrap!" > /var/www/html/index.html
   ```
3. Launch, wait for the instance to reach `running`, then hit its public IP in a browser — the page should already be live.
4. SSH in and confirm the script actually ran as expected:
   ```bash
   cat /var/log/cloud-init-output.log
   ```
5. **Test the one-shot rule:** stop and start the instance, then edit `/var/www/html/index.html` manually. Reboot the instance (not stop/start) and confirm your manual edit survives — proving User Data does *not* re-run on every boot.

---

## Lab 4: Deploy the Same Bootstrap via CloudFormation

**Goal:** turn Lab 3 into a repeatable, parameterized template.

1. Save this as `webserver.yaml`:
   ```yaml
   Parameters:
     EnvironmentName:
       Type: String
       Default: Production

   Resources:
     MyWebServerInstance:
       Type: AWS::EC2::Instance
       Properties:
         InstanceType: t4g.small
         ImageId: ami-0c55b159cbfafe1f0   # replace with a valid AMI for your region
         SecurityGroupIds:
           - !Ref WebServerSecurityGroup
         UserData:
           Fn::Base64:
             Fn::Sub: |
               #!/bin/bash
               yum update -y
               yum install -y httpd
               systemctl start httpd
               systemctl enable httpd
               echo "<h1>Welcome to the ${EnvironmentName} Web Server!</h1>" >> /var/www/html/index.html

     WebServerSecurityGroup:
       Type: AWS::EC2::SecurityGroup
       Properties:
         GroupDescription: Allow HTTP traffic
         SecurityGroupIngress:
           - IpProtocol: tcp
             FromPort: 80
             ToPort: 80
             CidrIp: 0.0.0.0/0
   ```
2. Deploy it:
   ```bash
   aws cloudformation create-stack \
     --stack-name ec2-userdata-demo \
     --template-body file://webserver.yaml \
     --parameters ParameterKey=EnvironmentName,ParameterValue=Staging
   ```
3. Once `CREATE_COMPLETE`, grab the instance's public IP and confirm the page shows **"Welcome to the Staging Web Server!"** — proving `Fn::Sub` correctly injected the parameter.

---

## Lab 5: Query and Lock Down Instance Metadata

**Goal:** see IMDSv1 vs IMDSv2 in action, then enforce the secure mode.

1. SSH into any running instance.
2. Try the legacy, unauthenticated call:
   ```bash
   curl http://169.254.169.254/latest/meta-data/instance-id
   ```
3. Now do it the IMDSv2 way:
   ```bash
   TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
     -H "X-aws-ec2-metadata-token-ttl-seconds: 6")
   curl -H "X-aws-ec2-metadata-token: $TOKEN" \
     http://169.254.169.254/latest/meta-data/instance-id
   ```
4. Enforce IMDSv2-only from your local machine (not the instance):
   ```bash
   aws ec2 modify-instance-metadata-options \
     --instance-id i-0123456789abcdef0 \
     --http-tokens required \
     --http-put-response-hop-limit 1
   ```
5. Re-run step 2 — it should now fail, proving IMDSv1 access is blocked.

---

## Lab 6: Force a Status-Check Recovery

**Goal:** understand the difference between a reboot and a stop/start at the hardware level.

1. Note your running instance's **public/private IP** and, if applicable, any **Instance Store** mount contents.
2. **Reboot** the instance (`sudo reboot` from inside, or via console) — confirm the IP and any instance-store data are unchanged (same physical host).
3. **Stop**, then **Start** the instance from the console.
4. Compare: the **public IP typically changes** (unless using an Elastic IP), and any **Instance Store data is wiped**, because AWS moved you to a new physical host.

---

## Lab 7: Launch Instances Into a Placement Group

**Goal:** see placement strategy affect your launch behavior.

```bash
# Spread group — max isolation, capped at 7 per AZ
aws ec2 create-placement-group --group-name demo-spread --strategy spread

aws ec2 run-instances \
  --image-id ami-0123456789example \
  --instance-type t3.micro \
  --count 3 \
  --placement "GroupName=demo-spread"
```

Try launching an 8th instance into the same Spread group in one AZ and observe the capacity error — this demonstrates the hard 7-instances-per-AZ limit in practice.
