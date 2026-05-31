---
title: "Z1 — General EKS Troubleshooting (Catch-All)"
description: "Fallback SOP for EKS issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown EKS issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Verify kubectl connectivity: `kubectl config current-context` and `kubectl cluster-info`
- Check cluster status: `aws eks describe-cluster --name <cluster> --query 'cluster.{status:status,version:version,endpoint:endpoint}'`
- Check node health: `kubectl get nodes -o wide`
- Check pod status across all namespaces: `kubectl get pods -A --field-selector status.phase!=Running`
- Check recent events: `kubectl get events --sort-by='.lastTimestamp' -A | tail -50`
- Check kube-system pods (critical infrastructure): `kubectl get pods -n kube-system`

SHOULD:
- Check node conditions: `kubectl describe nodes | grep -A10 Conditions`
- Check cluster add-ons: `aws eks list-addons --cluster-name <cluster>`
- Check CloudWatch for control plane logs (if enabled)
- Verify AWS credentials: `aws sts get-caller-identity`

MAY:
- Check CloudTrail for recent EKS API events
- Check AWS Health Dashboard for service events
- Review recent deployments or changes: `kubectl rollout history deployment -A`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Cluster unreachable → Endpoint access (A2)
- Cluster creation/upgrade issue → Cluster (A1, A3)
- Nodes NotReady → Node health (B1-B4)
- Pods Pending → Scheduling (C1)
- Pods CrashLoopBackOff → Application (C2)
- Pods OOMKilled → Memory (C3)
- Image pull errors → Registry (C4)
- DNS failures → CoreDNS (D2)
- Service connectivity → Networking (D1, D3, D4)
- Permission denied → IAM/RBAC (E1-E4)
- Storage issues → CSI drivers (F1-F3)
- Scaling not working → Autoscaling (G1-G3)
- Add-on failures → Add-ons (H1-H3)
- Missing metrics/logs → Observability (I1-I2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all kube-system pods for failures
- Review CloudWatch metrics for anomalies (if Container Insights enabled)
- Check aws-auth ConfigMap for recent changes
- Review VPC configuration (subnets, security groups, NACLs)
- Check for recent cluster or node group updates

SHOULD:
- Compare with a known-good cluster configuration
- Check AWS service health for the region
- Review recent IAM policy changes that might affect the cluster

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<cluster|nodes|pods|networking|iam_rbac|storage|scaling|addons|observability|unknown>"
investigation_path: "cluster-info → nodes → pods → events → <domain_classification>"
evidence:
  - type: cluster_status
    content: "<cluster details>"
  - type: node_health
    content: "<node status>"
  - type: pod_status
    content: "<failing pods>"
  - type: events
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```

## Safety Ratings
- GREEN: read-only (`kubectl config current-context`, `kubectl cluster-info`, `kubectl get nodes`, `kubectl get pods -A`, `kubectl get events`, `aws eks describe-cluster`, `aws sts get-caller-identity`)
- YELLOW: state-changing recoverable (`kubectl rollout restart`, `kubectl scale`, `aws eks update-cluster-config` for logging, update add-ons)
- RED: destructive/irreversible (`kubectl delete`, `kubectl drain`, cluster upgrade, editing aws-auth ConfigMap without backup)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Remediation requires cluster upgrade"
- "Change affects node group scaling in production"
- "Fix involves modifying network policies"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify cluster access after change"
- Revert: "Restore aws-auth from backup if locked out"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
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
