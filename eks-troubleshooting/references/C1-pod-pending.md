---
title: "C1 — Pod Pending"
description: "Diagnose and resolve pods stuck in Pending state"
status: active
severity: HIGH
triggers:
  - "Pending"
  - "FailedScheduling"
  - "Unschedulable"
  - "insufficient"
  - "taints"
owner: devops-agent
objective: "Identify why the pod cannot be scheduled and resolve the scheduling constraint"
context: "Pods enter Pending when the scheduler cannot find a suitable node. Common causes: insufficient resources (CPU/memory), node taints without matching tolerations, node selectors or affinity rules that don't match any node, IP exhaustion (VPC CNI), or no nodes available."
---

## Phase 1 — Triage

MUST:
- Check pod status and events: `kubectl describe pod <pod> -n <namespace>`
- Look for FailedScheduling events with the specific reason
- Check node resources: `kubectl describe nodes | grep -A5 "Allocated resources"`
- Check pod resource requests: `kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[*].resources}'`
- Check for taints on nodes: `kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints`

SHOULD:
- Check if the pod has nodeSelector or nodeAffinity: `kubectl get pod <pod> -n <namespace> -o yaml | grep -A5 'nodeSelector\|affinity'`
- Check for PodDisruptionBudgets that might block scheduling
- Verify VPC CNI IP availability: `kubectl get pods -n kube-system -l k8s-app=aws-node -o wide`
- Check if the pod matches a Fargate profile (if using Fargate)

MAY:
- Check scheduler logs in CloudWatch: `/aws/eks/<cluster>/cluster` (scheduler log type)
- Check for resource quotas in the namespace: `kubectl get resourcequota -n <namespace>`
- Check for LimitRanges: `kubectl get limitrange -n <namespace>`

## Phase 2 — Remediate

MUST:
- If insufficient resources: scale up the node group or reduce resource requests
- If taint mismatch: add the correct toleration to the pod spec or remove the taint
- If nodeSelector mismatch: fix the selector or label the appropriate nodes
- If IP exhaustion: add more subnet capacity or enable VPC CNI prefix delegation

SHOULD:
- Review resource requests vs actual usage — requests may be over-provisioned
- Use pod topology spread constraints for better distribution
- Consider using Karpenter or Cluster Autoscaler for automatic scaling

MAY:
- Use pod priority and preemption to schedule critical pods first
- Adjust resource quotas if they're blocking scheduling

## Common Issues

- symptoms: "0/N nodes are available: N Insufficient cpu"
  diagnosis: "No node has enough allocatable CPU for the pod's resource request."
  resolution: "Scale up nodes, reduce CPU requests, or add larger instance types."

- symptoms: "0/N nodes are available: N node(s) had taint {key: value}, that the pod didn't tolerate"
  diagnosis: "All nodes have a taint that the pod doesn't tolerate."
  resolution: "Add the matching toleration to the pod spec, or remove the taint from nodes."

- symptoms: "0/N nodes are available: N node(s) didn't match Pod's node affinity/selector"
  diagnosis: "Pod's nodeSelector or nodeAffinity doesn't match any node labels."
  resolution: "Fix the selector/affinity or label nodes appropriately."

- symptoms: "Failed to assign an IP address to the pod"
  diagnosis: "VPC CNI cannot allocate an IP — subnet exhaustion or ENI limit reached."
  resolution: "Check subnet available IPs. Enable prefix delegation or add larger subnets."

## Output Format

```yaml
root_cause: "Pod Pending — <specific_cause>"
evidence:
  - type: pod_events
    content: "<FailedScheduling event details>"
  - type: node_resources
    content: "<node allocatable vs allocated>"
severity: HIGH
mitigation:
  immediate: "Resolve the scheduling constraint"
  long_term: "Implement autoscaling and right-size resource requests"
```

## Safety Ratings
GREEN — Triage uses read-only commands (`kubectl describe pod`, `kubectl get nodes`, `kubectl describe nodes`). Remediation involves adjusting resource requests, adding tolerations, or labeling nodes — non-destructive changes that don't affect other running workloads.

## Escalation Conditions
- Remediation requires editing aws-auth ConfigMap in production
- Fix involves modifying RBAC ClusterRoleBindings
- Node group changes affect running production workloads
- Cluster endpoint access changes could lock out users
- IRSA trust policy modifications are needed

## Rollback
1. Before aws-auth changes: `kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-backup.yaml`
2. Before RBAC changes: Save current bindings with `kubectl get clusterrolebinding <name> -o yaml`
3. After change: Verify cluster access and workload health
4. If change causes lockout: Use cluster creator credentials or restore from backup
5. Cleanup: Remove any temporary RBAC bindings or test service accounts

## Data Sensitivity
| Command | Sensitivity | Handling |
|---------|------------|----------|
| `kubectl get configmap aws-auth` | HIGH | IAM-to-RBAC mappings — redact |
| `kubectl get sa -A -o yaml` | HIGH | Service account annotations with role ARNs — redact |
| `kubectl get events` | MEDIUM | Cluster events — summarize |
| `kubectl get nodes/pods` | LOW | Resource status — safe to include |

## Prohibited Actions
- NEVER suggest adding `system:masters` group for troubleshooting access
- NEVER suggest disabling Pod Security Standards/Admission
- NEVER suggest running containers as root to fix permission issues
- NEVER suggest `--force` delete of namespaces or persistent volumes
- NEVER suggest modifying kube-system resources without backup
- ALWAYS backup aws-auth before any modification
- ALWAYS use specific RBAC roles instead of cluster-admin

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
