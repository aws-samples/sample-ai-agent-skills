---
title: "A3 — Cluster Upgrade Failures"
description: "Diagnose and resolve EKS cluster version upgrade issues"
status: active
severity: HIGH
triggers:
  - "upgrade failed"
  - "update failed"
  - "version compatibility"
  - "deprecated API"
owner: devops-agent
objective: "Identify why the cluster upgrade failed and complete the upgrade successfully"
context: "EKS cluster upgrades must go one minor version at a time. Upgrades update the control plane first, then node groups and add-ons must be updated separately. Deprecated APIs, incompatible add-ons, and node group AMI mismatches are common failure causes."
---

## Phase 1 — Triage

MUST:
- Check cluster version and update status: `aws eks describe-cluster --name <cluster> --query 'cluster.{version:version,status:status,platformVersion:platformVersion}'`
- List pending updates: `aws eks list-updates --name <cluster>`
- Describe the failed update: `aws eks describe-update --name <cluster> --update-id <update-id>`
- Check for deprecated API usage: `kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis`

SHOULD:
- Check add-on compatibility: `aws eks describe-addon-versions --kubernetes-version <target-version>`
- Check current add-on versions: `aws eks list-addons --cluster-name <cluster>` then describe each
- Verify node group AMI compatibility: `aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> --query 'nodegroup.{version:version,releaseVersion:releaseVersion}'`
- Check CloudTrail for UpdateClusterVersion events

MAY:
- Run the Kubernetes deprecation checker: `kubectl api-resources` and compare with target version changelog
- Check if any webhooks might block the upgrade
- Review EKS release notes for the target version

## Phase 2 — Remediate

MUST:
- Resolve any deprecated API usage before retrying the upgrade
- Ensure all add-ons have compatible versions for the target Kubernetes version
- Upgrade one minor version at a time (e.g., 1.27 → 1.28, NOT 1.27 → 1.29)
- After control plane upgrade, update node groups: `aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name <ng>`

SHOULD:
- Update add-ons after the control plane upgrade: `aws eks update-addon --cluster-name <cluster> --addon-name <addon> --addon-version <version>`
- Test in a non-production cluster first
- Review the EKS Kubernetes version support calendar for EOL dates

MAY:
- Use eksctl for managed upgrades: `eksctl upgrade cluster --name <cluster> --version <version>`
- Create a new node group with the target version and drain the old one

## Common Issues

- symptoms: "Update status is Failed with 'Kubernetes version update is not allowed'"
  diagnosis: "Attempting to skip a minor version."
  resolution: "Upgrade sequentially. Check current version and upgrade to the next minor version only."

- symptoms: "Control plane upgraded but pods are failing"
  diagnosis: "Add-ons or workloads using deprecated APIs removed in the new version."
  resolution: "Update add-ons to compatible versions. Fix deprecated API usage in manifests."

- symptoms: "Node group update fails after control plane upgrade"
  diagnosis: "Node group AMI not available for the new version, or launch template incompatibility."
  resolution: "Check available AMIs: `aws ssm get-parameter --name /aws/service/eks/optimized-ami/<version>/amazon-linux-2/recommended/image_id`."

## Output Format

```yaml
root_cause: "Cluster upgrade failure — <specific_cause>"
evidence:
  - type: update_status
    content: "<describe-update output>"
  - type: deprecated_apis
    content: "<deprecated API usage if found>"
severity: HIGH
mitigation:
  immediate: "Resolve the blocking issue and retry the upgrade"
  long_term: "Implement pre-upgrade checks in CI/CD pipeline, test upgrades in staging first"
```

## Safety Ratings
YELLOW — Triage is read-only (`aws eks describe-cluster`, `kubectl get`). Remediation involves cluster upgrades (`aws eks update-cluster-version`), node group updates, and add-on updates which modify running infrastructure but are managed operations with built-in rollback.

## Escalation Conditions
- Remediation requires editing aws-auth ConfigMap in production
- Fix involves modifying RBAC ClusterRoleBindings
- Node group changes affect running production workloads
- Cluster endpoint access changes could lock out users
- IRSA trust policy modifications are needed

## Rollback
1. Before aws-auth changes: `kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-backup.yaml`
2. Before RBAC changes: Save current bindings with `kubectl get clusterrolebinding <name> -o yaml`
3. After change: Verify cluster access and workload health
4. If change causes lockout: Use cluster creator credentials or restore from backup
5. Cleanup: Remove any temporary RBAC bindings or test service accounts

## Data Sensitivity
| Command | Sensitivity | Handling |
|---------|------------|----------|
| `kubectl get configmap aws-auth` | HIGH | IAM-to-RBAC mappings — redact |
| `kubectl get sa -A -o yaml` | HIGH | Service account annotations with role ARNs — redact |
| `kubectl get events` | MEDIUM | Cluster events — summarize |
| `kubectl get nodes/pods` | LOW | Resource status — safe to include |

## Prohibited Actions
- NEVER suggest adding `system:masters` group for troubleshooting access
- NEVER suggest disabling Pod Security Standards/Admission
- NEVER suggest running containers as root to fix permission issues
- NEVER suggest `--force` delete of namespaces or persistent volumes
- NEVER suggest modifying kube-system resources without backup
- ALWAYS backup aws-auth before any modification
- ALWAYS use specific RBAC roles instead of cluster-admin

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
