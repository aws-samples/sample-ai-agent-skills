---
title: "H3 — Custom Add-ons"
description: "Diagnose and resolve issues with self-managed and custom add-ons on EKS"
status: active
severity: MEDIUM
triggers:
  - "custom add-on"
  - "Helm"
  - "self-managed"
  - "third-party"
  - "operator"
owner: devops-agent
objective: "Identify and fix issues with custom or self-managed add-ons running on EKS"
context: "Custom add-ons include self-managed installations of core components (VPC CNI, CoreDNS), third-party tools (Istio, Prometheus, cert-manager), and custom operators. They are managed via Helm, kubectl apply, or operators. They can conflict with EKS managed add-ons and may have EKS-specific requirements."
---

## Phase 1 — Triage

MUST:
- Identify the add-on and how it was installed (Helm, kubectl, operator)
- Check the add-on's pods: `kubectl get pods -n <namespace> -l <addon-labels>`
- Check pod logs: `kubectl logs -n <namespace> <pod> --tail=50`
- Describe pods for events: `kubectl describe pod <pod> -n <namespace>`
- Check if the add-on conflicts with an EKS managed add-on

SHOULD:
- For Helm-installed add-ons: `helm list -n <namespace>` and `helm status <release> -n <namespace>`
- Check the add-on's CRDs: `kubectl get crds | grep <addon>`
- Verify the add-on's service account and RBAC permissions
- Check if the add-on version is compatible with the cluster's Kubernetes version

MAY:
- Check Helm values: `helm get values <release> -n <namespace>`
- Review the add-on's documentation for EKS-specific requirements
- Check for webhook configurations that might interfere: `kubectl get validatingwebhookconfigurations` and `kubectl get mutatingwebhookconfigurations`

## Phase 2 — Remediate

MUST:
- If pod failures: fix the root cause (image pull, resource limits, configuration)
- If conflict with managed add-on: remove one — either switch to managed or keep self-managed
- If RBAC issues: fix the add-on's ClusterRole/Role and bindings
- If CRD issues: update or reinstall CRDs

SHOULD:
- Use Helm for add-on lifecycle management (install, upgrade, rollback)
- Pin add-on versions in Helm values for reproducibility
- Test add-on upgrades in non-production first

MAY:
- Migrate self-managed core add-ons to EKS managed add-ons
- Use GitOps (ArgoCD, Flux) for add-on management
- Set up monitoring for custom add-on health

## Common Issues

- symptoms: "Self-managed VPC CNI conflicts with EKS managed add-on"
  diagnosis: "Both managed and self-managed versions are trying to manage the same resources."
  resolution: "Choose one. Delete the managed add-on with `--preserve` or remove the self-managed installation."

- symptoms: "Webhook add-on (cert-manager, OPA) blocking pod creation"
  diagnosis: "Validating/mutating webhook is rejecting pod specs."
  resolution: "Check webhook logs. Temporarily disable the webhook if it's blocking critical workloads."

- symptoms: "Custom operator CRDs missing after cluster upgrade"
  diagnosis: "CRDs were not preserved during upgrade or were accidentally deleted."
  resolution: "Reinstall the CRDs. Check if the operator manages its own CRDs."

## Output Format

```yaml
root_cause: "Custom add-on issue — <specific_cause>"
evidence:
  - type: addon_pods
    content: "<pod status and logs>"
  - type: addon_config
    content: "<Helm values or deployment configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix the add-on configuration or resolve conflicts"
  long_term: "Use Helm/GitOps for add-on management, maintain compatibility documentation"
```

## Safety Ratings
- GREEN: read-only (`kubectl get pods`, `kubectl logs`, `kubectl describe pod`, `helm list`, `helm status`, `kubectl get crds`, `kubectl get validatingwebhookconfigurations`)
- YELLOW: state-changing recoverable (`helm upgrade`, `kubectl apply` CRD updates, `kubectl apply` RBAC fixes, `helm rollback`, disable/re-enable webhooks)
- RED: destructive/irreversible (`helm uninstall` in production, `kubectl delete crd` removing all custom resources, deleting webhook configurations affecting cluster-wide admission)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Change affects node group scaling in production"
- "Fix involves modifying validating/mutating webhooks in production"
- "Remediation requires uninstalling and reinstalling a custom add-on in production"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current Helm values before upgrade: `helm get values <release> -n <namespace>`"
- Pre-change: "Save current RBAC bindings before modification"
- Pre-change: "Save current webhook configurations before modification"
- Verification: "Verify add-on health and cluster functionality after change"
- Revert: "Rollback Helm release if upgrade causes issues: `helm rollback <release> -n <namespace>`"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- MEDIUM: "Node group configuration reveals instance types and scaling"
- MEDIUM: "Helm values may reveal secrets, endpoints, and configuration details"
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
