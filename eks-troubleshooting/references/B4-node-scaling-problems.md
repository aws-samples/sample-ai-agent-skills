---
title: "B4 — Node Scaling Problems"
description: "Diagnose and resolve EKS node group scaling failures (ASG-level)"
status: active
severity: HIGH
triggers:
  - "nodes not scaling"
  - "ASG not scaling"
  - "desired capacity"
  - "max size reached"
owner: devops-agent
objective: "Identify why the node group is not scaling and restore scaling behavior"
context: "Managed node groups use ASGs for scaling. Scaling can fail due to ASG limits, instance capacity, launch template errors, or subnet IP exhaustion. This runbook covers ASG-level scaling issues. For Cluster Autoscaler or Karpenter issues, see G1/G2."
---

## Phase 1 — Triage

MUST:
- Check node group scaling config: `aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> --query 'nodegroup.scalingConfig'`
- Check ASG current state: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[0].{Min:MinSize,Max:MaxSize,Desired:DesiredCapacity,Instances:Instances[*].{Id:InstanceId,State:LifecycleState}}'`
- Check ASG scaling activities: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --max-items 10`
- Look for failed launch attempts in the activity history

SHOULD:
- Check instance type availability in the target AZs
- Verify subnet IP availability for new nodes
- Check EC2 service quotas: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A`
- Check if scaling policies or scheduled actions are interfering

MAY:
- Check CloudTrail for RunInstances failures
- Verify the launch template is valid: `aws ec2 describe-launch-template-versions --launch-template-id <lt-id>`

## Phase 2 — Remediate

MUST:
- If max size reached: increase the node group max size: `aws eks update-nodegroup-config --cluster-name <cluster> --nodegroup-name <ng> --scaling-config maxSize=<new-max>`
- If instance capacity issue: add more instance types or AZs to the node group
- If subnet IP exhaustion: use larger subnets or add more subnets
- If launch template error: fix the template configuration

SHOULD:
- Use multiple instance types for better availability
- Spread across multiple AZs
- Set appropriate min/max/desired values

MAY:
- Create additional node groups for different workload types
- Consider Karpenter for more flexible scaling

## Common Issues

- symptoms: "ASG desired count increases but instances don't launch"
  diagnosis: "InsufficientInstanceCapacity or launch template error."
  resolution: "Check ASG activity for error messages. Add more instance types or AZs."

- symptoms: "Node group max size reached but more capacity needed"
  diagnosis: "Scaling config maxSize is too low."
  resolution: "Increase maxSize via update-nodegroup-config."

- symptoms: "New nodes launch but don't join the cluster"
  diagnosis: "Bootstrap failure — node can't reach API server or aws-auth doesn't include the node role."
  resolution: "Check node security groups, aws-auth ConfigMap, and bootstrap logs."

## Output Format

```yaml
root_cause: "Node scaling problem — <specific_cause>"
evidence:
  - type: asg_activity
    content: "<scaling activity errors>"
  - type: scaling_config
    content: "<current min/max/desired>"
severity: HIGH
mitigation:
  immediate: "Fix the scaling constraint (max size, instance type, subnet capacity)"
  long_term: "Use multiple instance types, AZs, and appropriate scaling limits"
```

## Safety Ratings
GREEN — Triage uses read-only commands (`aws eks describe-nodegroup`, `aws autoscaling describe-*`). Remediation involves adjusting ASG scaling parameters (`update-nodegroup-config maxSize`) which is a safe, non-disruptive configuration change.

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
