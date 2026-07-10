# AWS EKS — Troubleshooting Playbook

> Real-world failures you'll hit on EKS, why they happen, and exactly how to fix them. Organized by symptom.

## How to Use This Guide

1. Find the symptom (pod pending, ALB not created, kubectl unauthorized, etc.).
2. Read the "why" — root cause matters more than the fix.
3. Run the diagnostic commands to confirm.
4. Apply the fix.

## Table of Contents

1. [Pod scheduling & lifecycle](#1-pod-scheduling--lifecycle)
2. [Networking & DNS](#2-networking--dns)
3. [VPC CNI & IP exhaustion](#3-vpc-cni--ip-exhaustion)
4. [Ingress & LoadBalancer issues](#4-ingress--loadbalancer-issues)
5. [IAM, RBAC & authentication](#5-iam-rbac--authentication)
6. [Node & cluster health](#6-node--cluster-health)
7. [Karpenter troubleshooting](#7-karpenter-troubleshooting)
8. [Storage (EBS / EFS) issues](#8-storage-ebs--efs-issues)
9. [Upgrade failures](#9-upgrade-failures)
10. [kubectl and access issues](#10-kubectl-and-access-issues)
11. [Cost & orphaned resources](#11-cost--orphaned-resources)
12. [Diagnostic command toolkit](#12-diagnostic-command-toolkit)

---

## 1. Pod Scheduling & Lifecycle

### Symptom: Pod stuck in `Pending`

**Diagnostic:**
```bash
kubectl describe pod <pod> -n <ns> | tail -30
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

**Common root causes and fixes:**

| Cause | Signal in `describe` | Fix |
|-------|---------------------|-----|
| No node has enough CPU/memory | `0/N nodes are available: N Insufficient cpu` | Scale up the node group, install/tune Karpenter, or lower pod requests |
| No node matches node selector / affinity | `didn't match Pod's node affinity/selector` | Fix `nodeSelector` labels or add matching labels to nodes |
| Taints not tolerated | `had untolerated taint` | Add matching `tolerations` to the pod spec |
| PVC pending (WaitForFirstConsumer) | Bound to a PVC that's `Pending` | See [Storage section](#8-storage-ebs--efs-issues) |
| No pod IPs available on nodes | See [IP exhaustion](#3-vpc-cni--ip-exhaustion) | Enable prefix delegation or add larger subnets |

### Symptom: `ImagePullBackOff` / `ErrImagePull`

**Diagnostic:**
```bash
kubectl describe pod <pod> -n <ns>
```

**Fixes:**
- **Typo in the image name or tag** — check `spec.containers[].image`.
- **Private registry, no imagePullSecret** — create a `docker-registry` secret and reference it via `imagePullSecrets`.
- **ECR authentication** — nodes need `AmazonEC2ContainerRegistryReadOnly` on their role. Verify with `aws ecr get-authorization-token` from the node.
- **Cross-region ECR** — nodes can pull cross-region but slowly; consider replicating the repo.
- **Rate-limited from Docker Hub** — mirror the image into ECR.

### Symptom: `CrashLoopBackOff`

**Diagnostic:**
```bash
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous   # last crashed container
kubectl describe pod <pod> -n <ns>
```

**Fixes:**
- **App crashes on startup** — read the previous logs; fix the config or code.
- **OOMKilled** — see `Last State: Terminated, Reason: OOMKilled`. Increase memory limits.
- **Failing liveness probe** — the probe fires before the app is ready; add `initialDelaySeconds` or increase the threshold.
- **Missing config/secret** — check for `Error: couldn't find key` in events; create the Secret/ConfigMap.

### Symptom: Pod runs but is not `Ready` (0/1)

Cause: Readiness probe fails.
```bash
kubectl describe pod <pod> -n <ns>   # look for "Readiness probe failed"
```
Fix the probe path/port or increase `periodSeconds` / `failureThreshold`.

### Symptom: `Evicted` pods piling up

Nodes were under memory or disk pressure.
```bash
kubectl get pods -A --field-selector=status.phase=Failed
kubectl delete pods -A --field-selector=status.phase=Failed
```
Root cause: pods without limits, or noisy neighbors. Add proper `requests`/`limits` and use `PriorityClass` for critical workloads.

---

## 2. Networking & DNS

### Symptom: Pod can't resolve `svc.cluster.local` names

**Diagnostic:**
```bash
kubectl run -it --rm dns-debug --image=nicolaka/netshoot --restart=Never -- \
  nslookup kubernetes.default
```

**Fixes:**
- **CoreDNS pods not running:**
  ```bash
  kubectl get pods -n kube-system -l k8s-app=kube-dns
  kubectl logs -n kube-system -l k8s-app=kube-dns
  ```
  If missing, install the CoreDNS add-on.
- **CoreDNS scaled to zero** — happens after upgrade blips. Scale it back:
  ```bash
  kubectl scale deployment coredns -n kube-system --replicas=2
  ```
- **Security group blocking port 53** — cluster SG must allow UDP/TCP 53 within the VPC.
- **Node running out of `conntrack` entries** — bump `net.netfilter.nf_conntrack_max` via bootstrap script.

### Symptom: Pod-to-pod traffic blocked

- Check `NetworkPolicy` objects — a default-deny policy blocks all ingress until you add specific `allow` rules.
- Check the security group attached to the ENIs — it must allow intra-cluster traffic.

### Symptom: Pod can't reach the internet

- Check that private subnets have a route to a NAT Gateway.
- Check the route table associated with the subnet (`0.0.0.0/0` → NAT).
- If using PrivateLink endpoints, confirm the endpoint for `s3`, `ecr`, `sts` exists in the VPC.

### Symptom: External services can't reach the pod

- If it's a `ClusterIP` service, it's cluster-internal only. Use `LoadBalancer` or `Ingress`.
- Check `NetworkPolicy` for ingress restrictions.

---

## 3. VPC CNI & IP Exhaustion

### Symptom: Pods can't schedule because "no available IP addresses"

This is the classic VPC CNI failure. The default CNI hands each pod a real VPC IP; a busy node runs out.

**Diagnostic:**
```bash
# Check aws-node logs
kubectl logs -n kube-system -l k8s-app=aws-node --tail=100 | grep -i "ipam\|no free"

# Check ENIs on a node
INSTANCE_ID=$(kubectl get node <node> -o jsonpath='{.spec.providerID}' | awk -F'/' '{print $NF}')
aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query "Reservations[].Instances[].NetworkInterfaces[].{ENI:NetworkInterfaceId,IPs:PrivateIpAddresses[].PrivateIpAddress}"
```

**Fixes:**

1. **Enable prefix delegation** — pack `/28` prefixes instead of individual IPs (~16 IPs per allocation):
   ```bash
   kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
   ```
   Rolling-restart nodes to pick up the change.

2. **Tune WARM_IP_TARGET / MINIMUM_IP_TARGET** — don't pre-allocate more IPs than you need:
   ```bash
   kubectl set env daemonset aws-node -n kube-system WARM_IP_TARGET=5 MINIMUM_IP_TARGET=10
   ```

3. **Use larger subnets** — a `/24` (256 IPs) is a common landmine. Move workloads to `/20` subnets.

4. **Custom networking mode** — use secondary CIDR blocks on your VPC and assign pods IPs from a non-primary range. Set `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true` and create `ENIConfig` CRs per AZ.

### Symptom: `aws-node` pod in `CrashLoopBackOff`

- Check the IAM role attached to the node has `AmazonEKS_CNI_Policy`.
- Verify the CNI plugin version matches the cluster version (`aws eks describe-addon-versions`).

---

## 4. Ingress & LoadBalancer Issues

### Symptom: `Ingress` created but no ALB provisioned (empty `ADDRESS`)

**Diagnostic:**
```bash
kubectl describe ingress <name> -n <ns>
kubectl logs -n kube-system deploy/aws-load-balancer-controller --tail=100
```

**Root causes:**

| Root cause | Log signal | Fix |
|------------|-----------|-----|
| Controller not installed | `no endpoints available for service` when calling admission webhook | Install via Helm (see hands-on-labs Lab 4) |
| Missing IAM permissions | `AccessDenied` errors on `elasticloadbalancing:CreateLoadBalancer` | Reattach the AWS Load Balancer Controller policy to the service account role |
| Subnets not tagged | `couldn't auto-discover subnets` | Tag public subnets with `kubernetes.io/role/elb=1`, private with `kubernetes.io/role/internal-elb=1` |
| Wrong `ingressClassName` | Nothing happens | Set `spec.ingressClassName: alb` OR annotation `kubernetes.io/ingress.class: alb` |
| No matching subnets in enough AZs | `at least two subnets in two different Availability Zones` | Add more tagged subnets |

### Symptom: ALB provisioned but returns 5xx / 404

- **Target group empty:** confirm your service `selector` matches pod labels.
- **Health check failing:** ALB health-checks pods on `/`. Add an annotation:
  ```yaml
  alb.ingress.kubernetes.io/healthcheck-path: /healthz
  ```
- **`target-type: ip` needs pod IPs reachable** — ensure security groups allow the ALB to reach pods on the target port.

### Symptom: `Service` type `LoadBalancer` stuck in `Pending`

- If you have the AWS Load Balancer Controller installed, add:
  ```yaml
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
  ```
- Without the controller, EKS uses the legacy in-tree cloud provider — it should work but creates a Classic ELB. Check `kubectl describe svc` for events.

### Symptom: NLB doesn't preserve client IPs

By default NLB in "instance" mode masquerades the client IP. Fix with:
```yaml
service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
```

---

## 5. IAM, RBAC & Authentication

### Symptom: `error: You must be logged in to the server (Unauthorized)`

**Diagnostic:**
```bash
aws sts get-caller-identity     # confirms your IAM identity
kubectl config current-context  # confirms you're pointed at the right cluster
```

**Fixes:**

- **Your IAM identity isn't mapped into the cluster.**
  - For **Access Entries** clusters:
    ```bash
    aws eks list-access-entries --cluster-name <cluster>
    aws eks create-access-entry --cluster-name <cluster> \
      --principal-arn arn:aws:iam::<acct>:role/<your-role> --type STANDARD
    aws eks associate-access-policy --cluster-name <cluster> \
      --principal-arn arn:aws:iam::<acct>:role/<your-role> \
      --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
      --access-scope type=cluster
    ```
  - For **legacy `aws-auth` ConfigMap** clusters:
    ```bash
    kubectl edit configmap aws-auth -n kube-system
    ```
    Add your role under `mapRoles`.

- **Wrong AWS profile** — set `AWS_PROFILE` or run `aws configure`.
- **Assumed role but kubeconfig references user** — regenerate kubeconfig with the correct role:
  ```bash
  aws eks update-kubeconfig --name <cluster> --role-arn arn:aws:iam::<acct>:role/<role>
  ```

### Symptom: `Forbidden: user "X" cannot get resource "Y"`

Authentication worked (you're recognized), but authorization (RBAC) is missing.

```bash
# See what your identity can do
kubectl auth can-i --list

# Check a specific permission
kubectl auth can-i get pods --namespace default
```

Fix by adding a `RoleBinding` or `ClusterRoleBinding` that maps your identity to the right ClusterRole.

### Symptom: IRSA pod can't assume the role

**Diagnostic — from inside the pod:**
```bash
kubectl exec -it <pod> -- env | grep AWS_
# Should show AWS_ROLE_ARN and AWS_WEB_IDENTITY_TOKEN_FILE

kubectl exec -it <pod> -- aws sts get-caller-identity
```

**Root causes:**
- **Trust policy issuer mismatch** — the trust policy must reference the exact OIDC issuer URL of your cluster. Check with:
  ```bash
  aws eks describe-cluster --name <cluster> --query "cluster.identity.oidc.issuer"
  ```
- **ServiceAccount annotation missing:**
  ```yaml
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<acct>:role/<role>
  ```
- **Pod not using the annotated SA** — set `spec.serviceAccountName`.
- **OIDC provider not registered in IAM** — `eksctl utils associate-iam-oidc-provider --approve`.

### Symptom: Pod Identity association not working

- Ensure the `eks-pod-identity-agent` add-on is installed and running.
- Trust policy on the IAM role must use `pods.eks.amazonaws.com` (not `ec2.amazonaws.com`).
- Restart the pod after creating the association — env vars are injected at pod start.

---

## 6. Node & Cluster Health

### Symptom: Node stuck in `NotReady`

**Diagnostic:**
```bash
kubectl describe node <node>   # look at Conditions
aws ssm start-session --target <instance-id>
sudo journalctl -u kubelet -n 200 --no-pager
sudo systemctl status kubelet
```

**Common causes:**

| Cause | Fix |
|-------|-----|
| kubelet can't reach API server | Check security groups (443 to control plane ENI), check VPC routing |
| Disk pressure | Node's disk is full; increase EBS volume size, clean up images |
| Memory pressure | Reduce workloads, add larger instance types |
| `aws-node` (CNI) not running | Check DaemonSet, check IAM policy on node role |
| Wrong AMI or userdata | If using self-managed nodes, verify bootstrap script |

### Symptom: Node joined cluster but no pods can schedule on it

- Missing labels required by pod's `nodeSelector`.
- Node has a taint your pod isn't tolerating.
- `NoSchedule` taint set by Karpenter's cordon-during-consolidation.

### Symptom: All nodes get replaced unexpectedly

- Managed Node Group launch template updated → triggers a rolling replacement.
- Karpenter `expireAfter` reached (default 720h/30 days) — nodes recycle for security patching.
- AMI ID drift — a Terraform-managed node group can trigger replacement if the AMI ID resolves differently.

---

## 7. Karpenter Troubleshooting

### Symptom: Karpenter doesn't launch nodes despite pending pods

**Diagnostic:**
```bash
kubectl logs -f -n kube-system -l app.kubernetes.io/name=karpenter
kubectl get nodepool,ec2nodeclass
kubectl describe nodepool default
```

**Common root causes:**

| Cause | Log signal | Fix |
|-------|-----------|-----|
| No matching NodePool for pod requirements | `no nodepool matched` | Broaden NodePool `requirements` or add another NodePool |
| Subnets not tagged for discovery | `no subnets matched selector` | Add `karpenter.sh/discovery: <cluster>` tag on subnets |
| Security groups not tagged | `no security groups matched` | Tag SGs with `karpenter.sh/discovery: <cluster>` |
| IAM permissions missing | `UnauthorizedOperation` on RunInstances | Reattach Karpenter controller policy |
| NodeClass IAM role missing | `role does not exist` | Create `KarpenterNodeRole-<cluster>` instance profile |
| Hit the `limits.cpu` cap | `nodepool at cpu limit` | Raise the NodePool CPU limit |

### Symptom: Karpenter launches nodes but pods still don't schedule

- Node joins cluster but pod's `nodeSelector`/`affinity` still doesn't match — Karpenter picks based on requirements but not custom labels unless you tell it. Add labels via NodePool `spec.template.metadata.labels`.
- Pod requests are too large for any instance in the NodePool's allowed families.

### Symptom: Karpenter aggressively churns nodes (nodes disappearing every few minutes)

- Consolidation is too eager. Set `disruption.consolidationPolicy: WhenEmpty` instead of `WhenUnderutilized` if this is hurting you.
- Some workloads need `karpenter.sh/do-not-disrupt: "true"` annotation to survive consolidation.

### Symptom: Karpenter interruption handling not working (spot terminations)

- Ensure the SQS queue is created and `--set settings.interruptionQueue=<name>` is passed at install.
- EventBridge rules for spot ITN, rebalance, and scheduled maintenance must publish to that SQS.

---

## 8. Storage (EBS / EFS) Issues

### Symptom: PVC stuck in `Pending`

**Diagnostic:**
```bash
kubectl describe pvc <pvc> -n <ns>
kubectl logs -n kube-system -l app=ebs-csi-controller
```

**Fixes:**

| Cause | Signal | Fix |
|-------|--------|-----|
| No default StorageClass | `no default storage class set` | Annotate a SC as default |
| EBS CSI driver not installed | `no volume plugin matched` | Install the `aws-ebs-csi-driver` add-on |
| Driver lacks IAM permissions | `EBS volume creation failed: UnauthorizedOperation` | Attach `AmazonEBSCSIDriverPolicy` via IRSA or Pod Identity |
| `WaitForFirstConsumer` — no pod yet | PVC waits by design | Create the pod referencing the PVC |
| AZ mismatch | Volume created in wrong AZ | Use `WaitForFirstConsumer` mode so the volume is created in the pod's AZ |

### Symptom: Pod can't attach EBS volume

- **`MountVolume.MountDevice failed`** — the volume is already attached to another instance. This happens when a pod moves AZ. EBS is single-AZ; use `topologySpreadConstraints` to keep StatefulSet pods in the same AZ, or use EFS for multi-AZ.
- **Volume in wrong AZ** — see above; switch to `WaitForFirstConsumer`.

### Symptom: EBS volume orphaned after PVC deletion

- StorageClass `reclaimPolicy` is `Retain` — volume persists on purpose. Delete it manually:
  ```bash
  aws ec2 delete-volume --volume-id vol-xxx
  ```

### Symptom: EFS mount fails

- EFS mount targets must exist in each AZ where pods run.
- Security group on the mount target must allow inbound 2049 from the cluster SG.
- File system policy or access point misconfigured.

---

## 9. Upgrade Failures

### Symptom: `aws eks update-cluster-version` fails immediately

Common reasons:
- **API version skew** — control plane cannot be more than one minor version ahead of node kubelets. Upgrade node group first if you jumped a minor.
- **Deprecated APIs still in use** — Upgrade Insights will flag these. Fix or remove them before upgrading.

### Symptom: Control plane upgraded but pods break

- App uses a deprecated API version (e.g., `extensions/v1beta1`). Update manifests.
- Add-on version now incompatible. Upgrade `vpc-cni`, `coredns`, `kube-proxy`, `aws-ebs-csi-driver` to versions compatible with the new K8s minor.
- If unfixable in a hurry, **use the 7-day rollback**:
  ```bash
  aws eks update-cluster-version --name <cluster> --kubernetes-version <old> --force
  ```

### Symptom: Node group upgrade stuck

Managed NG uses rolling replacement. If a pod won't drain (PDB violation, orphaned pod), the upgrade hangs.

```bash
kubectl get pdb -A                # find restrictive PodDisruptionBudgets
kubectl get pods --field-selector=status.phase!=Running -A
```

Fix the PDB or force-delete stuck pods:
```bash
kubectl delete pod <pod> -n <ns> --grace-period=0 --force
```

### Symptom: Upgrade Insights shows issues you don't understand

Each insight has documentation. Common ones:
- **Deprecated API removals** — search-and-replace across your manifests.
- **kubelet certificate rotation** — should be automatic on managed nodes.
- **Add-on compatibility** — check version matrix, plan add-on updates in the same maintenance window.

---

## 10. kubectl and Access Issues

### Symptom: `Unable to connect to the server: dial tcp: i/o timeout`

- Cluster endpoint is `Private only` and you're not in the VPC → connect via VPN, Direct Connect, or SSM Session Manager on a bastion.
- Security group on your EC2 workstation doesn't allow outbound 443 → fix.
- Wrong region in kubeconfig → `aws eks update-kubeconfig --name <cluster> --region <r>`.

### Symptom: `error: exec plugin: invalid apiVersion "client.authentication.k8s.io/v1alpha1"`

Old kubeconfig format. Regenerate:
```bash
aws eks update-kubeconfig --name <cluster> --region <region>
```

### Symptom: kubectl very slow

- API server rate limiting because of too many list/watch clients. Check `kube-system` for chatty controllers.
- Old kubectl version. Upgrade to within one minor of the cluster version.
- The audit log volume may be huge — disable if not needed.

### Symptom: `The connection to the server ... was refused` after some idle time

Token expired. Re-run `aws eks update-kubeconfig`. For long-running sessions, use a role-assumed profile that auto-refreshes.

---

## 11. Cost & Orphaned Resources

### Symptom: AWS bill higher than expected after deleting the cluster

Common orphans:

```bash
# Orphaned Load Balancers
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerArn"

# Orphaned EBS volumes (unattached)
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[].{ID:VolumeId,Size:Size,AZ:AvailabilityZone}"

# Orphaned EIPs (unassociated)
aws ec2 describe-addresses --query "Addresses[?AssociationId==null]"

# Orphaned NAT Gateways
aws ec2 describe-nat-gateways --filter "Name=state,Values=available"

# Orphaned target groups (ALB-related)
aws elbv2 describe-target-groups --query "TargetGroups[].TargetGroupArn"
```

**Delete the cluster in the correct order** to avoid orphans:

1. Delete `Ingress` and `Service` type `LoadBalancer` → cleans up ALBs/NLBs.
2. Delete PVCs (with `reclaimPolicy: Delete`) → cleans up EBS volumes.
3. Uninstall Helm releases that provisioned AWS resources.
4. Delete the cluster.

### Symptom: Karpenter node not shutting down when idle

- `karpenter.sh/do-not-disrupt: "true"` annotation was set somewhere.
- `PodDisruptionBudget` blocking eviction.
- Pod without a controller (bare pod) — Karpenter won't disrupt these.

---

## 12. Diagnostic Command Toolkit

Bookmark these — they cover 80% of debugging.

### Cluster-wide health snapshot

```bash
# Every event, warnings only
kubectl get events -A --field-selector type=Warning --sort-by=.lastTimestamp

# All non-Running pods
kubectl get pods -A --field-selector=status.phase!=Running

# Node status summary
kubectl get nodes -o wide

# System pods
kubectl get pods -n kube-system
```

### Deep dive on a problem pod

```bash
POD=<pod>
NS=<namespace>

kubectl get pod $POD -n $NS -o yaml
kubectl describe pod $POD -n $NS
kubectl logs $POD -n $NS --all-containers=true --tail=200
kubectl logs $POD -n $NS --previous 2>/dev/null
kubectl get events -n $NS --field-selector involvedObject.name=$POD
```

### Node deep dive (via SSM, no SSH key required)

```bash
INSTANCE=<i-xxxxx>
aws ssm start-session --target $INSTANCE

# Once in:
sudo journalctl -u kubelet -n 500 --no-pager
sudo crictl ps -a
sudo crictl logs <container-id>
df -h                    # disk pressure
free -m                  # memory pressure
ip a                     # ENIs and IPs
```

### Network debugging with `netshoot`

```bash
kubectl run -it --rm netshoot --image=nicolaka/netshoot --restart=Never -- /bin/bash

# Inside:
nslookup <service>.<ns>.svc.cluster.local
dig kubernetes.default
curl -v http://<service>.<ns>:80
traceroute <ip>
```

### Prove your IAM identity vs kubectl identity

```bash
aws sts get-caller-identity                 # what AWS sees
kubectl auth whoami 2>/dev/null || \
  kubectl get --raw='/apis/authentication.k8s.io/v1/selfsubjectreview' -f -   # what K8s sees
kubectl auth can-i --list -n <ns>           # what you can do
```

### CloudWatch Logs Insights queries

**Find failed authentication attempts (audit log):**
```
fields @timestamp, user.username, verb, responseStatus.code, responseStatus.message
| filter responseStatus.code >= 400
| sort @timestamp desc
| limit 100
```

**API server errors (control plane log):**
```
fields @timestamp, @message
| filter @message like /error/
| sort @timestamp desc
| limit 50
```

---

## When All Else Fails

1. **Compare against a known-good cluster.** Spin up a fresh one with eksctl using the same version and diff manifests, add-on versions, and IAM.
2. **Check AWS Health Dashboard** for regional EKS incidents.
3. **File a support case** — include cluster ARN, timestamp, and the exact CLI/kubectl command output.
4. **EKS community Slack** and **StackOverflow `#amazon-eks`** are surprisingly responsive.

---

## Preventive Practices Checklist

| Practice | Why it matters |
|----------|----------------|
| Set resource `requests` **and** `limits` on every pod | Prevents noisy neighbors and OOM cascade |
| Use `PodDisruptionBudget` on stateful workloads | Survives node group upgrades |
| Use `topologySpreadConstraints` | Multi-AZ resilience |
| Enable control-plane logs from day one | Audit trail when things break |
| Tag every subnet with EKS discovery labels | ALB controller & Karpenter both need them |
| Practice cluster upgrades in staging first | Discover breaking changes safely |
| Regularly run **Upgrade Insights** even without upgrading | Catch deprecated APIs early |
| Keep add-ons on **latest recommended** versions | Avoid nasty version skew during upgrades |
| Automate cluster teardown scripts for labs | Prevent surprise bills |

---

Back to **[README.md](README.md)** • **[commands-cheatsheet.md](commands-cheatsheet.md)** • **[hands-on-labs.md](hands-on-labs.md)**
