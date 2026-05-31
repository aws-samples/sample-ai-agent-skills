---
title: "E1 — aws-auth ConfigMap Issues"
description: "Diagnose and resolve aws-auth ConfigMap misconfigurations that affect cluster access"
status: active
severity: CRITICAL
triggers:
  - "aws-auth"
  - "Unauthorized"
  - "locked out"
  - "cannot access cluster"
  - "mapRoles"
  - "mapUsers"
owner: devops-agent
objective: "Fix aws-auth ConfigMap to restore proper IAM-to-RBAC mapping"
context: "The aws-auth ConfigMap in kube-system maps IAM roles and users to Kubernetes RBAC groups. It controls who can access the cluster via kubectl. The cluster creator has implicit system:masters access and is NOT listed in aws-auth. Misconfiguration can lock all other users out."
---

## Phase 1 — Triage

MUST:
- Check current aws-auth ConfigMap: `kubectl get configmap aws-auth -n kube-system -o yaml`
- Verify your IAM identity: `aws sts get-caller-identity`
- Check if your IAM identity is in the aws-auth mapRoles or mapUsers
- If locked out: identify the cluster creator's IAM identity (they always have access)

SHOULD:
- Back up the current aws-auth: `kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-backup.yaml`
- Check CloudTrail for recent ConfigMap modifications
- Verify node IAM roles are in mapRoles with `system:bootstrappers` and `system:nodes` groups
- Check for YAML formatting errors in the ConfigMap

MAY:
- Check if EKS access entries are configured (newer alternative to aws-auth): `aws eks list-access-entries --cluster-name <cluster>`
- Use `eksctl get iamidentitymapping --cluster <cluster>` for eksctl-managed clusters
- Check authenticator logs in CloudWatch: `/aws/eks/<cluster>/cluster` (authenticator log type)

## Phase 2 — Remediate

MUST:
- If locked out: use the cluster creator's IAM identity to fix aws-auth
- If node roles missing: add the node IAM role to mapRoles:
  ```yaml
  - rolearn: arn:aws:iam::<account>:role/<node-role>
    username: system:node:{{EC2PrivateDNSName}}
    groups:
      - system:bootstrappers
      - system:nodes
  ```
- If user access missing: add the IAM role/user to mapRoles or mapUsers with appropriate groups
- Fix any YAML formatting errors (indentation, missing fields)

SHOULD:
- Use IAM roles (mapRoles) instead of IAM users (mapUsers) for better security
- Map roles to custom RBAC groups instead of system:masters for least privilege
- Consider migrating to EKS access entries for easier management

MAY:
- Use `eksctl create iamidentitymapping` for safer editing
- Set up emergency access procedures for aws-auth recovery

## Common Issues

- symptoms: "error: You must be logged in to the server (Unauthorized)"
  diagnosis: "IAM identity is not mapped in aws-auth ConfigMap."
  resolution: "Add the IAM role/user to aws-auth. Use the cluster creator's identity if locked out."

- symptoms: "Nodes show NotReady and cannot join the cluster"
  diagnosis: "Node IAM role is missing from aws-auth mapRoles."
  resolution: "Add the node role ARN with system:bootstrappers and system:nodes groups."

- symptoms: "aws-auth ConfigMap was accidentally deleted"
  diagnosis: "Without aws-auth, only the cluster creator can access the cluster."
  resolution: "Use the cluster creator's IAM identity to recreate aws-auth. Restore from backup if available."

- symptoms: "User can describe-cluster but kubectl returns Unauthorized"
  diagnosis: "IAM permissions (eks:DescribeCluster) and Kubernetes access (aws-auth) are separate layers."
  resolution: "Add the user's IAM identity to aws-auth ConfigMap."

## Output Format

```yaml
root_cause: "aws-auth ConfigMap — <specific_cause>"
evidence:
  - type: configmap
    content: "<aws-auth ConfigMap contents>"
  - type: iam_identity
    content: "<caller identity from sts>"
severity: CRITICAL
mitigation:
  immediate: "Fix aws-auth ConfigMap mapping"
  long_term: "Implement aws-auth backup procedures, consider EKS access entries, use IaC for aws-auth management"
```

## Safety Ratings
- GREEN: read-only (`kubectl get configmap aws-auth -n kube-system`, `aws sts get-caller-identity`, `aws eks list-access-entries`)
- YELLOW: state-changing recoverable (`kubectl apply` aws-auth with backup, `eksctl create iamidentitymapping`, adding new mapRoles entries)
- RED: destructive/irreversible (`kubectl edit configmap aws-auth` without backup, `kubectl delete configmap aws-auth`, removing mapRoles entries that could lock out users)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Remediation requires cluster upgrade"
- "Change affects node group scaling in production"
- "Fix involves modifying network policies"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing: `kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-backup.yaml`"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify cluster access for all mapped IAM identities after change"
- Revert: "Restore aws-auth from backup if locked out: use cluster creator credentials to apply backup"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- MEDIUM: "Node group configuration reveals instance types and scaling"
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
