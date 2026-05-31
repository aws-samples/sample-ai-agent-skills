---
title: "G1 — Cluster Autoscaler Issues"
description: "Diagnose and resolve Cluster Autoscaler scaling failures"
status: active
severity: HIGH
triggers:
  - "Cluster Autoscaler"
  - "not scaling up"
  - "not scaling down"
  - "scale-up"
  - "scale-down"
owner: devops-agent
objective: "Identify why Cluster Autoscaler is not scaling and fix the configuration"
context: "Cluster Autoscaler adjusts the desired count of ASGs (node groups) based on pending pods and node utilization. It scales up when pods are unschedulable due to insufficient resources, and scales down when nodes are underutilized. It requires IAM permissions to modify ASGs and must discover node groups via tags or explicit configuration."
---

## Phase 1 — Triage

MUST:
- Check Cluster Autoscaler pod status: `kubectl get pods -n kube-system -l app.kubernetes.io/name=cluster-autoscaler`
- Check Cluster Autoscaler logs: `kubectl logs -n kube-system -l app.kubernetes.io/name=cluster-autoscaler --tail=100`
- Check for pending pods: `kubectl get pods --all-namespaces --field-selector status.phase=Pending`
- Verify ASG tags: `aws autoscaling describe-auto-scaling-groups --query 'AutoScalingGroups[*].{Name:AutoScalingGroupName,Tags:Tags[?Key==\`k8s.io/cluster-autoscaler/enabled\`]}'`
- Check ASG min/max/desired: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>`

SHOULD:
- Check Cluster Autoscaler ConfigMap or deployment args for configuration
- Verify IAM permissions: needs `autoscaling:SetDesiredCapacity`, `autoscaling:TerminateInstanceInAutoScalingGroup`, `autoscaling:DescribeAutoScalingGroups`, etc.
- Check if scale-down is blocked by PodDisruptionBudgets or annotations
- Look for "scale-up" or "scale-down" entries in the autoscaler logs

MAY:
- Check the Cluster Autoscaler status ConfigMap: `kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml`
- Verify the autoscaler version matches the cluster Kubernetes version
- Check for node groups with `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` annotations

## Phase 2 — Remediate

MUST:
- If not discovering node groups: add ASG tags `k8s.io/cluster-autoscaler/enabled=true` and `k8s.io/cluster-autoscaler/<cluster-name>=owned`
- If IAM permissions missing: update the autoscaler's IAM role
- If ASG max reached: increase the ASG max size
- If scale-down blocked: check PDBs and pod annotations

SHOULD:
- Set `--balance-similar-node-groups=true` for multi-AZ clusters
- Set `--skip-nodes-with-system-pods=false` to allow scale-down of nodes with kube-system pods
- Match the Cluster Autoscaler version to the cluster Kubernetes version

MAY:
- Consider migrating to Karpenter for faster scaling and better bin-packing
- Configure expanders (random, most-pods, least-waste, priority) for scale-up decisions

## Common Issues

- symptoms: "Pods pending but Cluster Autoscaler doesn't scale up"
  diagnosis: "Autoscaler can't find a node group that fits the pending pods, or ASG max is reached."
  resolution: "Check autoscaler logs for 'could not find node group' messages. Increase ASG max or add larger instance types."

- symptoms: "Cluster Autoscaler scales up but new nodes don't become Ready"
  diagnosis: "Node bootstrap failure — separate from autoscaler issue."
  resolution: "See B1 (Node NotReady) and B2 (Managed Node Group Failures)."

- symptoms: "Nodes are underutilized but Cluster Autoscaler won't scale down"
  diagnosis: "Scale-down blocked by PDBs, local storage, or pod annotations."
  resolution: "Check for pods with `cluster-autoscaler.kubernetes.io/safe-to-evict: false`. Review PDBs."

## Output Format

```yaml
root_cause: "Cluster Autoscaler — <specific_cause>"
evidence:
  - type: autoscaler_logs
    content: "<relevant log entries>"
  - type: asg_config
    content: "<ASG min/max/desired and tags>"
severity: HIGH
mitigation:
  immediate: "Fix autoscaler configuration, IAM permissions, or ASG limits"
  long_term: "Implement autoscaler monitoring, right-size node groups, consider Karpenter"
```

## Safety Ratings
- GREEN: read-only (`kubectl get pods`, `kubectl logs`, `kubectl get pods --field-selector status.phase=Pending`, `aws autoscaling describe-auto-scaling-groups`)
- YELLOW: state-changing recoverable (`aws autoscaling create-or-update-tags` for ASG discovery, update autoscaler IAM role, adjust ASG max size, update autoscaler deployment args)
- RED: destructive/irreversible (`aws autoscaling set-desired-capacity` to zero in production, terminating instances via autoscaler misconfiguration)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Change affects node group scaling in production"
- "Fix involves modifying ASG scaling limits in production"
- "Remediation requires updating Cluster Autoscaler IAM permissions in production"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current ASG min/max/desired values before modification"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify autoscaler is discovering node groups and scaling correctly after change"
- Revert: "Restore ASG scaling parameters from backup if scaling behavior breaks"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- MEDIUM: "Node group configuration reveals instance types and scaling"
- MEDIUM: "ASG configuration reveals scaling limits and instance distribution"
- LOW: "Pod status and events"

## Prohibited Actions
- NEVER suggest adding `system:masters` group for troubleshooting access
- NEVER suggest disabling Pod Security Standards/Admission
- NEVER suggest running containers as root to fix permission issues
- NEVER suggest `kubectl delete namespace` in production without confirmation
- NEVER suggest modifying kube-system resources without backup

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
