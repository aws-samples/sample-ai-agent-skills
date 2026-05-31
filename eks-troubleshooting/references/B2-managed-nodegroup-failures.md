---
title: "B2 — Managed Node Group Failures"
description: "Diagnose and resolve EKS managed node group creation, update, and health issues"
status: active
severity: HIGH
triggers:
  - "node group failed"
  - "node group degraded"
  - "CREATE_FAILED"
  - "launch template"
  - "node group health"
owner: devops-agent
objective: "Identify why the managed node group is failing and restore it to healthy state"
context: "Managed node groups use Auto Scaling Groups with AWS-managed launch templates. AWS handles AMI selection, node bootstrapping, and graceful updates. Failures typically stem from IAM role issues, launch template errors, subnet capacity, or instance type availability."
---

## Phase 1 — Triage

MUST:
- Check node group status and health: `aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng>`
- Look at the health field for specific issue codes and messages
- Check the node group's scaling config: `aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> --query 'nodegroup.scalingConfig'`
- Verify the node IAM role: `aws iam get-role --role-name <node-role>` and check attached policies

SHOULD:
- Check the underlying ASG: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>`
- Check ASG activity history: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --max-items 10`
- Verify the launch template: `aws ec2 describe-launch-template-versions --launch-template-id <lt-id>`
- Check subnet capacity for the node group's subnets

MAY:
- Check CloudFormation stack events if created via eksctl
- Verify instance type availability in the target AZs
- Check for EC2 service limits

## Phase 2 — Remediate

MUST:
- If IAM role issue: ensure the node role has `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, and `AmazonEC2ContainerRegistryReadOnly`
- If launch template error: fix the template or let EKS manage it (remove custom template)
- If instance capacity issue: add more instance types to the node group or use different AZs
- If health issue persists: delete and recreate the node group

SHOULD:
- For updates: use rolling update strategy with appropriate maxUnavailable setting
- Ensure the node group AMI matches the cluster Kubernetes version
- Use multiple instance types for better availability

MAY:
- Use `aws eks update-nodegroup-config` to adjust scaling parameters
- Switch from custom AMI to EKS-optimized AMI for easier management

## Common Issues

- symptoms: "Node group status is CREATE_FAILED"
  diagnosis: "IAM role missing required policies or subnet/security group misconfiguration."
  resolution: "Check the health issues field. Fix IAM role or VPC configuration."

- symptoms: "Node group is DEGRADED with 'Ec2SubnetInvalidConfiguration'"
  diagnosis: "Subnets don't have enough IPs or are in unsupported AZs."
  resolution: "Use subnets with available IPs in supported AZs."

- symptoms: "Nodes join but immediately go NotReady"
  diagnosis: "Bootstrap script failure or VPC CNI cannot assign IPs."
  resolution: "Check kubelet logs on the node. Verify subnet IP availability."

- symptoms: "Node group update stuck in progress"
  diagnosis: "Pods cannot be drained from old nodes (PDB blocking or stuck finalizers)."
  resolution: "Check PodDisruptionBudgets. Manually drain stuck nodes if needed."

## Output Format

```yaml
root_cause: "Managed node group failure — <specific_cause>"
evidence:
  - type: nodegroup_health
    content: "<describe-nodegroup health issues>"
  - type: asg_activity
    content: "<ASG scaling activity errors>"
severity: HIGH
mitigation:
  immediate: "Fix the identified configuration issue"
  long_term: "Use IaC with validated node group configurations and multiple instance types"
```

## Safety Ratings
YELLOW — Triage is read-only (`aws eks describe-nodegroup`, `aws autoscaling describe-*`). Remediation involves node group configuration updates, IAM role changes, and potentially deleting/recreating node groups which affects running workloads but through managed operations.

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
