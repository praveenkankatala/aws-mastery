# AWS Elastic Beanstalk — CLI Commands Cheat Sheet

> Every command organized by what you're actually trying to do — not just an alphabetical dump. Bookmark this page.

📎 Related: [README.md](./README.md) · [hands-on-labs.md](./hands-on-labs.md) · [troubleshooting.md](./troubleshooting.md)

---

## 🧭 Table of Contents

1. [Installation & Setup](#-installation--setup)
2. [Project Initialization](#-project-initialization)
3. [Creating & Managing Environments](#-creating--managing-environments)
4. [Deploying Code](#-deploying-code)
5. [Checking Status & Health](#-checking-status--health)
6. [Logs & Debugging](#-logs--debugging)
7. [Environment Variables & Configuration](#-environment-variables--configuration)
8. [Scaling](#-scaling)
9. [SSH & Instance Access](#-ssh--instance-access)
10. [Blue/Green Deployments & Swapping](#-bluegreen-deployments--swapping)
11. [Cloning Environments](#-cloning-environments)
12. [Application Versions](#-application-versions)
13. [Platforms](#-platforms)
14. [Tags](#-tags)
15. [Cleanup / Termination](#-cleanup--termination)
16. [Equivalent AWS CLI Commands](#-equivalent-aws-cli-commands)
17. [Quick Reference Table (All Commands)](#-quick-reference-table-all-commands)

---

## 📦 Installation & Setup

```bash
# Install the AWS CLI v2 (if not already installed) — macOS example
brew install awscli

# Install the AWS CLI v2 — Linux example
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure your AWS credentials
aws configure
# Prompts for: Access Key ID, Secret Access Key, Default Region, Output format

# Install the EB CLI (requires Python + pip)
pip install awsebcli --upgrade --user

# Verify installations
aws --version
eb --version
```

---

## 🏗️ Project Initialization

```bash
# Interactive initialization — run inside your project's root folder
eb init

# Non-interactive init (great for scripting/CI)
eb init my-app --platform "Python 3.12" --region us-east-1

# Init with a specific existing keypair for SSH access
eb init --interactive
# (then choose "yes" when asked about SSH)

# List available platforms (to use with --platform)
eb platform list

# Re-run init if you need to change the platform or region later
eb init
```

**What `eb init` actually does:** creates a `.elasticbeanstalk/config.yml` file in your project that stores your application name, default region, and platform — so every future `eb` command knows what you're working with without re-specifying flags.

---

## 🚀 Creating & Managing Environments

```bash
# Create a new environment with defaults
eb create my-env

# Create with specific instance type
eb create my-env --instance-type t3.micro

# Create a Single Instance environment (no load balancer — cheaper, dev/test)
eb create my-env --single

# Create with a specific number of instances (min/max for Auto Scaling)
eb create my-env --min-instances 2 --max-instances 4

# Create in a specific VPC
eb create my-env --vpc.id vpc-xxxxxxxx --vpc.ec2subnets subnet-aaa,subnet-bbb --vpc.elbsubnets subnet-ccc,subnet-ddd

# Create with environment variables set at creation time
eb create my-env --envvars KEY1=value1,KEY2=value2

# Create a Worker Tier environment
eb create my-worker-env --tier worker

# List all environments for the current application
eb list

# Switch the "default" environment eb commands target
eb use my-env

# View full configuration of the current environment
eb config

# Edit configuration interactively (opens in your default editor)
eb config my-env

# Save the current environment's config as a reusable template
eb config save my-env --cfg my-template-name

# Update an environment from a saved config template
eb config put my-template-name
```

---

## 📤 Deploying Code

```bash
# Deploy the current directory's code to the current environment
eb deploy

# Deploy to a specific (non-default) environment
eb deploy my-other-env

# Deploy with a custom application version label
eb deploy --label v1.2.3

# Deploy a specific, already-uploaded application version (rollback/promote)
eb deploy --version v1.2.0

# Deploy with a message describing the change
eb deploy --message "Fix checkout bug"

# Stage changes without committing to git first (deploy uncommitted changes)
eb deploy --staged

# Preview what would be deployed without actually deploying (dry check via status)
eb status
```

> 💡 **Note:** `eb deploy` by default zips and deploys your **last git commit**. Uncommitted changes are ignored unless you pass `--staged` (which uses `git add`-ed files) — this trips up beginners constantly.

---

## 🩺 Checking Status & Health

```bash
# Basic status — environment name, health, URL, running version
eb status

# Detailed status
eb status --verbose

# Enhanced health report (per-instance breakdown)
eb health

# Continuously refreshing health view (like `top` for your environment)
eb health --refresh

# View recent environment events (deployment history, errors, warnings)
eb events

# Follow events in real time (like `tail -f`)
eb events --follow

# Open the environment URL in your default browser
eb open
```

---

## 📜 Logs & Debugging

```bash
# Pull the last 100 lines of logs from all instances
eb logs

# Pull a full log bundle (more detailed, downloaded as files)
eb logs --all

# Stream logs continuously (requires log streaming enabled)
eb logs --stream

# Get logs from a specific instance (useful in multi-instance environments)
eb logs --instance i-0123456789abcdef0

# Zip and download logs locally
eb logs --zip
```

---

## 🔧 Environment Variables & Configuration

```bash
# Set one or more environment variables
eb setenv DATABASE_URL=postgres://... DEBUG=false

# Print current environment variables
eb printenv

# Print env vars for a specific environment
eb printenv my-other-env
```

---

## 📈 Scaling

```bash
# Manually scale to a specific number of instances
eb scale 4

# Scale a specific environment
eb scale 4 --env my-env

# (For fine-grained scaling policy tuning — min/max/CPU thresholds —
#  use `eb config` or define via .ebextensions, see hands-on-labs.md)
```

---

## 🔑 SSH & Instance Access

```bash
# SSH into an instance in the environment (requires a keypair set during eb init)
eb ssh

# SSH into a specific instance (when multiple exist)
eb ssh --instance i-0123456789abcdef0

# SSH with port forwarding / custom SSH args
eb ssh --command "tail -f /var/log/eb-engine.log"
```

---

## 🔄 Blue/Green Deployments & Swapping

```bash
# Step 1: Create a brand-new "green" environment running the new version
eb create my-env-green --version v2.0.0

# Step 2: Test the green environment thoroughly via its own URL
eb status my-env-green

# Step 3: Swap CNAMEs between blue (live) and green (staged) — instant cutover
eb swap my-env-blue --destination_name my-env-green

# Step 4: Once confident, terminate the old (now-idle) environment
eb terminate my-env-blue
```

---

## 🧬 Cloning Environments

```bash
# Clone an existing environment's configuration into a new one
eb clone my-env --clone_name my-env-clone

# Clone into a specific region
eb clone my-env --clone_name my-env-clone --region us-west-2
```

---

## 🏷️ Application Versions

```bash
# List application versions (via AWS CLI — EB CLI has no direct "list versions")
aws elasticbeanstalk describe-application-versions --application-name my-app

# Deploy a specific previously-uploaded version (fast rollback)
eb deploy --version v1.0.0

# Delete an old application version (via AWS CLI)
aws elasticbeanstalk delete-application-version \
  --application-name my-app --version-label v0.9.0 --delete-source-bundle
```

---

## 🖥️ Platforms

```bash
# List all available managed platforms
eb platform list

# Show details of the platform used by the current environment
eb platform show

# Select/change platform for the current environment
eb platform select
```

---

## 🏷️ Tags

```bash
# Add tags to an environment
eb tags --add Project=Learning,Owner=YourName

# Remove tags
eb tags --delete Project

# List current tags
eb tags --list
```

---

## 🧹 Cleanup / Termination

```bash
# Terminate a specific environment (deletes EC2, ELB, ASG — NOT the application itself)
eb terminate my-env

# Terminate without confirmation prompt (careful!)
eb terminate my-env --force

# Delete the entire application (all environments + versions must be gone first)
aws elasticbeanstalk delete-application --application-name my-app --terminate-env-by-force
```

> ⚠️ **Always double check** `eb status` before terminating — this deletes real infrastructure. If you have a coupled RDS database, make sure deletion protection / snapshot-on-terminate is set the way you want **before** running this.

---

## 🔗 Equivalent AWS CLI Commands

The EB CLI is a friendly wrapper — everything it does maps to the underlying `aws elasticbeanstalk` API. Useful for CI/CD pipelines or when the EB CLI isn't installed.

```bash
# List applications
aws elasticbeanstalk describe-applications

# List environments
aws elasticbeanstalk describe-environments --application-name my-app

# Describe environment health
aws elasticbeanstalk describe-environment-health \
  --environment-name my-env --attribute-names All

# Create an application
aws elasticbeanstalk create-application --application-name my-app

# Create an environment
aws elasticbeanstalk create-environment \
  --application-name my-app \
  --environment-name my-env \
  --solution-stack-name "64bit Amazon Linux 2023 v4.1.1 running Python 3.12" \
  --option-settings file://options.json

# Create an application version (after uploading source bundle to S3)
aws elasticbeanstalk create-application-version \
  --application-name my-app \
  --version-label v1.0.0 \
  --source-bundle S3Bucket="my-bucket",S3Key="app-v1.0.0.zip"

# Update an environment to a new version
aws elasticbeanstalk update-environment \
  --environment-name my-env \
  --version-label v1.0.0

# Terminate an environment
aws elasticbeanstalk terminate-environment --environment-name my-env

# Describe recent events
aws elasticbeanstalk describe-events --environment-name my-env --max-records 20
```

---

## 📋 Quick Reference Table (All Commands)

| Command | Purpose |
|---|---|
| `eb init` | Initialize/link a project to Elastic Beanstalk |
| `eb create` | Provision a new environment |
| `eb deploy` | Deploy the latest code to an environment |
| `eb status` | Show environment status summary |
| `eb health` | Show detailed instance-level health |
| `eb events` | Show recent environment events |
| `eb logs` | Retrieve application/server logs |
| `eb open` | Open the environment URL in a browser |
| `eb setenv` | Set environment variables |
| `eb printenv` | Print current environment variables |
| `eb config` | View or edit environment configuration |
| `eb scale` | Manually change instance count |
| `eb ssh` | SSH into an instance |
| `eb swap` | Swap CNAMEs between two environments (blue/green) |
| `eb clone` | Clone an environment's configuration |
| `eb list` | List all environments |
| `eb use` | Set the default environment for commands |
| `eb tags` | Manage environment tags |
| `eb platform` | View/select platform information |
| `eb terminate` | Tear down an environment |
| `eb abort` | Cancel an in-progress deployment/update |
| `eb restore` | Restore a previously terminated environment |
| `eb labs` | Access experimental EB CLI features |

---

**Next step:** Put these into practice in [`hands-on-labs.md`](./hands-on-labs.md). 🧪
