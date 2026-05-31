---
title: "G2 — Karpenter Issues"
description: "Diagnose and resolve Karpenter node provisioning and scaling problems"
status: active
severity: HIGH
triggers:
  - "Karpenter"
  - "NodePool"
  - "NodeClaim"
  - "provisioner"
  - "node provisioning"
owner: devops-agent
objective: "Identify why Karpenter is not provisioning nodes and fix the configuration"
context: "Karpenter provisions nodes directly via EC2 fleet API, bypassing ASGs. It uses NodePool (formerly Provisioner) and EC2NodeClass (formerly AWSNodeTemplate) CRDs. Karpenter is faster than Cluster Autoscaler and provides better bin-packing. It must NOT run alongside Cluster Autoscaler."
---

## Phase 1 — Triage

MUST:
- Check Karpenter controller pod: `kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter`
- Check Karpenter logs: `kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter --tail=100`
- Check NodePool resources: `kubectl get nodepools`
- Check EC2NodeClass resources: `kubectl get ec2nodeclasses`
- Check NodeClaim status: `kubectl get nodeclaims`

SHOULD:
- Describe the NodePool for limits and requirements: `kubectl describe nodepool <name>`
- Check Karpenter's IAM role permissions (needs EC2 fleet, pricing, SSM permissions)
- Verify the EC2NodeClass references correct subnets and security groups
- Check for pending pods that should trigger provisioning

MAY:
- Check Karpenter metrics: `kubectl get --raw /metrics | grep karpenter`
- Verify instance type availability in the configured AZs
- Check for Karpenter disruption budgets

## Phase 2 — Remediate

MUST:
- If controller not running: check deployment, IRSA, and IAM permissions
- If NodePool missing or misconfigured: create/fix the NodePool with appropriate requirements
- If EC2NodeClass wrong: fix subnet selector, security group selector, and AMI configuration
- If IAM permissions insufficient: update the Karpenter controller role and node role

SHOULD:
- Set NodePool limits (CPU, memory) to prevent runaway scaling
- Configure consolidation for cost optimization
- Use multiple instance types and AZs for availability

MAY:
- Set up Karpenter disruption budgets to control node replacement
- Configure custom AMIs via EC2NodeClass
- Use NodePool weights for priority-based provisioning

## Common Issues

- symptoms: "Pods pending but Karpenter doesn't provision nodes"
  diagnosis: "NodePool requirements don't match pod requirements, or limits are reached."
  resolution: "Check NodePool requirements (instance types, AZs, architecture). Check limits."

- symptoms: "Karpenter provisions nodes but they don't join the cluster"
  diagnosis: "Node bootstrap failure — AMI, user data, or security group issue."
  resolution: "Check EC2NodeClass AMI and security group configuration. Verify aws-auth includes the Karpenter node role."

- symptoms: "Karpenter logs show 'insufficient capacity' errors"
  diagnosis: "EC2 capacity unavailable for the requested instance types in the target AZs."
  resolution: "Add more instance types and AZs to the NodePool requirements."

- symptoms: "Karpenter keeps terminating and reprovisioning nodes"
  diagnosis: "Consolidation is too aggressive or disruption budgets are not set."
  resolution: "Configure consolidation policy and disruption budgets appropriately."

## Output Format

```yaml
root_cause: "Karpenter issue — <specific_cause>"
evidence:
  - type: karpenter_logs
    content: "<relevant controller log entries>"
  - type: nodepool_config
    content: "<NodePool and EC2NodeClass configuration>"
severity: HIGH
mitigation:
  immediate: "Fix Karpenter configuration, IAM permissions, or NodePool requirements"
  long_term: "Implement Karpenter monitoring, set appropriate limits and disruption budgets"
```

## Safety Ratings
- GREEN: read-only (`kubectl get nodepools`, `kubectl get ec2nodeclasses`, `kubectl get nodeclaims`, `kubectl logs`, `kubectl describe nodepool`)
- YELLOW: state-changing recoverable (`kubectl apply` NodePool/EC2NodeClass updates, update Karpenter IAM role, configure consolidation policy, set disruption budgets)
- RED: destructive/irreversible (`kubectl delete nodepool` in production causing node termination, `kubectl delete nodeclaim` terminating production nodes, misconfiguring consolidation causing mass node replacement)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Change affects node group scaling in production"
- "Fix involves modifying Karpenter NodePool limits or requirements in production"
- "Remediation requires updating Karpenter IAM permissions in production"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current NodePool and EC2NodeClass configuration before modification"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify Karpenter is provisioning nodes correctly after change"
- Revert: "Restore NodePool/EC2NodeClass from backup if provisioning breaks"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- MEDIUM: "Node group configuration reveals instance types and scaling"
- MEDIUM: "EC2NodeClass reveals subnet selectors, security groups, and AMI configuration"
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
