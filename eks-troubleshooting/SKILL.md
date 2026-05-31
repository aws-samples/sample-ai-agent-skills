---
name: eks-diagnostics
description: >
  Use this skill to investigate and troubleshoot Amazon EKS problems
  by analyzing cluster configurations, node health, pod scheduling,
  networking, IAM/RBAC, storage, scaling, add-ons, and following
  structured runbooks. Activate when: cluster creation failures,
  node not ready, pod pending or CrashLoopBackOff, OOMKilled,
  image pull errors, VPC CNI issues, CoreDNS failures, service
  connectivity problems, ingress/ALB issues, IRSA failures,
  aws-auth ConfigMap misconfiguration, RBAC denied, pod identity
  issues, EBS/EFS CSI driver problems, PVC pending, cluster
  autoscaler or Karpenter issues, HPA not scaling, add-on failures,
  version compatibility, cluster upgrades, or the user says
  something is wrong with EKS without naming specific symptoms.
compatibility: >
  Requires AWS CLI, kubectl, eksctl, CloudWatch (Container Insights),
  and CloudTrail permissions. Optional: Helm, Karpenter CLI.
---

# EKS Diagnostics

## When to use

Any EKS investigation where the console alone is insufficient — cluster lifecycle issues, node health, pod scheduling failures, networking problems, IAM/RBAC misconfigurations, storage driver issues, autoscaling failures, add-on management, or upgrade planning.

## Investigation workflow

### Step 1 — Collect and triage

```
# Cluster status, version, endpoint, logging configuration
aws eks describe-cluster --name <cluster> --query 'cluster.{status:status,version:version,endpoint:endpoint,logging:logging,platformVersion:platformVersion}'

# Node health and readiness
kubectl get nodes -o wide

# Pods not in Running state across all namespaces
kubectl get pods --all-namespaces --field-selector status.phase!=Running

# Node conditions and resource pressure
kubectl describe node <node-name>

# Recent cluster events sorted by timestamp
kubectl get events --sort-by='.lastTimestamp' -A | tail -50
```

Triage returns:
- Cluster state, Kubernetes version, platform version
- Node readiness, roles, instance types, AZs
- Failing or pending pods with namespace context
- Node conditions (MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable)
- Recent events indicating scheduling, networking, or resource issues

If the cluster API server is unreachable, that IS the root cause domain. Don't chase pod-level symptoms.

### Step 2 — Domain deep dive (only if needed)

```
# Container logs (current and previous crash)
kubectl logs <pod-name> -n <namespace> --previous
kubectl logs <pod-name> -n <namespace> -c <container>

# Pod details — events, conditions, resource requests/limits
kubectl describe pod <pod-name> -n <namespace>

# Managed node group health and scaling configuration
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <nodegroup>

# Installed add-ons and their status
aws eks list-addons --cluster-name <cluster>
aws eks describe-addon --cluster-name <cluster> --addon-name <addon>

# CloudWatch Container Insights (if enabled)
aws logs filter-log-events --log-group-name /aws/containerinsights/<cluster>/performance --start-time <epoch>

# Control plane logs
aws logs filter-log-events --log-group-name /aws/eks/<cluster>/cluster --start-time <epoch>
```

Read `references/eks-guardrails.md` before concluding on any EKS issue.

### Step 3 — Detailed investigation (low-confidence cases only)

```
# CloudTrail for EKS API events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=eks.amazonaws.com --max-results 20

# VPC flow logs for networking issues
aws logs filter-log-events --log-group-name <vpc-flow-log-group> --filter-pattern "REJECT"

# IAM-to-RBAC mapping
kubectl get configmap aws-auth -n kube-system -o yaml

# Service account inventory (IRSA, pod identity)
kubectl get sa -A -o yaml | grep -A2 'annotations'

# Endpoint health
kubectl get endpoints -A

# OIDC provider for IRSA
aws eks describe-cluster --name <cluster> --query 'cluster.identity.oidc.issuer'
aws iam list-open-id-connect-providers
```

## Tool quick reference

| Tool / Command | When to use |
|----------------|-------------|
| `aws eks describe-cluster` | Cluster status, version, endpoint, logging, OIDC |
| `aws eks describe-nodegroup` | Managed node group health, scaling config, AMI |
| `aws eks describe-addon` | Add-on status, version, configuration conflicts |
| `aws eks list-updates` | Pending or failed cluster/add-on/nodegroup updates |
| `kubectl get nodes -o wide` | Node status, roles, version, instance type, AZ |
| `kubectl get pods -A` | Pod status across all namespaces |
| `kubectl describe node/pod` | Detailed resource info, events, conditions |
| `kubectl logs` | Container logs, previous crash logs, init container logs |
| `kubectl get events -A` | Cluster-wide event stream |
| `kubectl get configmap aws-auth -n kube-system` | IAM-to-RBAC mapping |
| `kubectl get sa -A` | Service accounts (IRSA, pod identity) |
| `eksctl utils describe-stacks` | CloudFormation stack status for eksctl-managed clusters |
| `eksctl get nodegroup` | Node group summary for eksctl clusters |
| CloudWatch Container Insights | Pod/node/cluster metrics and logs |

## Gotchas: EKS

These are the mistakes commonly made during EKS troubleshooting.

- The aws-auth ConfigMap in kube-system is the bridge between IAM and Kubernetes RBAC. Misconfiguration locks you out of the cluster entirely. Always back up before editing: `kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-backup.yaml`.
- IRSA (IAM Roles for Service Accounts) requires three things: (1) OIDC provider associated with the cluster, (2) service account annotated with `eks.amazonaws.com/role-arn`, (3) IAM role trust policy allowing the OIDC provider. Missing any one causes silent permission failures.
- VPC CNI plugin assigns pod IPs from VPC subnets. Each ENI has a limited number of secondary IPs based on instance type. IP exhaustion is common in large clusters — pods go Pending with "failed to assign an IP address" errors.
- CoreDNS runs as pods on worker nodes, not as a managed service. If nodes are unhealthy, CoreDNS pods fail, and all DNS resolution in the cluster breaks. This cascades to every pod that resolves service names.
- AWS Load Balancer Controller (formerly ALB Ingress Controller) needs specific IAM permissions AND subnet tags (`kubernetes.io/role/elb` for public, `kubernetes.io/role/internal-elb` for private). Missing tags = no ALB created, no error in the Ingress resource.
- EBS CSI driver is a separate add-on required for EBS volumes since EKS 1.23. The in-tree provisioner is deprecated. Without the add-on, PVCs using `gp2`/`gp3` StorageClasses stay Pending.
- Cluster endpoint access has two toggles: public and private. Public-only means kubectl works from anywhere but nodes use public internet. Private-only means kubectl only works from within the VPC. Public+private is the recommended default.
- Managed node groups, self-managed nodes, and Fargate have different troubleshooting paths. Managed node groups: check ASG + launch template. Self-managed: you own everything. Fargate: check Fargate profile selectors and namespace/label matching.
- kubectl context and kubeconfig issues are the #1 "it's not working" cause. Always verify: `kubectl config current-context` and refresh with `aws eks update-kubeconfig --name <cluster> --region <region>`.
- Pod Security Standards (PSS) replaced PodSecurityPolicy (PSP) in Kubernetes 1.25. PSP resources are silently ignored in 1.25+. Use Pod Security Admission (PSA) labels on namespaces instead.
- Karpenter and Cluster Autoscaler are mutually exclusive scaling solutions with different models. Cluster Autoscaler works with ASGs and node groups. Karpenter provisions nodes directly via EC2 fleet API, bypassing ASGs entirely.
- Node NotReady usually means kubelet or CNI plugin failure. Check `kubectl describe node` for conditions and `journalctl -u kubelet` on the node. Common causes: kubelet cannot reach API server, VPC CNI crash, disk pressure, or certificate expiry.
- EKS control plane is managed by AWS. You have NO direct access to etcd, kube-apiserver, or kube-scheduler. Troubleshoot via API server logs in CloudWatch (5 log types: api, audit, authenticator, controllerManager, scheduler).
- Max pods per node depends on the instance type's ENI limits: `(number_of_ENIs × (IPs_per_ENI - 1)) + 2`. Exceeding this causes pod scheduling failures with "Too many pods" message.
- Fargate profiles use namespace and label selectors. DaemonSets, privileged containers, and HostNetwork are NOT supported on Fargate. Fargate pods get their own microVM — no node-level access.
- Service account token volume projection is enabled by default. Tokens are audience-bound and time-limited (default 1 hour, auto-refreshed by kubelet). Old-style non-expiring tokens are deprecated.

## Anti-hallucination rules

1. Always cite specific kubectl output, AWS CLI responses, or log entries as evidence.
2. Never claim direct access to etcd, kube-apiserver config, or control plane components — EKS control plane is fully managed by AWS.
3. Never suggest running DaemonSets or privileged containers on Fargate — they are not supported.
4. Never claim cluster upgrades can skip minor versions — upgrades must go one minor version at a time (e.g., 1.27 → 1.28, not 1.27 → 1.29).
5. Never assume aws-auth ConfigMap can be recovered without cluster access — if you lock yourself out, you need the cluster creator's IAM identity or AWS support.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 34 runbooks

Runbooks are organized by failure domain. Use the appropriate runbook based on the symptom category.

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cluster | A1–A3 | Cluster creation failures, endpoint access issues, cluster upgrade failures |
| B — Nodes | B1–B4 | Node NotReady, managed node group failures, Fargate profile issues, node scaling problems |
| C — Pods | C1–C4 | Pending pods, CrashLoopBackOff, OOMKilled, image pull errors |
| D — Networking | D1–D4 | VPC CNI / IP exhaustion, CoreDNS failures, service connectivity, ingress/ALB issues |
| E — IAM & RBAC | E1–E4 | aws-auth ConfigMap issues, IRSA failures, RBAC denied, pod identity problems |
| F — Storage | F1–F3 | EBS CSI driver issues, EFS CSI driver issues, PVC pending |
| G — Scaling | G1–G3 | Cluster Autoscaler issues, Karpenter issues, HPA not scaling |
| H — Add-ons | H1–H3 | Add-on installation/update failures, version compatibility, custom add-ons |
| I — Observability | I1–I2 | Container Insights setup/issues, control plane and application logging |
| Z — Catch-All | Z1 | General EKS troubleshooting |
