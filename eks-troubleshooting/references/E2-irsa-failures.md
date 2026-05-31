---
title: "E2 — IRSA Failures"
description: "Diagnose and resolve IAM Roles for Service Accounts (IRSA) issues"
status: active
severity: HIGH
triggers:
  - "IRSA"
  - "service account"
  - "AssumeRoleWithWebIdentity"
  - "OIDC"
  - "pod IAM"
owner: devops-agent
objective: "Identify why IRSA is not working and fix the IAM role assumption chain"
context: "IRSA allows pods to assume IAM roles via service accounts. It requires three components: (1) OIDC provider, (2) annotated service account, (3) IAM role trust policy. The pod gets a projected service account token that it exchanges for temporary AWS credentials via STS."
---

## Phase 1 — Triage

MUST:
- Check the OIDC provider: `aws eks describe-cluster --name <cluster> --query 'cluster.identity.oidc.issuer'`
- Verify OIDC provider exists in IAM: `aws iam list-open-id-connect-providers`
- Check service account annotation: `kubectl get sa <sa-name> -n <namespace> -o yaml`
- Verify the annotation has `eks.amazonaws.com/role-arn: arn:aws:iam::<account>:role/<role>`
- Check the IAM role trust policy: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'`

SHOULD:
- Verify the trust policy condition matches the service account: `system:serviceaccount:<namespace>:<sa-name>`
- Check if the pod is using the correct service account: `kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.serviceAccountName}'`
- Verify the projected token is mounted: `kubectl exec <pod> -n <namespace> -- ls /var/run/secrets/eks.amazonaws.com/serviceaccount/`
- Test from inside the pod: `kubectl exec <pod> -n <namespace> -- aws sts get-caller-identity`

MAY:
- Check the token audience: `kubectl exec <pod> -n <namespace> -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | cut -d. -f2 | base64 -d`
- Verify the IAM role has the required permissions policies attached
- Check CloudTrail for AssumeRoleWithWebIdentity failures

## Phase 2 — Remediate

MUST:
- If OIDC provider missing: create it: `eksctl utils associate-iam-oidc-provider --cluster <cluster> --approve`
- If service account annotation missing: annotate it: `kubectl annotate sa <sa-name> -n <namespace> eks.amazonaws.com/role-arn=arn:aws:iam::<account>:role/<role>`
- If trust policy wrong: update the IAM role trust policy to include the correct OIDC provider and service account condition
- If pod not using the service account: update the pod spec with `serviceAccountName: <sa-name>`

SHOULD:
- Restart pods after fixing IRSA (tokens are mounted at pod creation): `kubectl rollout restart deployment/<name> -n <namespace>`
- Use the exact OIDC issuer URL (without https://) in the trust policy
- Verify the trust policy uses `StringEquals` (exact match) or `StringLike` (wildcard) correctly

MAY:
- Use `eksctl create iamserviceaccount` for automated IRSA setup
- Consider migrating to EKS Pod Identity for simpler configuration

## Common Issues

- symptoms: "Pod uses default node role instead of the IRSA role"
  diagnosis: "IRSA is not configured correctly — one of the three components is missing."
  resolution: "Check all three: OIDC provider, service account annotation, IAM role trust policy."

- symptoms: "AssumeRoleWithWebIdentity returns 'Not authorized to perform sts:AssumeRoleWithWebIdentity'"
  diagnosis: "IAM role trust policy doesn't allow the OIDC provider or service account."
  resolution: "Fix the trust policy. Ensure the OIDC provider ARN and service account condition are correct."

- symptoms: "Token not mounted in the pod at /var/run/secrets/eks.amazonaws.com/"
  diagnosis: "Service account annotation was added after pod creation, or webhook is not running."
  resolution: "Restart the pod. Check the amazon-eks-pod-identity-webhook is running."

- symptoms: "IRSA works for some pods but not others in the same namespace"
  diagnosis: "Pods are using different service accounts, or the trust policy has a specific SA condition."
  resolution: "Verify each pod's serviceAccountName matches the annotated service account."

## Output Format

```yaml
root_cause: "IRSA failure — <specific_cause>"
evidence:
  - type: oidc_provider
    content: "<OIDC issuer URL and IAM provider status>"
  - type: service_account
    content: "<SA annotation and pod assignment>"
  - type: trust_policy
    content: "<IAM role trust policy>"
severity: HIGH
mitigation:
  immediate: "Fix the missing IRSA component and restart pods"
  long_term: "Use IaC for IRSA setup, implement IRSA validation in CI/CD"
```

## Safety Ratings
- GREEN: read-only (`kubectl get sa`, `aws iam get-role`, `aws eks describe-cluster`, `aws iam list-open-id-connect-providers`)
- YELLOW: state-changing recoverable (`kubectl annotate sa`, `kubectl rollout restart`, `eksctl create iamserviceaccount`, update IAM trust policy)
- RED: destructive/irreversible (`aws iam delete-open-id-connect-provider`, deleting IRSA roles used by production workloads, modifying trust policies that lock out service accounts)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Remediation requires cluster upgrade"
- "Change affects node group scaling in production"
- "Fix involves modifying IAM trust policies for production service accounts"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current IAM role trust policy before modification: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'`"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify IRSA credentials from inside pod after change: `kubectl exec <pod> -- aws sts get-caller-identity`"
- Revert: "Restore IAM trust policy from backup if service accounts lose access"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- HIGH: "OIDC provider configuration reveals cluster identity federation details"
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
