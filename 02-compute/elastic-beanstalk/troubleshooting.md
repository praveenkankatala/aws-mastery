# AWS Elastic Beanstalk — Troubleshooting Guide

> Every one of these has happened to real people learning EB — most of them happened to whoever wrote this guide too. Find your symptom, understand *why* it happens (not just the fix), and move on.

📎 Related: [README.md](./README.md) · [commands-cheatsheet.md](./commands-cheatsheet.md) · [hands-on-labs.md](./hands-on-labs.md)

---

## 🧭 How to Use This Guide

Each entry follows the same structure:
- **Symptom** — what you're seeing
- **Root Cause** — why it actually happens (understanding this prevents the *next* five bugs)
- **Fix** — the concrete steps to resolve it
- **Prevention** — how to stop it from happening again

---

## 🖥️ CLI & Setup Issues

### `eb: command not found` {#eb-command-not-found}

**Symptom:** After `pip install awsebcli --upgrade --user`, running `eb` gives "command not found."

**Root Cause:** `pip install --user` installs executables into a user-local `bin`/`Scripts` folder that isn't automatically on your system `PATH`.

**Fix:**
```bash
# Find where it was installed
python3 -m site --user-base
# e.g., /Users/you/.local  →  binary is at /Users/you/.local/bin

# Add to PATH (add this line to ~/.zshrc, ~/.bashrc, or your shell profile)
export PATH="$HOME/.local/bin:$PATH"

# Reload your shell
source ~/.zshrc   # or ~/.bashrc
```
On Windows, add the `Scripts` folder shown by `python -m site --user-site` (replace `site-packages` with `Scripts`) to your System PATH via Environment Variables settings.

**Prevention:** Consider installing inside a Python virtual environment where `bin`/`Scripts` is automatically active, or use `pipx install awsebcli` which handles PATH setup for you.

---

### `Unable to locate credentials`

**Symptom:** Any `eb` or `aws` command fails immediately with a credentials error.

**Root Cause:** `aws configure` was never run, was run for a different profile, or your credentials expired/were revoked.

**Fix:**
```bash
aws configure
# Re-enter Access Key ID, Secret Access Key, region

# Verify it worked
aws sts get-caller-identity
```
If you use named profiles:
```bash
aws configure --profile eb-learner
export AWS_PROFILE=eb-learner
```

**Prevention:** Never delete/rotate an IAM access key while `aws configure` still references it without updating `~/.aws/credentials` at the same time.

---

### `eb init` picks the wrong region or platform

**Symptom:** Environment gets created in `us-east-1` when you meant `ap-south-1`, or the wrong language runtime is selected.

**Root Cause:** `eb init` remembers your last choice in `.elasticbeanstalk/config.yml`, and non-interactive flags weren't passed.

**Fix:**
```bash
eb init --interactive
# Walk through the prompts again and pick correctly this time

# Or directly edit .elasticbeanstalk/config.yml:
# default_region: ap-south-1
```

**Prevention:** Always specify `--region` and `--platform` explicitly when scripting `eb init` for reproducibility.

---

## 🚀 Deployment Issues

### Deployment fails with "ERROR: ServiceError - Environment update finished unsuccessfully"

**Symptom:** `eb deploy` runs, uploads fine, then fails partway through with a vague service error.

**Root Cause:** Usually a health-check failure — the new application version failed to start correctly, so EB refused to route traffic to it (this is EB protecting you from a bad deploy).

**Fix:**
```bash
# Pull the real error from the logs — the CLI error is rarely the actual reason
eb logs

# Look specifically at:
#   /var/log/eb-engine.log      (deployment/platform engine errors)
#   /var/log/web.stdout.log     (your app's own errors)
```
Common underlying causes found here: missing dependency in `requirements.txt`/`package.json`, wrong entry-point filename, app crashing on startup due to a missing environment variable, or a port binding mismatch.

**Prevention:** Test your app **locally** first with the exact same start command EB will use, and keep dependency files in sync with what you actually import.

---

### `eb deploy` says "Nothing to commit" or deploys old code

**Symptom:** You edited files, ran `eb deploy`, but the live site still shows the old version.

**Root Cause:** `eb deploy` bundles your **last git commit** by default — uncommitted changes are silently ignored.

**Fix:**
```bash
git add .
git commit -m "Your change description"
eb deploy

# OR, to deploy uncommitted/staged changes directly:
eb deploy --staged
```

**Prevention:** Make committing part of muscle memory: `git add . && git commit -m "..." && eb deploy` as one mental unit.

---

### Deployment "hangs" or takes an extremely long time

**Symptom:** `eb create` or `eb deploy` sits for 15+ minutes with no progress.

**Root Cause:** Usually one of: (a) a health check is failing repeatedly and EB keeps retrying, (b) instance can't reach the internet to download dependencies (no NAT Gateway / wrong subnet in a custom VPC), or (c) a `.ebextensions` script is hanging (e.g., an interactive package install prompt).

**Fix:**
```bash
# Check events for repeated failures
eb events --follow

# If it's a VPC networking issue, verify instances are either
# in a public subnet with an IGW, or a private subnet with a NAT Gateway
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<your-vpc-id>"

# If stuck, abort and investigate rather than waiting indefinitely
eb abort
```

**Prevention:** Always test `.ebextensions` scripts for non-interactive flags (e.g., `apt-get install -y`, never a bare `apt-get install`).

---

## 🩺 Health & Runtime Issues

### Environment health shows "Severe" or "Degraded" (red/yellow)

**Symptom:** `eb health` or the console dashboard shows a red or yellow status.

**Root Cause:** Could be application-level (5xx errors, crashes) or infrastructure-level (failed instance health checks, CPU/memory exhaustion).

**Fix:**
```bash
# Get the specific cause, not just the color
eb health --refresh
# Look at the "Causes" column — e.g., "100% of the requests are erroring
# with HTTP 5xx" or "Instance deployment failed"

eb logs --all
```
Match the cause to a fix:
- **5xx spike** → check application logs for stack traces
- **Failed instance health checks** → check `/var/log/eb-engine.log` for startup failures
- **High CPU/memory** → your instance type may be undersized for the load; check `eb config` and consider a larger instance type or lower thread/worker counts

**Prevention:** Enable **Enhanced Health Reporting** (it's default on modern platforms, but verify via `.ebextensions`) — Basic reporting only gives you a color with no explanation.

---

### `502 Bad Gateway` when visiting the environment URL

**Symptom:** The load balancer/Nginx responds, but with a 502.

**Root Cause:** Nginx (the reverse proxy) is up and running, but it can't reach your application process — meaning your app crashed, isn't listening on the expected port, or hasn't started yet.

**Fix:**
```bash
eb ssh
# Once connected:
sudo systemctl status web    # or the relevant app service name for your platform
sudo journalctl -u web -n 100 --no-pager
```
Check that your app is listening on the port EB expects (commonly port `8000` internally, proxied from `80`/`443` — this varies slightly by platform, check platform docs). Confirm your `Procfile` (if used) or platform-specific startup command is correct.

**Prevention:** Always include a lightweight `/health` endpoint in your app and verify it returns `200` locally before deploying.

---

### App works via `eb ssh` manually but not through the browser

**Symptom:** Running the app manually on the instance works, but the live URL still 502s or shows nothing.

**Root Cause:** You're likely running the app on a different port/interface than what the platform's reverse-proxy config expects, or the platform's process manager (systemd) isn't the one actually starting your app — you started it manually outside that supervision.

**Fix:** Don't run apps manually for anything beyond debugging — always deploy through `eb deploy` so the platform's process manager and Nginx config stay in sync. If you need a custom start command, define it properly via a `Procfile` (most platforms) rather than SSH-launching it yourself.

**Prevention:** Treat `eb ssh` as read-only/diagnostic access, not a way to "fix" a running instance by hand — changes made this way disappear on the next deploy or instance replacement anyway.

---

### Environment stuck in "Updating" state indefinitely

**Symptom:** The console shows the environment perpetually "Updating" and no new commands seem to take effect.

**Root Cause:** A previous update didn't finish cleanly, or a health-check requirement for the deployment policy (e.g., Immutable) is permanently failing so EB never completes the swap.

**Fix:**
```bash
# Cancel the stuck update first
eb abort

# Then check what actually failed
eb events --max-items 30
```
If `eb abort` doesn't unstick it, check the CloudFormation stack directly in the console (Elastic Beanstalk stacks are prefixed `awseb-*`) — sometimes a manually-deleted resource (like a security group someone removed by hand) leaves CloudFormation unable to reconcile state.

**Prevention:** Never manually delete or modify EB-managed resources (security groups, launch configs, etc.) directly in their respective consoles — always go through EB's config/`.ebextensions`.

---

## 🔐 Permissions & IAM Issues

### "User is not authorized to perform: elasticbeanstalk:CreateApplication"

**Symptom:** Any `eb`/`aws elasticbeanstalk` command fails with an explicit `AccessDenied`.

**Root Cause:** The IAM user/role you're authenticated as doesn't have the necessary permissions attached.

**Fix:** Attach the AWS managed policy `AdministratorAccess-AWSElasticBeanstalk` (for learning) to your IAM user, or work with your account admin to get a properly scoped custom policy.

**Prevention:** Set up correct IAM permissions in Lab 0 *before* starting, rather than debugging it mid-deployment.

---

### App can't access S3/DynamoDB/other AWS services from inside the instance

**Symptom:** Your application code gets `AccessDenied` calling other AWS services, even though your *personal* AWS CLI credentials clearly have access.

**Root Cause:** This is the single most common EB confusion — **your app runs under the Instance Profile, not your personal credentials, and definitely not the EB Service Role.** These are three separate identities.

**Fix:**
1. Find the instance profile: **EB Console → Configuration → Security → EC2 instance profile**
2. Go to **IAM → Roles →** that instance profile's role
3. Attach a policy granting exactly the permissions your app needs (e.g., `AmazonS3ReadOnlyAccess` if it only reads from S3)

**Prevention:** Remember the split — **Service Role** = EB managing AWS resources on your behalf; **Instance Profile** = your running code calling AWS APIs. Debug permission errors by first asking "which of these two is actually making this call?"

---

### SSH into instance fails with "Permission denied (publickey)"

**Symptom:** `eb ssh` fails to connect.

**Root Cause:** No keypair was configured during `eb init`, or the local private key doesn't match what's registered.

**Fix:**
```bash
# Reconfigure SSH via init
eb init --interactive
# When asked "Do you want to set up SSH for your instances?", choose Yes
# and either create a new keypair or select an existing one

# Then retry
eb ssh
```

**Prevention:** Always opt into SSH setup during `eb init`, even if you don't think you'll need it — adding it retroactively requires recreating the environment in some cases, since existing instances launched without a keypair can't retroactively gain one.

---

## 📦 `.ebextensions` & Configuration Issues

### `.ebextensions` config is silently ignored

**Symptom:** You added a `.config` file but nothing changed after deploy.

**Root Cause:** Most commonly: the folder is misspelled (`.ebextension` vs `.ebextensions`), the file isn't valid YAML/JSON, or it's not actually included in the zipped source bundle (e.g., excluded by `.gitignore` when EB deploys from git).

**Fix:**
```bash
# Confirm exact folder name and location (must be at project root)
ls -la .ebextensions/

# Validate YAML syntax
python3 -c "import yaml; yaml.safe_load(open('.ebextensions/01-environment.config'))"

# Confirm git isn't excluding it
git status .ebextensions/
git check-ignore -v .ebextensions/01-environment.config   # should print nothing
```

**Prevention:** Always `git add` and confirm `git status` shows your `.ebextensions` files as tracked before deploying — a `.gitignore` rule can silently exclude the very folder that would have fixed your problem.

---

### Conflicting settings between console changes and `.ebextensions`

**Symptom:** A setting keeps reverting, or behaves inconsistently between deploys.

**Root Cause:** Manual changes made in the console are **overridden** by `.ebextensions` on the next deploy, since the config files are the declared source of truth every time code is pushed.

**Fix:** Decide on one source of truth. If you want a setting to persist, put it in `.ebextensions`, not the console. If you must adjust something quickly via console for an emergency, follow up by codifying that same change in `.ebextensions` so it isn't lost on the next deploy.

**Prevention:** Treat the console as read-only/diagnostic for any environment that has `.ebextensions` config — the two should never be your "two sources of truth" for the same setting.

---

## 🗄️ Database Connectivity Issues

### App can't connect to RDS: "Connection timed out"

**Symptom:** `/db-check` (or any DB call) hangs and times out.

**Root Cause:** Almost always a **Security Group** issue — the RDS instance's security group doesn't allow inbound traffic from the EB instances' security group.

**Fix:**
1. Note your EB environment's EC2 security group ID (**EB Console → Configuration → Instances**)
2. Go to the RDS instance's security group → **Inbound rules → Edit**
3. Add a rule: **Type:** PostgreSQL/MySQL (per your engine), **Source:** the EB EC2 security group ID (not an IP range)

**Prevention:** When creating a decoupled RDS instance, set up this security group rule immediately as part of provisioning — don't wait for the first connection failure to discover it's missing.

---

### App can't connect to RDS: "password authentication failed"

**Symptom:** Connection reaches the database but is rejected.

**Root Cause:** Wrong credentials in the `DATABASE_URL` environment variable, or special characters in the password weren't URL-encoded.

**Fix:**
```bash
# Verify what EB actually has set
eb printenv | grep DATABASE_URL

# If your password has special characters like @ : / ?, URL-encode them
# e.g., a password "p@ss/word" becomes "p%40ss%2Fword" in the connection string
```

**Prevention:** Avoid special characters in auto-generated database passwords when possible, or always run them through a URL-encoding step before building the connection string.

---

## 🌐 Networking, Domain & HTTPS Issues

### Custom domain shows "This site can't be reached"

**Symptom:** `app.yourdomain.com` doesn't resolve at all.

**Root Cause:** DNS record either wasn't created, points to the wrong target, or hasn't propagated yet.

**Fix:**
```bash
# Check what your domain currently resolves to
dig app.yourdomain.com

# Compare against your EB environment's actual load balancer DNS name
aws elasticbeanstalk describe-environment-resources \
  --environment-name my-env --query "EnvironmentResources.LoadBalancers"
```
Confirm the Route 53 record is an **Alias (A record)** pointing at the load balancer (not a CNAME to the raw EB URL, which won't work for apex domains and is less efficient for subdomains too).

**Prevention:** DNS changes can take a few minutes up to 48 hours depending on TTL — always allow propagation time before assuming something is broken.

---

### HTTPS works but shows a certificate warning

**Symptom:** Browser shows "Not Secure" or a cert mismatch warning.

**Root Cause:** The ACM certificate's domain name doesn't exactly match the domain you're visiting (e.g., cert issued for `yourdomain.com` but you're visiting `www.yourdomain.com`), or the cert is still in `PENDING_VALIDATION`.

**Fix:**
```bash
aws acm describe-certificate --certificate-arn <your-cert-arn> \
  --query "Certificate.Status"
# Must show "ISSUED"
```
If mismatched, request a new certificate covering both the apex and `www` subdomain (or a wildcard `*.yourdomain.com`).

**Prevention:** When requesting a certificate, always add both `yourdomain.com` and `www.yourdomain.com` (or use a wildcard) as Subject Alternative Names up front.

---

## 🐳 Docker Platform Issues

### Docker environment fails with "failed to build image"

**Symptom:** Deploying a Docker-platform EB app fails during the image build step.

**Root Cause:** Usually a broken `Dockerfile`, a missing build context file, or an architecture mismatch (building for ARM locally but deploying to an x86 EB instance type, or vice versa).

**Fix:**
```bash
# Test the build locally first, exactly as EB would run it
docker build -t test-image .

# If it's an architecture issue, build explicitly for the target platform
docker build --platform linux/amd64 -t test-image .
```

**Prevention:** Always validate `docker build` succeeds locally before running `eb deploy` — EB's error output for failed builds is often less detailed than Docker's own.

---

## 💸 Cost & Resource Issues

### Unexpected AWS bill after finishing the labs

**Symptom:** A charge appears days/weeks after you thought you were done.

**Root Cause:** An environment, RDS instance, Elastic IP, or Load Balancer was left running — EB doesn't auto-delete anything unless you explicitly terminate it.

**Fix:** Follow [Lab 8: Cleanup & Cost Control](./hands-on-labs.md#lab-8-cleanup--cost-control) precisely, then verify in the console:
```bash
aws elasticbeanstalk describe-environments \
  --query "Environments[?Status!='Terminated']"
aws rds describe-db-instances --query "DBInstances[].DBInstanceIdentifier"
aws ec2 describe-addresses   # unattached Elastic IPs bill hourly!
```

**Prevention:** Set up an **AWS Budget alert** (Billing Console → Budgets) for a low threshold (e.g., $5) so you're notified immediately if anything is accidentally left running.

---

## 🧠 General Debugging Checklist

When something breaks and you're not sure where to start, work through this order:

1. `eb status` — is the environment even healthy?
2. `eb events --follow` — what's the most recent thing EB tried to do, and did it fail?
3. `eb health --refresh` — what specific cause is reported?
4. `eb logs --all` — what does the application itself say, in its own log output?
5. `eb ssh` — if logs aren't enough, connect and inspect the process/service directly
6. Check **Security Groups** — is this actually a networking/permissions problem disguised as an app problem?
7. Check **IAM** — Service Role vs. Instance Profile — which one does this operation actually need?
8. As a last resort, check the underlying **CloudFormation stack** (`awseb-*`) in the console for the literal AWS-level error message

---

**That's the full picture.** Between [README.md](./README.md) for the "why," [hands-on-labs.md](./hands-on-labs.md) for the "how," and this file for "when it breaks," you now have everything needed to run Elastic Beanstalk confidently in a real project. 🎉
