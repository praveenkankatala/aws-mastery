# AWS CloudFormation — Hands-On Labs (Zero → Production)

Ten progressive labs. Each one states **What** you build, **Why** it matters, and **How** to do it step by step. Do them in order — later labs reuse earlier concepts.

> 💰 **Cost note:** everything uses free-tier-eligible resources where possible (t2.micro/t3.micro, S3, VPC). **Always run the cleanup step** at the end of each lab.

---

## Lab 0 — Environment Setup

**What:** working AWS CLI + linting tools.
**Why:** every later lab depends on it.

```bash
aws --version                      # AWS CLI v2 required
aws configure                      # keys + default region + json output
aws sts get-caller-identity        # verify identity
pip install cfn-lint               # template linter
mkdir cfn-labs && cd cfn-labs
```

✅ **Success check:** `aws sts get-caller-identity` returns your account ID.

---

## Lab 1 — Your First Stack (S3 Bucket)

**What:** deploy a single versioned, encrypted S3 bucket.
**Why:** learn the template skeleton, create/describe/delete cycle, and stack events.

**Step 1 —** create `lab1-s3.yaml`:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Lab 1 - first stack, one S3 bucket

Resources:
  DemoBucket:
    Type: AWS::S3::Bucket
    Properties:
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

Outputs:
  BucketName:
    Description: Generated bucket name
    Value: !Ref DemoBucket
  BucketArn:
    Value: !GetAtt DemoBucket.Arn
```

**Step 2 —** lint, validate, deploy:

```bash
cfn-lint lab1-s3.yaml
aws cloudformation validate-template --template-body file://lab1-s3.yaml
aws cloudformation create-stack --stack-name lab1 --template-body file://lab1-s3.yaml
aws cloudformation wait stack-create-complete --stack-name lab1
```

**Step 3 —** inspect:

```bash
aws cloudformation describe-stacks --stack-name lab1 --query "Stacks[0].Outputs" --output table
aws cloudformation describe-stack-events --stack-name lab1 --output table
```

Notice: you never named the bucket — CloudFormation generated a **physical ID** from the stack name + logical ID + random suffix. This avoids name collisions across environments.

**Cleanup:**

```bash
aws cloudformation delete-stack --stack-name lab1
aws cloudformation wait stack-delete-complete --stack-name lab1
```

---

## Lab 2 — Parameters, Mappings, Conditions & Outputs (EC2 Web Server)

**What:** a parameterized EC2 web server with a security group; instance size varies by environment.
**Why:** one template, many environments — the core value of IaC.

**Step 1 —** create `lab2-ec2.yaml`:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Lab 2 - parameterized EC2 web server

Parameters:
  EnvType:
    Type: String
    Default: dev
    AllowedValues: [dev, prod]
  LatestAmi:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
  MyIp:
    Type: String
    Description: Your public IP in CIDR form, e.g. 203.0.113.10/32
    AllowedPattern: ^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}/\d{1,2}$

Mappings:
  EnvMap:
    dev:  { InstanceType: t3.micro }
    prod: { InstanceType: t3.small }

Conditions:
  IsProd: !Equals [!Ref EnvType, prod]

Resources:
  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP from anywhere, SSH from my IP
      SecurityGroupIngress:
        - { IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: 0.0.0.0/0 }
        - { IpProtocol: tcp, FromPort: 22, ToPort: 22, CidrIp: !Ref MyIp }

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmi
      InstanceType: !FindInMap [EnvMap, !Ref EnvType, InstanceType]
      SecurityGroupIds: [!GetAtt WebSG.GroupId]
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web"
      UserData:
        Fn::Base64: |
          #!/bin/bash -xe
          dnf install -y httpd
          echo "<h1>Hello from CloudFormation ($(hostname))</h1>" > /var/www/html/index.html
          systemctl enable --now httpd

Outputs:
  PublicURL:
    Value: !Sub "http://${WebServer.PublicDnsName}"
  InstanceId:
    Value: !Ref WebServer
```

**Step 2 —** deploy dev, then prod, from the same file:

```bash
aws cloudformation create-stack --stack-name lab2-dev \
  --template-body file://lab2-ec2.yaml \
  --parameters ParameterKey=EnvType,ParameterValue=dev \
               ParameterKey=MyIp,ParameterValue=$(curl -s ifconfig.me)/32
aws cloudformation wait stack-create-complete --stack-name lab2-dev
```

**Step 3 —** open the `PublicURL` output in a browser. Note the SSM parameter type resolved the **latest Amazon Linux AMI automatically** — no hardcoded, region-specific AMI IDs.

**Cleanup:** `aws cloudformation delete-stack --stack-name lab2-dev`

---

## Lab 3 — Safe Updates with Change Sets

**What:** modify Lab 2's stack via a change set and observe the difference between in-place update and replacement.
**Why:** change sets are how you avoid accidentally destroying production resources.

**Step 1 —** redeploy `lab2-dev` if deleted. Then edit the template: change the instance `Tags` value to `"${AWS::StackName}-web-v2"` **and** change `InstanceType` mapping for dev to `t3.small`.

**Step 2 —** create and review the change set:

```bash
aws cloudformation create-change-set --stack-name lab2-dev \
  --change-set-name resize \
  --template-body file://lab2-ec2.yaml \
  --parameters ParameterKey=EnvType,UsePreviousValue=true \
               ParameterKey=MyIp,UsePreviousValue=true

aws cloudformation describe-change-set --stack-name lab2-dev --change-set-name resize \
  --query "Changes[].ResourceChange.{R:LogicalResourceId,Action:Action,Replace:Replacement}" \
  --output table
```

You'll see `Modify` with `Replace: Conditional/False` — instance type change is *some interruption* (stop/start), not replacement. Now try changing the security group's `GroupDescription`: that property is **immutable → Replacement: True**. This is exactly the surprise change sets exist to catch.

**Step 3 —** execute or discard:

```bash
aws cloudformation execute-change-set --stack-name lab2-dev --change-set-name resize
aws cloudformation wait stack-update-complete --stack-name lab2-dev
```

**Cleanup:** delete the stack.

---

## Lab 4 — Cross-Stack References (Network + App)

**What:** a network stack that **exports** VPC/subnet IDs, and an app stack that **imports** them.
**Why:** layering — network teams own the network stack; app teams consume it. Independent lifecycles.

**Step 1 —** `lab4-network.yaml`:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Lab 4 - network layer with exports

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.10.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags: [{ Key: Name, Value: lab4-vpc }]

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.10.1.0/24
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select [0, !GetAZs ""]

  IGW:
    Type: AWS::EC2::InternetGateway
  AttachIGW:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties: { VpcId: !Ref VPC, InternetGatewayId: !Ref IGW }

  PublicRT:
    Type: AWS::EC2::RouteTable
    Properties: { VpcId: !Ref VPC }
  DefaultRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachIGW                   # explicit dependency: route needs attached IGW
    Properties:
      RouteTableId: !Ref PublicRT
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref IGW
  SubnetRTAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: { SubnetId: !Ref PublicSubnet, RouteTableId: !Ref PublicRT }

Outputs:
  VpcId:
    Value: !Ref VPC
    Export: { Name: !Sub "${AWS::StackName}-VpcId" }
  PublicSubnetId:
    Value: !Ref PublicSubnet
    Export: { Name: !Sub "${AWS::StackName}-PublicSubnetId" }
```

**Step 2 —** `lab4-app.yaml` (imports the exports):

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Lab 4 - app layer importing network exports

Parameters:
  NetworkStackName: { Type: String, Default: lab4-network }
  LatestAmi:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

Resources:
  AppSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: HTTP only
      VpcId:
        Fn::ImportValue: !Sub "${NetworkStackName}-VpcId"
      SecurityGroupIngress:
        - { IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: 0.0.0.0/0 }

  AppServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmi
      InstanceType: t3.micro
      SubnetId:
        Fn::ImportValue: !Sub "${NetworkStackName}-PublicSubnetId"
      SecurityGroupIds: [!GetAtt AppSG.GroupId]
```

**Step 3 —** deploy in order, then prove the protection:

```bash
aws cloudformation create-stack --stack-name lab4-network --template-body file://lab4-network.yaml
aws cloudformation wait stack-create-complete --stack-name lab4-network
aws cloudformation create-stack --stack-name lab4-app --template-body file://lab4-app.yaml
aws cloudformation wait stack-create-complete --stack-name lab4-app

# Try to delete the network stack while imported → it FAILS by design
aws cloudformation delete-stack --stack-name lab4-network
aws cloudformation describe-stack-events --stack-name lab4-network --max-items 3
```

**Cleanup (correct order):** delete `lab4-app` first, then `lab4-network`.

---

## Lab 5 — Nested Stacks

**What:** a root stack that instantiates the Lab 4 network template as a child and wires its outputs into an app child.
**Why:** single deployable unit built from reusable components.

**Step 1 —** upload children to S3:

```bash
BUCKET=cfn-labs-$(aws sts get-caller-identity --query Account --output text)
aws s3 mb s3://$BUCKET
aws s3 cp lab4-network.yaml s3://$BUCKET/
aws s3 cp lab4-app-nested.yaml s3://$BUCKET/     # copy of lab4-app with VpcId/SubnetId as plain Parameters instead of ImportValue
```

**Step 2 —** `lab5-root.yaml`:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Lab 5 - root stack composing nested stacks

Parameters:
  ArtifactBucket: { Type: String }

Resources:
  Network:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub "https://${ArtifactBucket}.s3.amazonaws.com/lab4-network.yaml"

  App:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub "https://${ArtifactBucket}.s3.amazonaws.com/lab4-app-nested.yaml"
      Parameters:
        VpcId: !GetAtt Network.Outputs.VpcId
        SubnetId: !GetAtt Network.Outputs.PublicSubnetId
```

**Step 3 —** deploy the root (needs `CAPABILITY_AUTO_EXPAND`? No — only for macros; nested stacks just work):

```bash
aws cloudformation create-stack --stack-name lab5-root \
  --template-body file://lab5-root.yaml \
  --parameters ParameterKey=ArtifactBucket,ParameterValue=$BUCKET
```

Observe in the console: three stacks appear — the root plus two children marked **NESTED**.

> Alternative: skip manual S3 uploads with `aws cloudformation package`, which rewrites local `TemplateURL` paths for you.

**Cleanup:** delete **only** `lab5-root` — children cascade automatically. Then `aws s3 rb s3://$BUCKET --force`.

---

## Lab 6 — cfn-init + cfn-signal + CreationPolicy (Proper Bootstrapping)

**What:** an EC2 instance that installs and configures Apache via metadata, and only reports `CREATE_COMPLETE` after the app is actually running.
**Why:** without signaling, CloudFormation says "complete" the moment the VM boots — even if your app failed to install.

**Step 1 —** `lab6-bootstrap.yaml` (key parts):

```yaml
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    CreationPolicy:
      ResourceSignal: { Count: 1, Timeout: PT10M }
    Metadata:
      AWS::CloudFormation::Init:
        configSets:
          default: [install, configure]
        install:
          packages:
            dnf: { httpd: [] }
        configure:
          files:
            /var/www/html/index.html:
              content: "<h1>Bootstrapped by cfn-init</h1>"
              mode: "000644"
          services:
            systemd:
              httpd: { enabled: true, ensureRunning: true }
    Properties:
      ImageId: !Ref LatestAmi
      InstanceType: t3.micro
      SecurityGroupIds: [!GetAtt WebSG.GroupId]
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} \
            --resource WebServer --configsets default --region ${AWS::Region}
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} \
            --resource WebServer --region ${AWS::Region}
```

(Add the same `LatestAmi` parameter and `WebSG` security group as Lab 2.)

**Step 2 —** deploy and watch: the stack stays `CREATE_IN_PROGRESS` until the signal arrives (`Received SUCCESS signal` appears in events).

**Step 3 —** break it on purpose: change the package name to `httpdd` (typo), update the stack, and watch cfn-signal report failure → automatic rollback. Then SSH in and read `/var/log/cfn-init.log`. **This is the debugging muscle you'll use in real incidents.**

**Cleanup:** delete the stack.

---

## Lab 7 — Drift Detection

**What:** deliberately change a resource in the console, then catch it.
**Why:** drift is the silent killer of IaC trust.

```bash
# 1. Deploy lab2-dev again, then manually add an inbound rule (port 8080) to WebSG in the console.

# 2. Detect drift
DRIFT_ID=$(aws cloudformation detect-stack-drift --stack-name lab2-dev \
  --query StackDriftDetectionId --output text)
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id $DRIFT_ID

# 3. See exactly what changed
aws cloudformation describe-stack-resource-drifts --stack-name lab2-dev \
  --stack-resource-drift-status-filters MODIFIED \
  --query "StackResourceDrifts[].{R:LogicalResourceId,Diff:PropertyDifferences}"
```

**Remediate** either by (a) removing the console change, or (b) adding the rule to the template and updating — then re-run drift detection to confirm `IN_SYNC`.

**Cleanup:** delete the stack.

---

## Lab 8 — Protection Mechanisms (Policies, Termination Protection, DeletionPolicy)

**What:** make a stack production-hardened.
**Why:** the difference between a lab template and something you'd let a team touch.

```bash
# 1. Deploy Lab 1's bucket stack again, adding to the template:
#      DeletionPolicy: Retain
#      UpdateReplacePolicy: Retain
#    on the bucket resource.

# 2. Enable termination protection
aws cloudformation update-termination-protection \
  --stack-name lab8 --enable-termination-protection

# 3. Try to delete → blocked
aws cloudformation delete-stack --stack-name lab8   # ValidationError

# 4. Apply a stack policy denying updates to the bucket
aws cloudformation set-stack-policy --stack-name lab8 --stack-policy-body '{
  "Statement":[
    {"Effect":"Allow","Action":"Update:*","Principal":"*","Resource":"*"},
    {"Effect":"Deny","Action":"Update:*","Principal":"*","Resource":"LogicalResourceId/DemoBucket"}
  ]}'

# 5. Attempt an update that touches the bucket → denied by policy.

# 6. Cleanup: disable protection, delete the stack — the bucket SURVIVES
#    (DeletionPolicy: Retain). Delete it manually:
aws cloudformation update-termination-protection --stack-name lab8 --no-enable-termination-protection
aws cloudformation delete-stack --stack-name lab8
aws s3 rb s3://<retained-bucket-name> --force
```

---

## Lab 9 — Capstone: 3-Tier Architecture (VPC + ALB + Auto Scaling)

**What:** a production-shaped deployment — VPC across 2 AZs, Application Load Balancer, Auto Scaling group of web servers with rolling updates and health-signaled instances.
**Why:** combines every concept: parameters, mappings, functions, dependencies, creation/update policies, outputs.

**Architecture:**

```
            Internet
               │
        ┌──────▼──────┐
        │     ALB     │  (public subnets, 2 AZs)
        └──────┬──────┘
        ┌──────▼──────────────┐
        │  Target Group :80   │
        └──────┬──────────────┘
     ┌─────────┴─────────┐
┌────▼────┐         ┌────▼────┐
│  EC2    │   ...   │  EC2    │   Auto Scaling Group (min 2, max 4)
│ (AZ-a)  │         │ (AZ-b)  │   Launch Template + cfn-init
└─────────┘         └─────────┘
```

**Key template fragments** (`lab9-capstone.yaml` — assemble with the VPC pieces from Lab 4 duplicated across two AZs):

```yaml
  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets: [!Ref PublicSubnetA, !Ref PublicSubnetB]
      SecurityGroups: [!GetAtt AlbSG.GroupId]

  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      VpcId: !Ref VPC
      Port: 80
      Protocol: HTTP
      HealthCheckPath: /
      TargetType: instance

  Listener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP
      DefaultActions: [{ Type: forward, TargetGroupArn: !Ref TargetGroup }]

  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: !Ref LatestAmi
        InstanceType: t3.micro
        SecurityGroupIds: [!GetAtt WebSG.GroupId]
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash -xe
            dnf install -y httpd
            echo "<h1>$(hostname) via CloudFormation</h1>" > /var/www/html/index.html
            systemctl enable --now httpd
            /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} \
              --resource WebASG --region ${AWS::Region}

  WebASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    CreationPolicy:
      ResourceSignal: { Count: 2, Timeout: PT15M }
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 1
        MaxBatchSize: 1
        PauseTime: PT10M
        WaitOnResourceSignals: true
    Properties:
      VPCZoneIdentifier: [!Ref PublicSubnetA, !Ref PublicSubnetB]
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      MinSize: "2"
      MaxSize: "4"
      TargetGroupARNs: [!Ref TargetGroup]

Outputs:
  SiteURL:
    Value: !Sub "http://${ALB.DNSName}"
```

**Deploy and verify:**

```bash
aws cloudformation deploy --stack-name lab9 --template-file lab9-capstone.yaml
curl $(aws cloudformation describe-stacks --stack-name lab9 \
  --query "Stacks[0].Outputs[?OutputKey=='SiteURL'].OutputValue" --output text)
```

Refresh the URL a few times — the hostname changes as the ALB round-robins across AZs.

**Exercise the UpdatePolicy:** change the index.html text in UserData and `deploy` again — watch instances replaced **one at a time** with signals, zero downtime.

**Cleanup:** delete the stack (everything cascades — this is the payoff of IaC).

---

## Lab 10 — StackSets (Optional: needs 2+ accounts or an AWS Organization)

**What:** push a compliance baseline (e.g., an S3 access-logging bucket + IAM role) to multiple accounts/regions.
**Why:** enterprise governance at scale.

**How (self-managed model):**

1. In the **admin** account create `AWSCloudFormationStackSetAdministrationRole` (AWS provides a template).
2. In every **target** account create `AWSCloudFormationStackSetExecutionRole` trusting the admin account.
3. Then:

```bash
aws cloudformation create-stack-set --stack-set-name baseline \
  --template-body file://baseline.yaml --capabilities CAPABILITY_NAMED_IAM

aws cloudformation create-stack-instances --stack-set-name baseline \
  --accounts 111111111111 222222222222 \
  --regions ap-south-1 \
  --operation-preferences FailureToleranceCount=0,MaxConcurrentCount=1

aws cloudformation list-stack-instances --stack-set-name baseline
```

**Cleanup:** `delete-stack-instances` (with `--no-retain-stacks`), then `delete-stack-set`.

---

## 🎓 What You Can Now Claim

- Author templates using every section, intrinsic function, and pseudo parameter
- Deploy, update (safely, via change sets), and destroy stacks from the CLI
- Design layered architectures with cross-stack exports and nested stacks
- Bootstrap and health-signal EC2 with cfn-init/cfn-signal + Creation/Update policies
- Detect and remediate drift; harden stacks with policies and protection
- Roll out multi-account baselines with StackSets

Next steps: rebuild Lab 9 in **AWS SAM** or **CDK**, wire `aws cloudformation deploy` into a GitHub Actions pipeline, and add **cfn-guard** rules to the pipeline.
