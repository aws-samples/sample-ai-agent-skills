---
title: "E2 — Flux GitOps"
description: "Diagnose Flux GitOps integration issues in EKS Anywhere"
status: active
severity: MEDIUM
triggers: ["Flux sync failed", "GitOps not reconciling", "git repository error", "kustomization failed", "Flux source error"]
owner: devops-agent
objective: "Resolve Flux GitOps issues including repository sync, reconciliation, and configuration drift"
context: "EKS Anywhere integrates Flux for GitOps-based cluster management. Flux continuously reconciles cluster state with a Git repository. Manual changes to Flux-managed resources are automatically reverted."
---

## Phase 1 — Triage

MUST:
- Check Flux controllers: `kubectl get pods -n flux-system --kubeconfig <workload-kubeconfig>`
- Check GitRepository source: `kubectl get gitrepositories -A --kubeconfig <workload-kubeconfig>`
- Check Kustomization status: `kubectl get kustomizations -A --kubeconfig <workload-kubeconfig>`

SHOULD:
- Review Flux source controller logs: `kubectl logs -n flux-system deployment/source-controller --tail=200`
- Check Flux kustomize controller logs: `kubectl logs -n flux-system deployment/kustomize-controller --tail=200`
- Verify Git repository accessibility and credentials

MAY:
- Force reconciliation: `flux reconcile source git <name>`
- Check Flux events: `kubectl get events -n flux-system --sort-by='.lastTimestamp'`

## Phase 2 — Remediate

MUST:
- Fix Git repository authentication (SSH keys, tokens, deploy keys)
- Resolve Kustomization build errors (invalid YAML, missing resources)
- Ensure Flux controllers are running and healthy

SHOULD:
- Verify Git repository URL and branch are correct in GitRepository resource
- Check that the repository path contains valid Kubernetes manifests

MAY:
- Suspend and resume Flux reconciliation to reset state
- Update Flux components if version incompatibility is detected

## Safety Ratings

```
safety_ratings:
  - "kubectl get gitrepositories/kustomizations: GREEN — read-only status check"
  - "kubectl logs flux controllers: GREEN — read-only log retrieval"
  - "flux reconcile: YELLOW — triggers immediate reconciliation, may apply pending changes"
  - "flux suspend/resume: YELLOW — pauses/resumes automatic reconciliation"
```

## Escalation Conditions

- Flux controllers crash-loop preventing all GitOps reconciliation
- Git repository credentials expired and cannot be rotated without admin access
- Kustomization applies destructive changes that affect production workloads
- Flux reconciliation loop causes continuous resource churn

## Data Sensitivity

- `kubectl get gitrepositories`: MEDIUM — contains repository URLs and branch information
- `kubectl get kustomizations`: LOW — returns reconciliation status
- `Flux controller logs`: MEDIUM — may contain repository paths and reconciliation details
- `Git repository secrets`: HIGH — contain SSH keys or access tokens

## Prohibited Actions

- NEVER delete the flux-system namespace — this removes all GitOps management
- NEVER commit secrets in plain text to the GitOps repository
- NEVER force-apply changes that bypass Flux reconciliation in production
- NEVER modify Flux-managed resources directly with kubectl when GitOps is active

## Phase 3 — Rollback

1. If a bad commit was applied, revert the Git commit and push: Flux will reconcile automatically
2. If Flux was suspended, resume it: `flux resume kustomization <name>`
3. If Flux controllers were modified, restore from original deployment manifests
4. Verify all Kustomizations are reconciled: `kubectl get kustomizations -A`
5. Check that cluster state matches the Git repository

## Common Issues

- symptoms: "GitRepository source fails — authentication required"
  diagnosis: "Git credentials (SSH key or token) are missing or expired."
  resolution: "Update the Git repository secret with valid credentials."

## Output Format

```yaml
root_cause: "flux_gitops — <specific_cause>"
evidence:
  - type: flux_error
    content: "<error details from Flux controllers>"
severity: MEDIUM
mitigation:
  immediate: "Fix GitOps sync"
  long_term: "Monitor Flux reconciliation and set up alerts"
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
