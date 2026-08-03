# AWS Elastic Beanstalk — Hands-On Labs

> Learning by reading only gets you halfway. These labs take you from an empty folder to a production-grade, database-backed, HTTPS-secured, auto-scaling application — for real, on real AWS infrastructure.

📎 Related: [README.md](./README.md) · [commands-cheatsheet.md](./commands-cheatsheet.md) · [troubleshooting.md](./troubleshooting.md)

⚠️ **Cost note:** These labs use real AWS resources. Most fit comfortably within the AWS Free Tier if you're within your first 12 months and clean up after each lab (see the cleanup steps!). If you're outside the Free Tier, expect a small cost (typically a few dollars if you complete everything and clean up promptly). **Lab 8 shows you exactly how to tear everything down.**

---

## 🧭 Lab Roadmap

| Lab | Title | What You'll Learn |
|---|---|---|
| [Lab 0](#lab-0-environment-setup) | Environment Setup | Installing tools, configuring credentials, IAM basics |
| [Lab 1](#lab-1-your-first-deployment-hello-world) | Your First Deployment | `eb init`, `eb create`, `eb deploy`, `eb open` |
| [Lab 2](#lab-2-configuration-as-code-with-ebextensions) | Configuration as Code | `.ebextensions`, environment variables |
| [Lab 3](#lab-3-zero-downtime-deployments-in-action) | Zero-Downtime Deployments | Deployment policies, rolling updates, immutable deploys |
| [Lab 4](#lab-4-connecting-a-database-rds) | Connecting a Database | Decoupled RDS setup and connection |
| [Lab 5](#lab-5-background-jobs-with-the-worker-tier) | Background Jobs | Worker Tier + SQS |
| [Lab 6](#lab-6-custom-domain--https) | Custom Domain & HTTPS | Route 53 + ACM |
| [Lab 7](#lab-7-cicd-with-github-actions) | CI/CD Automation | GitHub Actions auto-deploy on push |
| [Lab 8](#lab-8-cleanup--cost-control) | Cleanup & Cost Control | Safely tearing everything down |

---

## Lab 0: Environment Setup

### Goal
Get every tool installed and your AWS account ready to deploy.

### Step 1 — Create an IAM User (never use root!)

1. Log into the [AWS Console](https://console.aws.amazon.com) with your root account (one-time only)
2. Go to **IAM → Users → Create user**
3. Name it something like `eb-learner`
4. Attach the policy **`AdministratorAccess-AWSElasticBeanstalk`** (purpose-built for EB learning; tighten later for real production use)
5. Create an **access key** for this user (choose "Command Line Interface (CLI)" as the use case)
6. **Save the Access Key ID and Secret Access Key somewhere safe** — the secret is shown only once

### Step 2 — Install the AWS CLI

```bash
# macOS
brew install awscli

# Windows (PowerShell, run as admin)
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify:
```bash
aws --version
# Expect: aws-cli/2.x.x ...
```

### Step 3 — Configure Credentials

```bash
aws configure
```
You'll be prompted for:
```
AWS Access Key ID [None]: <paste your access key>
AWS Secret Access Key [None]: <paste your secret key>
Default region name [None]: us-east-1
Default output format [None]: json
```

### Step 4 — Install the EB CLI

```bash
pip install awsebcli --upgrade --user
```

Verify:
```bash
eb --version
# Expect: EB CLI 3.x.x ...
```

> 🩹 **If `eb` isn't found after install:** your Python user scripts folder isn't on your PATH. See [troubleshooting.md](./troubleshooting.md#eb-command-not-found) for the fix.

### ✅ Checkpoint
Run `aws sts get-caller-identity` — you should see your account ID and the `eb-learner` user ARN printed back. If that works, you're ready.

---

## Lab 1: Your First Deployment (Hello World)

### Goal
Deploy a real, working Flask app to Elastic Beanstalk from a completely empty folder.

### Step 1 — Create the Project

```bash
mkdir eb-hello-world && cd eb-hello-world
git init
```

### Step 2 — Write the Application

Create `application.py`:
```python
from flask import Flask

application = Flask(__name__)

@application.route('/')
def home():
    return "Hello from Elastic Beanstalk! 🚀"

@application.route('/health')
def health():
    return {"status": "healthy"}, 200

if __name__ == '__main__':
    application.run(debug=True)
```

> 📝 **Naming note:** On the Python platform, EB looks for a variable named `application` by default (this is a WSGI convention). Naming your Flask instance `app` instead requires extra config — stick with `application` to keep things simple.

Create `requirements.txt`:
```
flask==3.0.3
```

### Step 3 — Commit to Git

EB CLI deploys your **last git commit**, so commit before deploying:
```bash
echo "venv/
__pycache__/
*.pyc
.elasticbeanstalk/*
!.elasticbeanstalk/*.cfg.yml
!.elasticbeanstalk/*.global.yml" > .gitignore

git add .
git commit -m "Initial Hello World app"
```

### Step 4 — Initialize Elastic Beanstalk

```bash
eb init -p python-3.12 eb-hello-world --region us-east-1
```

This creates a `.elasticbeanstalk/config.yml` file. If prompted about SSH, choose **yes** and let it create a new keypair — you'll want SSH access for later debugging.

### Step 5 — Create the Environment

```bash
eb create eb-hello-world-env --single --instance-type t3.micro
```

- `--single` skips the load balancer (cheaper, fine for learning)
- This step takes **3–5 minutes**. Behind the scenes, EB is: uploading your source bundle to S3, creating a CloudFormation stack, launching an EC2 instance, installing Python, installing your dependencies, and starting your app under a WSGI server.

You'll see live status output ending in something like:
```
Environment created successfully.
```

### Step 6 — Open It

```bash
eb open
```

Your browser should open to `http://eb-hello-world-env.xxxxxx.us-east-1.elasticbeanstalk.com/` showing **"Hello from Elastic Beanstalk! 🚀"**

### Step 7 — Make a Change and Redeploy

Edit `application.py`:
```python
@application.route('/')
def home():
    return "Hello from Elastic Beanstalk! Now updated. ✅"
```

```bash
git add .
git commit -m "Update homepage message"
eb deploy
```

Refresh your browser — the new message appears. **That's the entire core deployment loop** you'll use for the rest of your life with EB: edit → commit → `eb deploy`.

### ✅ Checkpoint
- `eb status` shows `Health: Green`
- `eb health` shows `Ok` status
- Your browser shows the updated message

---

## Lab 2: Configuration as Code with `.ebextensions`

### Goal
Stop clicking around the console — define your environment's configuration in version-controlled files.

### Step 1 — Add a Config Folder

In your project root:
```bash
mkdir .ebextensions
```

### Step 2 — Set Environment Variables Declaratively

Create `.ebextensions/01-environment.config`:
```yaml
option_settings:
  aws:elasticbeanstalk:application:environment:
    FLASK_ENV: production
    APP_VERSION: "1.0"
```

### Step 3 — Configure Instance & Scaling Settings

Create `.ebextensions/02-scaling.config`:
```yaml
option_settings:
  aws:autoscaling:asg:
    MinSize: 1
    MaxSize: 2
  aws:autoscaling:launchconfiguration:
    InstanceType: t3.micro
```

### Step 4 — Enable Enhanced Health Reporting & Log Streaming

Create `.ebextensions/03-monitoring.config`:
```yaml
option_settings:
  aws:elasticbeanstalk:healthreporting:system:
    SystemType: enhanced
  aws:elasticbeanstalk:cloudwatch:logs:
    StreamLogs: true
    DeleteOnTerminate: false
    RetentionInDays: 7
```

### Step 5 — Read the Variable in Your App

Update `application.py`:
```python
import os
from flask import Flask

application = Flask(__name__)

@application.route('/')
def home():
    env = os.environ.get('FLASK_ENV', 'unknown')
    version = os.environ.get('APP_VERSION', '0.0')
    return f"Hello from Elastic Beanstalk! Environment: {env}, Version: {version} 🚀"

if __name__ == '__main__':
    application.run(debug=True)
```

### Step 6 — Deploy

```bash
git add .
git commit -m "Add .ebextensions configuration"
eb deploy
```

### Step 7 — Verify

```bash
eb printenv
```
You should see `FLASK_ENV` and `APP_VERSION` listed. Visit the site — the message now shows both values, proving they came from EB's config, not hardcoded values.

### ✅ Checkpoint
Your environment's entire configuration now lives in Git. Delete the environment and recreate it (`eb create` again) — it comes back identically configured. **This is the whole point of Infrastructure as Code.**

---

## Lab 3: Zero-Downtime Deployments in Action

### Goal
See the difference between deployment policies with your own eyes.

### Step 1 — Simulate a Slow Startup

To actually *see* a rolling deployment in progress (instead of it finishing too fast to observe), temporarily add an artificial delay to `application.py`'s startup — or simply keep this lab conceptual and focus on configuration; either works.

### Step 2 — Switch to Rolling with Additional Batch

Create `.ebextensions/04-deployment-policy.config`:
```yaml
option_settings:
  aws:elasticbeanstalk:command:
    DeploymentPolicy: RollingWithAdditionalBatch
    BatchSizeType: Fixed
    BatchSize: 1
```

Scale up first so you have more than one instance to actually see the rollout batch:
```bash
eb scale 2
```

Deploy a small change and watch events in real time in a second terminal:
```bash
# Terminal 1
eb deploy

# Terminal 2 (run simultaneously)
eb events --follow
```

You'll see events like `Environment update is starting`, instances being taken in/out of rotation in batches, and finally `Environment update completed successfully`.

### Step 3 — Switch to Immutable Deployments

Edit `.ebextensions/04-deployment-policy.config`:
```yaml
option_settings:
  aws:elasticbeanstalk:command:
    DeploymentPolicy: Immutable
```

```bash
git add .
git commit -m "Switch to immutable deployments"
eb deploy
```

Watch `eb events --follow` again — notice EB now launches a **temporary second Auto Scaling Group**, waits for it to pass health checks, then swaps traffic over and tears down the old one. Check the console's EC2 dashboard during this deploy — you'll briefly see double the instances.

### Step 4 — Blue/Green Pattern (Manual)

For the biggest, riskiest changes, skip in-place updates entirely:

```bash
# Create a completely separate environment with the new version
eb create eb-hello-world-green --single --instance-type t3.micro

# Test it thoroughly on its own URL
eb open eb-hello-world-green

# When confident, swap CNAMEs — instant cutover, zero downtime
eb swap eb-hello-world-env --destination_name eb-hello-world-green

# Old environment is now idle and safe to remove
eb terminate eb-hello-world-env
```

### ✅ Checkpoint
You've now performed a rolling deploy, an immutable deploy, and a full blue/green swap. You understand the tradeoffs of each — see the [comparison table in README.md](./README.md#2-deployment-policies-how-new-code-rolls-out) to reinforce it.

---

## Lab 4: Connecting a Database (RDS)

### Goal
Attach a real PostgreSQL database — **decoupled** from the EB environment, the production-correct way.

### Step 1 — Create the RDS Instance (Outside of EB, on Purpose)

Via AWS CLI:
```bash
aws rds create-db-instance \
  --db-instance-identifier eb-learning-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username ebadmin \
  --master-user-password "ChangeMe123!" \
  --allocated-storage 20 \
  --publicly-accessible false
```

Wait for it to become available (a few minutes):
```bash
aws rds describe-db-instances --db-instance-identifier eb-learning-db \
  --query "DBInstances[0].DBInstanceStatus"
```

### Step 2 — Allow Your EB Instances to Reach It

Find your EB environment's Security Group and the RDS instance's Security Group in the console, then add an **inbound rule** on the RDS security group: allow port `5432` from the EB EC2 security group (not from `0.0.0.0/0` — scope it down!).

### Step 3 — Get the Connection Endpoint

```bash
aws rds describe-db-instances --db-instance-identifier eb-learning-db \
  --query "DBInstances[0].Endpoint.Address" --output text
```

### Step 4 — Set the Connection String as an Environment Variable

```bash
eb setenv DATABASE_URL="postgresql://ebadmin:ChangeMe123!@<your-endpoint>:5432/postgres"
```

### Step 5 — Use It in Your App

```python
import os
import psycopg2
from flask import Flask

application = Flask(__name__)

@application.route('/db-check')
def db_check():
    try:
        conn = psycopg2.connect(os.environ['DATABASE_URL'])
        conn.close()
        return {"database": "connected ✅"}
    except Exception as e:
        return {"database": "connection failed ❌", "error": str(e)}, 500
```

Add `psycopg2-binary==2.9.9` to `requirements.txt`, then:
```bash
git add .
git commit -m "Add RDS connectivity check"
eb deploy
```

Visit `/db-check` — you should see `{"database": "connected ✅"}`.

> 🧠 **Why decoupled?** Because this RDS instance was created *outside* EB, running `eb terminate` on your environment will **never** touch it. Your data survives environment rebuilds, blue/green swaps, and experiments. This is the production pattern — the "create RDS through the EB console" shortcut is fine for quick throwaway demos only.

### ✅ Checkpoint
`/db-check` returns success, and you understand *why* the database was created separately rather than through the EB environment wizard.

---

## Lab 5: Background Jobs with the Worker Tier

### Goal
Offload slow/async work to a dedicated Worker Tier environment backed by SQS.

### Step 1 — Create a Worker App

New folder `eb-worker-app`:
```python
# application.py
from flask import Flask, request

application = Flask(__name__)

@application.route('/', methods=['GET'])
def health():
    return "Worker is running", 200

@application.route('/', methods=['POST'])
def process_message():
    # SQS messages arrive here as an HTTP POST from the EB worker daemon
    message_body = request.data.decode('utf-8')
    print(f"Processing job: {message_body}")
    # ... do the actual background work here ...
    return "", 200
```

`requirements.txt`:
```
flask==3.0.3
```

### Step 2 — Initialize and Create the Worker Environment

```bash
cd eb-worker-app
git init && git add . && git commit -m "Initial worker app"
eb init -p python-3.12 eb-worker-app --region us-east-1
eb create eb-worker-env --tier worker
```

EB automatically creates an **SQS queue** and configures the `aws-sqsd` daemon on the worker instances to poll it and forward messages to your app as HTTP POSTs to `/`.

### Step 3 — Find the Queue and Send a Test Message

```bash
# Get the queue URL EB created
aws elasticbeanstalk describe-environment-resources \
  --environment-name eb-worker-env

# Send a test message
aws sqs send-message \
  --queue-url <queue-url-from-above> \
  --message-body '{"job": "send_welcome_email", "user_id": 42}'
```

### Step 4 — Confirm Processing

```bash
eb logs
```
Look for your `print(f"Processing job: ...")` output in the log bundle — proof the message was pulled off the queue and handled by your app.

### ✅ Checkpoint
You've decoupled a slow/async task (in real life: emails, video processing, report generation) from your user-facing web tier using a pattern that scales the two independently.

---

## Lab 6: Custom Domain & HTTPS

### Goal
Serve your app on a real domain over HTTPS.

> 📌 Requires a domain registered in Route 53 (or transferable DNS elsewhere). If you don't own a domain, read through this lab conceptually — the pattern is what matters.

### Step 1 — Request a Certificate in ACM

```bash
aws acm request-certificate \
  --domain-name app.yourdomain.com \
  --validation-method DNS \
  --region us-east-1
```

### Step 2 — Validate via DNS

Add the CNAME validation record ACM gives you to your Route 53 hosted zone (console → Certificate Manager → your cert → "Create records in Route 53" button does this automatically). Wait for status to become `ISSUED`.

### Step 3 — Attach the Certificate to Your EB Load Balancer

Create `.ebextensions/05-https.config`:
```yaml
option_settings:
  aws:elb:listener:443:
    ListenerProtocol: HTTPS
    SSLCertificateId: arn:aws:acm:us-east-1:<account-id>:certificate/<cert-id>
    InstancePort: 80
    InstanceProtocol: HTTP
```

> ⚠️ Requires a load-balanced environment (not `--single`). Recreate the environment without `--single` if needed, or add a load balancer via `eb config`.

### Step 4 — Point Your Domain at the Environment

In Route 53, create an **A record (Alias)** for `app.yourdomain.com` pointing to your EB environment's load balancer.

### Step 5 — Redirect HTTP → HTTPS (Best Practice)

Add an Nginx config override, `.platform/nginx/conf.d/https-redirect.conf`:
```nginx
server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

Deploy:
```bash
git add .
git commit -m "Add HTTPS support and HTTP redirect"
eb deploy
```

### ✅ Checkpoint
`https://app.yourdomain.com` loads your app with a valid padlock; plain `http://` redirects automatically.

---

## Lab 7: CI/CD with GitHub Actions

### Goal
`git push` to `main` → automatic deploy to Elastic Beanstalk, no manual `eb deploy` ever again.

### Step 1 — Create an IAM User for CI (Least Privilege)

Create a dedicated IAM user (e.g., `github-actions-deployer`) with a scoped policy limited to `elasticbeanstalk:*`, `s3:*` (for the app-versions bucket), and `cloudformation:*` — avoid reusing your personal admin credentials in CI.

### Step 2 — Add Secrets to GitHub

In your GitHub repo → **Settings → Secrets and variables → Actions**, add:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

### Step 3 — Create the Workflow File

`.github/workflows/deploy.yml`:
```yaml
name: Deploy to Elastic Beanstalk

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Generate deployment package
        run: zip -r deploy.zip . -x "*.git*"

      - name: Deploy to Elastic Beanstalk
        uses: einaregilsson/beanstalk-deploy@v22
        with:
          aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          application_name: eb-hello-world
          environment_name: eb-hello-world-env
          version_label: "gh-${{ github.sha }}"
          region: us-east-1
          deployment_package: deploy.zip
```

### Step 4 — Push and Watch

```bash
git add .
git commit -m "Add CI/CD pipeline"
git push origin main
```

Check the **Actions** tab on GitHub — you'll see the workflow run, and `eb status` will show a new application version deployed automatically.

### ✅ Checkpoint
Every push to `main` now deploys itself. You never need to run `eb deploy` from your laptop for this project again.

---

## Lab 8: Cleanup & Cost Control

### Goal
Tear down everything you built in these labs so you don't get an unexpected bill.

```bash
# Terminate each environment created during the labs
eb terminate eb-hello-world-env --force
eb terminate eb-hello-world-green --force
eb worker-env --force   # (use: eb terminate eb-worker-env --force)
eb terminate eb-worker-env --force

# Delete the decoupled RDS instance from Lab 4 (skip final snapshot only for learning resources!)
aws rds delete-db-instance \
  --db-instance-identifier eb-learning-db \
  --skip-final-snapshot

# Delete the applications themselves (removes stored versions in S3 too)
aws elasticbeanstalk delete-application \
  --application-name eb-hello-world --terminate-env-by-force
aws elasticbeanstalk delete-application \
  --application-name eb-worker-app --terminate-env-by-force

# Double-check nothing is left running
aws elasticbeanstalk describe-environments \
  --query "Environments[?Status!='Terminated'].[EnvironmentName,Status]" --output table

# Double-check no orphaned EC2 instances remain
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].[InstanceId,Tags]"
```

Also manually verify in the console:
- **EC2 → Instances** — nothing running
- **EC2 → Load Balancers** — none left over
- **RDS → Databases** — none left over
- **Route 53** — remove any test records you created
- **ACM** — certificates are free, safe to leave or delete
- **S3** — the `elasticbeanstalk-<region>-<account-id>` bucket holds old app versions; safe to empty/delete if you're done for good

### ✅ Checkpoint
`aws elasticbeanstalk describe-environments` returns nothing but `Terminated` statuses (or nothing at all), and your AWS Billing Dashboard shows no unexpected running resources.

---

## 🎓 What You've Learned

By completing these labs, you've hands-on practiced:
- The full deploy loop (`init` → `create` → `deploy`)
- Infrastructure as Code via `.ebextensions`
- All major deployment policies and when to use each
- Decoupled, production-correct database architecture
- Asynchronous background processing with the Worker Tier
- Production HTTPS setup with a custom domain
- Full CI/CD automation
- Responsible cost management and cleanup

**Next step:** Keep [`troubleshooting.md`](./troubleshooting.md) open in a tab the next time something breaks — because something always breaks eventually, and that's normal. 🛠️
