---
title: "F2 — IAM Authenticator"
description: "Diagnose AWS IAM Authenticator issues in EKS Anywhere"
status: active
severity: HIGH
triggers: ["IAM authenticator error", "authentication failed", "aws-iam-authenticator crash", "RBAC denied", "kubeconfig auth error"]
owner: devops-agent
objective: "Resolve IAM Authenticator issues including token validation, ConfigMap mapping, and authentication failures"
context: "EKS Anywhere can use aws-iam-authenticator for Kubernetes authentication using AWS IAM identities. It runs as a DaemonSet on control plane nodes and validates tokens against AWS IAM."
---

## Phase 1 — Triage

MUST:
- Check IAM authenticator pods: `kubectl get pods -n kube-system -l app=aws-iam-authenticator --kubeconfig <workload-kubeconfig>`
- Review authenticator logs: `kubectl logs -n kube-system ds/aws-iam-authenticator --tail=200 --kubeconfig <workload-kubeconfig>`
- Check aws-auth ConfigMap: `kubectl get configmap aws-auth -n kube-system -o yaml --kubeconfig <workload-kubeconfig>`

SHOULD:
- Verify IAM identity mapping in aws-auth ConfigMap
- Check STS endpoint connectivity from control plane nodes
- Verify the IAM role/user ARN format in the ConfigMap

MAY:
- Test token generation: `aws eks get-token --cluster-name <cluster-name>`
- Check API server authentication webhook configuration

## Phase 2 — Remediate

MUST:
- Fix aws-auth ConfigMap mappings (correct IAM ARN format, valid Kubernetes groups)
- Resolve IAM authenticator pod crashes
- Ensure control plane nodes can reach AWS STS endpoints

SHOULD:
- Verify IAM roles/users exist and are not deleted
- Check that kubeconfig uses the correct cluster name for token generation

MAY:
- Add backup authentication method (client certificates) for emergency access
- Configure IAM authenticator logging verbosity for debugging

## Safety Ratings

```
safety_ratings:
  - "kubectl get pods aws-iam-authenticator: GREEN — read-only status check"
  - "kubectl get configmap aws-auth: GREEN — read-only configuration check"
  - "kubectl edit configmap aws-auth: RED — incorrect changes lock out all IAM-authenticated users"
  - "aws eks get-token: GREEN — generates temporary authentication token"
```

## Escalation Conditions

- All IAM-authenticated users locked out of the cluster
- aws-iam-authenticator pods crash on all control plane nodes
- AWS STS endpoint unreachable from on-premises network
- aws-auth ConfigMap accidentally deleted or corrupted

## Data Sensitivity

- `kubectl get configmap aws-auth`: MEDIUM — contains IAM ARN to Kubernetes RBAC mappings
- `aws-iam-authenticator logs`: MEDIUM — may contain IAM identity details and authentication attempts
- `aws eks get-token`: HIGH — generates bearer token for cluster authentication
- `API server webhook config`: MEDIUM — contains authenticator endpoint configuration

## Prohibited Actions

- NEVER delete the aws-auth ConfigMap — this locks out all IAM-authenticated users
- NEVER remove the last admin mapping from aws-auth without alternative access
- NEVER expose IAM authenticator tokens in logs or support bundles
- NEVER configure aws-auth with overly broad IAM role mappings (e.g., mapping * to cluster-admin)

## Phase 3 — Rollback

1. If aws-auth ConfigMap was corrupted, restore from backup: `kubectl apply -f aws-auth-backup.yaml`
2. If locked out, use certificate-based kubeconfig (admin kubeconfig from cluster creation) to restore access
3. If authenticator pods were modified, restore DaemonSet from original manifests
4. Verify authentication works: `kubectl auth can-i get pods --as=<iam-mapped-user>`
5. Document the authentication issue and update emergency access procedures

## Common Issues

- symptoms: "Authentication failed — could not authenticate with AWS IAM"
  diagnosis: "IAM role ARN in aws-auth ConfigMap does not match the caller's assumed role."
  resolution: "Verify the IAM role ARN format (no path) and update aws-auth ConfigMap."

## Output Format

```yaml
root_cause: "iam_authenticator — <specific_cause>"
evidence:
  - type: auth_error
    content: "<error details from authenticator logs>"
severity: HIGH
mitigation:
  immediate: "Restore authentication"
  long_term: "Maintain backup access method and aws-auth backups"
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
