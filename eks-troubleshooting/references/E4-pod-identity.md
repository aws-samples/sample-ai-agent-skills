---
title: "E4 — Pod Identity Issues"
description: "Diagnose and resolve EKS Pod Identity association problems"
status: active
severity: HIGH
triggers:
  - "pod identity"
  - "EKS Pod Identity"
  - "pod-identity-agent"
  - "pod credentials"
owner: devops-agent
objective: "Identify why EKS Pod Identity is not working and fix the association"
context: "EKS Pod Identity is a newer alternative to IRSA for granting IAM permissions to pods. It uses the EKS Pod Identity Agent (a DaemonSet add-on) and pod identity associations configured via the EKS API. It's simpler than IRSA — no OIDC provider or trust policy management needed."
---

## Phase 1 — Triage

MUST:
- Check if the Pod Identity Agent add-on is installed: `aws eks describe-addon --cluster-name <cluster> --addon-name eks-pod-identity-agent`
- Check Pod Identity Agent pods: `kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent`
- List pod identity associations: `aws eks list-pod-identity-associations --cluster-name <cluster>`
- Describe the specific association: `aws eks describe-pod-identity-association --cluster-name <cluster> --association-id <id>`
- Verify the pod's service account matches the association

SHOULD:
- Check the IAM role's trust policy allows `pods.eks.amazonaws.com` as a principal
- Verify the pod is using the correct service account: `kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.serviceAccountName}'`
- Test credentials from inside the pod: `kubectl exec <pod> -n <namespace> -- aws sts get-caller-identity`
- Check Pod Identity Agent logs: `kubectl logs -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent --tail=50`

MAY:
- Check if IRSA is also configured (Pod Identity takes precedence over IRSA)
- Verify the EKS cluster version supports Pod Identity (1.24+)
- Check CloudTrail for AssumeRoleForPodIdentity events

## Phase 2 — Remediate

MUST:
- If add-on not installed: install it: `aws eks create-addon --cluster-name <cluster> --addon-name eks-pod-identity-agent`
- If association missing: create it: `aws eks create-pod-identity-association --cluster-name <cluster> --namespace <ns> --service-account <sa> --role-arn <role-arn>`
- If trust policy wrong: update the IAM role trust policy to allow `pods.eks.amazonaws.com`
- Restart pods after creating/fixing the association

SHOULD:
- Ensure the Pod Identity Agent add-on is up to date
- Use Pod Identity for new workloads (simpler than IRSA)
- Verify the IAM role has the required permission policies

MAY:
- Migrate existing IRSA workloads to Pod Identity
- Use tags on pod identity associations for organization

## Common Issues

- symptoms: "Pod uses node role instead of the pod identity role"
  diagnosis: "Pod Identity Agent not installed or association not configured."
  resolution: "Install the eks-pod-identity-agent add-on and create the pod identity association."

- symptoms: "Pod Identity Agent pods are CrashLoopBackOff"
  diagnosis: "Agent cannot start — often due to node permissions or incompatible version."
  resolution: "Check agent logs. Update the add-on to the latest compatible version."

- symptoms: "Association exists but pod still uses wrong credentials"
  diagnosis: "Pod was created before the association. Tokens are injected at pod creation."
  resolution: "Restart the pod: `kubectl rollout restart deployment/<name> -n <namespace>`."

## Output Format

```yaml
root_cause: "Pod Identity issue — <specific_cause>"
evidence:
  - type: addon_status
    content: "<Pod Identity Agent add-on status>"
  - type: association
    content: "<pod identity association details>"
severity: HIGH
mitigation:
  immediate: "Fix the Pod Identity configuration and restart pods"
  long_term: "Standardize on Pod Identity for new workloads, automate associations via IaC"
```

## Safety Ratings
- GREEN: read-only (`aws eks describe-addon`, `aws eks list-pod-identity-associations`, `kubectl get pods`, `kubectl get sa`)
- YELLOW: state-changing recoverable (`aws eks create-pod-identity-association`, `aws eks create-addon`, `kubectl rollout restart`, update IAM trust policy)
- RED: destructive/irreversible (`aws eks delete-pod-identity-association` for production workloads, deleting Pod Identity Agent add-on, modifying IAM roles used by production pods)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Remediation requires cluster upgrade"
- "Change affects node group scaling in production"
- "Fix involves modifying IAM roles or pod identity associations in production"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current pod identity associations: `aws eks list-pod-identity-associations --cluster-name <cluster>`"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify pod credentials after change: `kubectl exec <pod> -- aws sts get-caller-identity`"
- Revert: "Recreate pod identity association from backup if pods lose IAM access"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- HIGH: "Pod identity associations reveal IAM role mappings per namespace/service account"
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
