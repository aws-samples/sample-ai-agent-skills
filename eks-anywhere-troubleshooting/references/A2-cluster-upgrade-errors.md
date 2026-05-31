---
title: "A2 — Cluster Upgrade Errors"
description: "Diagnose EKS Anywhere cluster upgrade failures"
status: active
severity: CRITICAL
triggers: ["cluster upgrade failed", "eksctl anywhere upgrade error", "upgrade stuck", "rolling update failed"]
owner: devops-agent
objective: "Resolve EKS Anywhere cluster upgrade issues including version skips and rolling update failures"
context: "EKS Anywhere upgrades are sequential (no version skipping). Upgrades use rolling replacement of control plane and worker nodes via CAPI."
---

## Phase 1 — Triage

MUST:
- Check current cluster version: `kubectl get clusters.anywhere.eks.amazonaws.com -A -o yaml --kubeconfig <mgmt-kubeconfig>`
- Check machine rollout status: `kubectl get machines -A -o wide --kubeconfig <mgmt-kubeconfig>`
- Review CAPI controller logs: `kubectl logs -n capi-system deployment/capi-controller-manager --tail=200 --kubeconfig <mgmt-kubeconfig>`

SHOULD:
- Verify target version is sequential (no minor version skip)
- Check KubeadmControlPlane rollout: `kubectl get kcp -A -o wide --kubeconfig <mgmt-kubeconfig>`
- Review MachineDeployment status: `kubectl get machinedeployments -A -o wide --kubeconfig <mgmt-kubeconfig>`

MAY:
- Check etcd cluster health on control plane nodes
- Review node drain events for stuck pods

## Phase 2 — Remediate

MUST:
- Ensure upgrade follows sequential version path (e.g., 1.27→1.28, not 1.27→1.29)
- Fix stuck machines by checking provider-specific provisioning issues
- Resolve PodDisruptionBudget conflicts blocking node drain

SHOULD:
- Increase upgrade timeout if nodes are slow to provision
- Verify new VM templates or OS images are available for the target version

MAY:
- Manually cordon and drain problematic nodes to unblock rolling update
- Scale down non-critical workloads to speed up node drain

## Safety Ratings

```
safety_ratings:
  - "kubectl get clusters/machines: GREEN — read-only status check"
  - "eksctl anywhere upgrade cluster: YELLOW — modifies cluster infrastructure, rolling replacement"
  - "kubectl cordon/drain: YELLOW — affects workload scheduling"
  - "kubectl logs: GREEN — read-only log retrieval"
```

## Escalation Conditions

- Upgrade stuck for more than 2 hours with no machine progress
- Control plane nodes fail to join after replacement
- etcd cluster loses quorum during upgrade
- Multiple worker nodes fail health checks after upgrade

## Data Sensitivity

- `kubectl get clusters/machines`: LOW — returns resource metadata
- `kubectl logs capi-controller-manager`: MEDIUM — may contain infrastructure details
- `kubectl get kcp -o yaml`: MEDIUM — contains control plane configuration
- `etcd health check`: LOW — returns cluster health status

## Prohibited Actions

- NEVER attempt to skip Kubernetes minor versions during upgrade
- NEVER delete CAPI machine resources manually during an active upgrade
- NEVER force-delete control plane machines — this can cause etcd quorum loss
- NEVER roll back by manually downgrading Kubernetes binaries on nodes

## Phase 3 — Rollback

1. If upgrade is stuck, check if rollback is possible: `kubectl get kcp -A -o yaml` to see rollout history
2. Revert the cluster spec to the previous version and re-apply: `eksctl anywhere upgrade cluster -f <previous-spec.yaml>`
3. If machines are stuck in provisioning, delete the stuck Machine resource (worker only): `kubectl delete machine <name> -n <namespace>`
4. Verify control plane health after rollback: `kubectl get nodes --kubeconfig <workload-kubeconfig>`
5. Document the failed upgrade attempt and root cause for retry planning

## Common Issues

- symptoms: "Upgrade rejected — cannot skip minor version"
  diagnosis: "Attempted to skip a Kubernetes minor version."
  resolution: "Upgrade sequentially through each minor version."

## Output Format

```yaml
root_cause: "cluster_upgrade — <specific_cause>"
evidence:
  - type: upgrade_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Unblock upgrade or rollback"
  long_term: "Test upgrades in staging first"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-cluster"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get configmap aws-auth"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get pods"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest system:masters group for troubleshooting"
  - "NEVER suggest disabling RBAC or Pod Security Standards"
  - "NEVER suggest running pods as root to fix permissions"
