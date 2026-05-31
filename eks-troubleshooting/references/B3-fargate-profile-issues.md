---
title: "B3 — Fargate Profile Issues"
description: "Diagnose and resolve EKS Fargate pod scheduling and profile configuration problems"
status: active
severity: HIGH
triggers:
  - "Fargate"
  - "pod not scheduling on Fargate"
  - "Fargate profile"
  - "FailedScheduling"
owner: devops-agent
objective: "Identify why pods are not scheduling on Fargate and fix the Fargate profile configuration"
context: "Fargate profiles define which pods run on Fargate using namespace and label selectors. Pods must match a Fargate profile to be scheduled on Fargate. DaemonSets, privileged containers, HostNetwork, and HostPort are NOT supported on Fargate."
---

## Phase 1 — Triage

MUST:
- List Fargate profiles: `aws eks list-fargate-profiles --cluster-name <cluster>`
- Describe the Fargate profile: `aws eks describe-fargate-profile --cluster-name <cluster> --fargate-profile-name <profile>`
- Check the profile's namespace and label selectors
- Check the pod's namespace and labels: `kubectl get pod <pod> -n <namespace> --show-labels`
- Verify the pod matches the Fargate profile selectors

SHOULD:
- Check if the pod uses unsupported features: DaemonSet, privileged, HostNetwork, HostPort
- Verify the Fargate pod execution role: `aws iam get-role --role-name <fargate-role>`
- Check pod events for scheduling errors: `kubectl describe pod <pod> -n <namespace>`
- Verify the Fargate profile subnets have available IPs

MAY:
- Check if the namespace has other pods successfully running on Fargate
- Verify the Fargate profile status is ACTIVE
- Check CloudTrail for CreateFargateProfile events

## Phase 2 — Remediate

MUST:
- If selector mismatch: update the Fargate profile to match the pod's namespace/labels, or update the pod's labels
- If unsupported feature: remove DaemonSet, privileged, HostNetwork, or HostPort from the pod spec
- If IAM role issue: ensure the Fargate pod execution role has `AmazonEKSFargatePodExecutionRolePolicy`
- If subnet IP exhaustion: use subnets with available IPs (Fargate only uses private subnets)

SHOULD:
- Create separate Fargate profiles for different namespaces
- Ensure Fargate profile subnets are private (Fargate does not support public subnets)
- For logging: configure the Fargate built-in Fluent Bit log router via ConfigMap

MAY:
- Delete and recreate the Fargate profile if it's in a bad state
- Use `kubectl annotate` to add Fargate-specific annotations

## Common Issues

- symptoms: "Pod stuck in Pending, no Fargate node appears"
  diagnosis: "Pod namespace/labels don't match any Fargate profile selector."
  resolution: "Check Fargate profile selectors and pod labels. They must match exactly."

- symptoms: "Fargate pod fails with 'ErrImagePull' for ECR images"
  diagnosis: "Fargate pod execution role lacks ECR permissions or VPC has no NAT/VPC endpoint for ECR."
  resolution: "Add ECR permissions to the Fargate role. Ensure NAT gateway or ECR VPC endpoints exist."

- symptoms: "DaemonSet pods not running on Fargate nodes"
  diagnosis: "DaemonSets are not supported on Fargate. This is by design."
  resolution: "Use sidecar containers or Fargate's built-in Fluent Bit for logging instead of DaemonSets."

## Output Format

```yaml
root_cause: "Fargate profile issue — <specific_cause>"
evidence:
  - type: fargate_profile
    content: "<profile selectors and status>"
  - type: pod_labels
    content: "<pod namespace and labels>"
severity: HIGH
mitigation:
  immediate: "Fix selector matching or remove unsupported features"
  long_term: "Document Fargate profile selectors and validate pod specs against Fargate constraints"
```

## Safety Ratings
GREEN — Triage uses read-only commands (`aws eks describe-fargate-profile`, `kubectl get/describe`). Remediation involves updating Fargate profile selectors and pod labels, which are low-risk changes that don't affect existing running workloads.

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
