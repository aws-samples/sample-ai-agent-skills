---
title: "H2 — Add-on Version Compatibility"
description: "Diagnose and resolve EKS add-on version compatibility issues"
status: active
severity: MEDIUM
triggers:
  - "version compatibility"
  - "addon version"
  - "incompatible"
  - "deprecated"
owner: devops-agent
objective: "Identify add-on version mismatches and update to compatible versions"
context: "EKS add-ons have a compatibility matrix with Kubernetes versions. After a cluster upgrade, add-ons may need updating. Running incompatible versions can cause subtle failures — add-ons may appear running but behave incorrectly. Each add-on has a default version per Kubernetes version."
---

## Phase 1 — Triage

MUST:
- List current add-on versions: `for addon in $(aws eks list-addons --cluster-name <cluster> --query 'addons[]' --output text); do echo "$addon: $(aws eks describe-addon --cluster-name <cluster> --addon-name $addon --query 'addon.addonVersion' --output text)"; done`
- Check cluster Kubernetes version: `aws eks describe-cluster --name <cluster> --query 'cluster.version'`
- Check compatible versions for each add-on: `aws eks describe-addon-versions --addon-name <addon> --kubernetes-version <version>`
- Identify the default (recommended) version: `aws eks describe-addon-versions --addon-name <addon> --kubernetes-version <version> --query 'addons[0].addonVersions[?compatibilities[0].defaultVersion==\`true\`].addonVersion'`

SHOULD:
- Check if any add-ons are running versions older than the default for the current Kubernetes version
- Verify add-on health after cluster upgrades
- Check for deprecation notices in add-on descriptions

MAY:
- Review the EKS add-on version compatibility documentation
- Check if custom configuration values are compatible with the new add-on version
- Test version upgrades in a non-production cluster

## Phase 2 — Remediate

MUST:
- Update incompatible add-ons: `aws eks update-addon --cluster-name <cluster> --addon-name <addon> --addon-version <compatible-version>`
- Update add-ons after every cluster version upgrade
- Verify add-on health after updating: `aws eks describe-addon --cluster-name <cluster> --addon-name <addon>`

SHOULD:
- Update add-ons in order: VPC CNI → CoreDNS → kube-proxy → others
- Use `--resolve-conflicts PRESERVE` to keep custom configurations
- Document the add-on version matrix for your clusters

MAY:
- Automate add-on version checks in CI/CD pipelines
- Set up alerts for add-on version drift

## Common Issues

- symptoms: "Add-on running but cluster behavior is inconsistent after upgrade"
  diagnosis: "Add-on version is incompatible with the new Kubernetes version."
  resolution: "Update the add-on to a version compatible with the current cluster version."

- symptoms: "Cannot update add-on — 'addon version is not supported'"
  diagnosis: "Requested version is not available for the current Kubernetes version."
  resolution: "Use describe-addon-versions to find available versions."

- symptoms: "VPC CNI add-on outdated, new features not working"
  diagnosis: "VPC CNI version doesn't support the requested feature (e.g., prefix delegation)."
  resolution: "Update VPC CNI to the latest compatible version."

## Output Format

```yaml
root_cause: "Add-on version compatibility — <specific_cause>"
evidence:
  - type: addon_versions
    content: "<current vs compatible versions>"
  - type: cluster_version
    content: "<cluster Kubernetes version>"
severity: MEDIUM
mitigation:
  immediate: "Update add-ons to compatible versions"
  long_term: "Automate add-on version management, update add-ons as part of cluster upgrade process"
```

## Safety Ratings
- GREEN: read-only (`aws eks describe-addon`, `aws eks list-addons`, `aws eks describe-addon-versions`, `aws eks describe-cluster`)
- YELLOW: state-changing recoverable (`aws eks update-addon` to compatible version, update add-on configuration values with `--resolve-conflicts PRESERVE`)
- RED: destructive/irreversible (`aws eks update-addon --resolve-conflicts OVERWRITE` destroying custom configurations, downgrading add-ons to incompatible versions)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Remediation requires cluster upgrade"
- "Change affects node group scaling in production"
- "Fix involves updating critical add-ons (VPC CNI, CoreDNS, kube-proxy) in production"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current add-on versions and configuration before update"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify add-on health and cluster functionality after version update"
- Revert: "Reinstall previous add-on version if compatibility issues arise"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- MEDIUM: "Node group configuration reveals instance types and scaling"
- LOW: "Add-on version information"
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
