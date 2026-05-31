---
title: "E3 — RBAC Denied"
description: "Diagnose and resolve Kubernetes RBAC permission denied errors"
status: active
severity: HIGH
triggers:
  - "forbidden"
  - "RBAC"
  - "cannot list"
  - "cannot get"
  - "cannot create"
  - "User cannot"
owner: devops-agent
objective: "Identify the RBAC permission gap and grant appropriate access"
context: "Kubernetes RBAC controls what actions users and service accounts can perform. RBAC uses Roles/ClusterRoles (define permissions) and RoleBindings/ClusterRoleBindings (assign permissions to subjects). EKS maps IAM identities to Kubernetes users/groups via aws-auth ConfigMap."
---

## Phase 1 — Triage

MUST:
- Identify the exact error message and the resource/verb being denied
- Check who you are in Kubernetes: `kubectl auth whoami` (1.27+) or check aws-auth mapping
- Test specific permissions: `kubectl auth can-i <verb> <resource> -n <namespace>`
- List all permissions for the current user: `kubectl auth can-i --list -n <namespace>`
- Check RoleBindings in the namespace: `kubectl get rolebindings -n <namespace> -o wide`

SHOULD:
- Check ClusterRoleBindings: `kubectl get clusterrolebindings -o wide | grep <username-or-group>`
- Check the Role/ClusterRole referenced by the binding: `kubectl describe role <role> -n <namespace>` or `kubectl describe clusterrole <role>`
- Verify the aws-auth ConfigMap maps your IAM identity to the correct Kubernetes group
- Check if the user is in the correct group: compare aws-auth groups with RoleBinding subjects

MAY:
- Check audit logs in CloudWatch for the denied request details
- Use `kubectl auth can-i --as=<user>` to test permissions for other users (requires impersonation rights)
- Check for aggregated ClusterRoles that might affect permissions

## Phase 2 — Remediate

MUST:
- If missing RoleBinding: create one binding the user/group to the appropriate Role
- If Role lacks permissions: add the required verbs and resources to the Role
- If aws-auth group mismatch: update aws-auth to map the IAM identity to the correct Kubernetes group
- Use least privilege — grant only the specific permissions needed

SHOULD:
- Use ClusterRoles with RoleBindings for reusable permission sets scoped to namespaces
- Use built-in ClusterRoles where appropriate: `view`, `edit`, `admin`, `cluster-admin`
- Document RBAC policies and review them periodically

MAY:
- Use RBAC aggregation for modular permission management
- Implement namespace-level admin delegation
- Set up RBAC audit logging to track permission usage

## Common Issues

- symptoms: "User 'system:anonymous' cannot list resource 'pods'"
  diagnosis: "Request is not authenticated — likely kubeconfig or token issue."
  resolution: "Fix kubeconfig: `aws eks update-kubeconfig --name <cluster>`. Check aws-auth."

- symptoms: "User '<iam-arn>' cannot get resource 'nodes'"
  diagnosis: "IAM identity is authenticated but lacks RBAC permissions for the requested resource."
  resolution: "Create a ClusterRoleBinding granting the user's group access to the required resources."

- symptoms: "Service account cannot list secrets in namespace"
  diagnosis: "The service account's Role doesn't include 'secrets' resource with 'list' verb."
  resolution: "Update the Role to include the required resource and verb."

## Output Format

```yaml
root_cause: "RBAC denied — <specific_cause>"
evidence:
  - type: rbac_test
    content: "<kubectl auth can-i output>"
  - type: role_bindings
    content: "<relevant RoleBindings and Roles>"
severity: HIGH
mitigation:
  immediate: "Create or update the appropriate Role/RoleBinding"
  long_term: "Implement RBAC review process, use least privilege, document access policies"
```

## Safety Ratings
- GREEN: read-only (`kubectl auth can-i`, `kubectl get rolebindings`, `kubectl get clusterrolebindings`, `kubectl describe role`)
- YELLOW: state-changing recoverable (`kubectl create rolebinding`, `kubectl create clusterrolebinding`, `kubectl apply` Role/ClusterRole updates)
- RED: destructive/irreversible (`kubectl delete clusterrolebinding`, editing ClusterRoles that affect cluster-wide access, granting cluster-admin to broad groups)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Remediation requires cluster upgrade"
- "Change affects node group scaling in production"
- "Fix involves modifying network policies"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current RBAC bindings before modification: `kubectl get clusterrolebinding <name> -o yaml > crb-backup.yaml`"
- Pre-change: "Save current Role/ClusterRole before modification: `kubectl get clusterrole <name> -o yaml > cr-backup.yaml`"
- Verification: "Verify cluster access and permissions after change: `kubectl auth can-i --list`"
- Revert: "Restore RBAC bindings from backup if permissions break"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- HIGH: "ClusterRoleBindings reveal who has elevated cluster access"
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
