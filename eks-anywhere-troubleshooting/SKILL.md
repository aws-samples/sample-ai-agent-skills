---
name: eks-anywhere-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot EKS Anywhere
  problems by analyzing cluster creation, upgrades, node provisioning,
  machine health checks, provider issues, networking, packages,
  GitOps, cluster management, authentication, backup/restore,
  and following structured runbooks.
  Activate when: cluster creation failures, upgrade errors, node
  provisioning issues, vSphere or bare metal problems, Cilium
  networking issues, load balancer failures, curated package errors,
  Flux GitOps problems, or the user says something is wrong with
  EKS Anywhere.
compatibility: >
  Requires eksctl-anywhere CLI, kubectl access to management and
  workload clusters, and provider-specific credentials (vSphere,
  bare metal IPMI/BMC, CloudStack, Nutanix). AWS CLI needed for
  curated packages and IAM Authenticator.
---

# EKS Anywhere Diagnostics

## When to use

Any EKS Anywhere investigation — cluster lifecycle, node provisioning, infrastructure provider issues, networking, curated packages, GitOps, cluster management service, IAM authentication, or backup/restore operations.

## Investigation workflow

### Step 1 — Collect and triage

```
eksctl anywhere version
kubectl get clusters -A --kubeconfig <mgmt-kubeconfig>
kubectl get machines -A --kubeconfig <mgmt-kubeconfig> -o wide
```

### Step 2 — Domain deep dive

```
kubectl get clusters.anywhere.eks.amazonaws.com -A -o yaml --kubeconfig <mgmt-kubeconfig>
kubectl logs -n eksa-system deployment/eksa-controller-manager --tail=200 --kubeconfig <mgmt-kubeconfig>
kubectl get machinehealthchecks -A --kubeconfig <mgmt-kubeconfig>
```

### Step 3 — Detailed investigation

```
kubectl get events -A --sort-by='.lastTimestamp' --kubeconfig <mgmt-kubeconfig> | tail -50
eksctl anywhere generate support-bundle --cluster <cluster-name> --kubeconfig <mgmt-kubeconfig>
kubectl get helmreleases -A --kubeconfig <workload-kubeconfig>
```

Read `references/guardrails.md` before concluding on any EKS Anywhere issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `eksctl anywhere create cluster` | Create new EKS Anywhere cluster |
| `eksctl anywhere upgrade cluster` | Upgrade cluster version |
| `eksctl anywhere generate support-bundle` | Collect diagnostics bundle |
| `kubectl get clusters.anywhere` | List EKS Anywhere cluster resources |
| `kubectl get machines` | Check CAPI machine status |
| `kubectl get machinehealthchecks` | Review machine health checks |
| `kubectl get helmreleases` | Check curated package status |

## Gotchas: EKS Anywhere

- EKS Anywhere uses Cluster API (CAPI) under the hood. Machine, MachineDeployment, and KubeadmControlPlane resources are CAPI objects, not EKS Anywhere-specific.
- The management cluster runs CAPI controllers that manage workload clusters. Losing the management cluster means losing the ability to manage workload clusters.
- vSphere provider requires specific permissions on vCenter — template VMs, resource pools, folders, and networks must be pre-configured.
- Bare metal provider uses Tinkerbell for provisioning. BMC/IPMI credentials and hardware inventory must be accurate.
- Cilium is the default CNI. Replacing it is NOT supported. Cilium policy enforcement mode affects network behavior.
- Curated packages require an AWS account and ECR token. Packages are pulled from ECR and require periodic token refresh.
- Flux GitOps integration manages cluster configuration declaratively. Manual changes to Flux-managed resources will be reverted.

## Anti-hallucination rules

1. Always cite specific cluster names, machine names, or controller logs as evidence.
2. Management cluster and workload cluster are separate entities. Never conflate their kubeconfigs.
3. CAPI resources (Machine, MachineDeployment) are different from EKS Anywhere resources (Cluster, VSphereDatacenterConfig). Never mix them.
4. EKS Anywhere is NOT EKS. Do not suggest EKS-specific APIs (eks:DescribeCluster) for EKS Anywhere clusters.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cluster Lifecycle | A1-A2 | Creation failures, upgrade errors |
| B — Node Management | B1-B2 | Node provisioning, machine health checks |
| C — Providers | C1-C2 | vSphere provider, bare metal provider |
| D — Networking | D1-D2 | Cilium CNI, load balancer issues |
| E — Packages & GitOps | E1-E2 | Curated packages, Flux GitOps |
| F — Management & Auth | F1-F2 | Cluster management service, IAM authenticator |
| G — Backup & Restore | G1 | Backup and restore |
| Z — Catch-All | Z1 | General troubleshooting |
