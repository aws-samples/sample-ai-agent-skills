---
title: "H1 — Add-on Installation/Update Failures"
description: "Diagnose and resolve EKS managed add-on installation and update failures"
status: active
severity: HIGH
triggers:
  - "add-on failed"
  - "addon"
  - "CREATE_FAILED"
  - "DEGRADED"
  - "add-on installation"
owner: devops-agent
objective: "Identify why the EKS add-on failed to install or update and fix the issue"
context: "EKS managed add-ons (VPC CNI, CoreDNS, kube-proxy, EBS CSI, EFS CSI, Pod Identity Agent, etc.) are installed and managed via the EKS API. They can fail due to version incompatibility, IAM permission issues, configuration conflicts, or resource constraints."
---

## Phase 1 — Triage

MUST:
- Check add-on status: `aws eks describe-addon --cluster-name <cluster> --addon-name <addon>`
- Look at the health field and any issues reported
- List all add-ons: `aws eks list-addons --cluster-name <cluster>`
- Check for pending updates: `aws eks list-updates --name <cluster> --addon-name <addon>`
- Describe the failed update: `aws eks describe-update --name <cluster> --update-id <update-id>`

SHOULD:
- Check the add-on's pods in kube-system: `kubectl get pods -n kube-system -l <addon-label>`
- Check pod logs for the add-on
- Verify the add-on version is compatible with the cluster version: `aws eks describe-addon-versions --addon-name <addon> --kubernetes-version <version>`
- Check if there are configuration conflicts (custom values vs managed values)

MAY:
- Check CloudTrail for CreateAddon or UpdateAddon events
- Verify the add-on's service account and IRSA configuration
- Check if a self-managed version of the add-on is already installed (conflicts with managed add-on)

## Phase 2 — Remediate

MUST:
- If version incompatible: use a compatible version: `aws eks describe-addon-versions --addon-name <addon> --kubernetes-version <version> --query 'addons[0].addonVersions[*].addonVersion'`
- If configuration conflict: resolve conflicts with `--resolve-conflicts OVERWRITE` or `PRESERVE`
- If IAM issue: fix the add-on's service account role
- If self-managed conflict: remove the self-managed installation first

SHOULD:
- Update add-ons after cluster upgrades to maintain compatibility
- Use `--configuration-values` for custom configuration instead of editing resources directly
- Test add-on updates in non-production first

MAY:
- Delete and reinstall the add-on if it's in a bad state: `aws eks delete-addon --cluster-name <cluster> --addon-name <addon> --preserve`
- Use `--preserve` flag when deleting to keep the add-on's resources running

## Common Issues

- symptoms: "Add-on status is DEGRADED"
  diagnosis: "Add-on pods are not healthy — check pod status and logs."
  resolution: "Describe the add-on for health issues. Check pod logs in kube-system."

- symptoms: "Add-on creation fails with 'ConfigurationConflict'"
  diagnosis: "A self-managed version of the add-on already exists."
  resolution: "Use `--resolve-conflicts OVERWRITE` to replace, or remove the self-managed version first."

- symptoms: "Add-on update fails with version compatibility error"
  diagnosis: "The requested add-on version is not compatible with the cluster's Kubernetes version."
  resolution: "Use `describe-addon-versions` to find compatible versions."

## Output Format

```yaml
root_cause: "Add-on failure — <specific_cause>"
evidence:
  - type: addon_status
    content: "<describe-addon output>"
  - type: addon_pods
    content: "<pod status and logs>"
severity: HIGH
mitigation:
  immediate: "Fix the add-on configuration, version, or IAM permissions"
  long_term: "Automate add-on updates in CI/CD, maintain version compatibility matrix"
```

## Safety Ratings
- GREEN: read-only (`aws eks describe-addon`, `aws eks list-addons`, `aws eks list-updates`, `kubectl get pods -n kube-system`, `kubectl logs`)
- YELLOW: state-changing recoverable (`aws eks update-addon`, `aws eks create-addon`, `aws eks delete-addon --preserve`, resolve configuration conflicts with OVERWRITE/PRESERVE)
- RED: destructive/irreversible (`aws eks delete-addon` without `--preserve` removing running add-on resources, force-overwriting custom add-on configurations)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Remediation requires cluster upgrade"
- "Change affects node group scaling in production"
- "Fix involves deleting and reinstalling a critical add-on (VPC CNI, CoreDNS, kube-proxy) in production"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current add-on configuration values before update: `aws eks describe-addon --cluster-name <cluster> --addon-name <addon>`"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify add-on health and pod status after change"
- Revert: "Reinstall previous add-on version if update causes failures"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- MEDIUM: "Node group configuration reveals instance types and scaling"
- MEDIUM: "Add-on configuration values may reveal custom settings and IAM role ARNs"
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
