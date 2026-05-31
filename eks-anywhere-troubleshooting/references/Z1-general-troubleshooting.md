---
title: "Z1 — General EKS Anywhere Troubleshooting"
description: "Catch-all runbook for EKS Anywhere issues"
status: active
severity: MEDIUM
triggers: ["EKS Anywhere error", "eksctl anywhere not working", "cluster issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified EKS Anywhere issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check eksctl-anywhere version: `eksctl anywhere version`
- List clusters: `kubectl get clusters.anywhere.eks.amazonaws.com -A --kubeconfig <mgmt-kubeconfig>`
- Check all machines: `kubectl get machines -A -o wide --kubeconfig <mgmt-kubeconfig>`

SHOULD:
- Review recent events: `kubectl get events -A --sort-by='.lastTimestamp' --kubeconfig <mgmt-kubeconfig> | tail -50`
- Check controller pods: `kubectl get pods -n eksa-system -n capi-system --kubeconfig <mgmt-kubeconfig>`

MAY:
- Generate support bundle: `eksctl anywhere generate support-bundle --cluster <cluster-name>`
- Check AWS Health Dashboard for service issues

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for identified issue area

## Safety Ratings

```
safety_ratings:
  - "eksctl anywhere version: GREEN — read-only version check"
  - "kubectl get clusters/machines: GREEN — read-only status check"
  - "kubectl get events: GREEN — read-only event retrieval"
  - "eksctl anywhere generate support-bundle: GREEN — read-only diagnostics collection"
```

## Escalation Conditions

- Cluster is completely unreachable and no specific runbook applies
- Multiple subsystems are failing simultaneously
- Issue cannot be classified after systematic investigation
- Support bundle reveals issues requiring AWS support engagement

## Data Sensitivity

- `eksctl anywhere version`: LOW — returns version metadata
- `kubectl get clusters/machines`: LOW — returns resource metadata
- `kubectl get events`: LOW — returns cluster events
- `support bundle`: HIGH — contains comprehensive cluster state including potential secrets

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without identifying root cause
- NEVER delete cluster resources as a troubleshooting step
- NEVER share support bundles in public channels without sanitizing sensitive data
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore any resources that were modified during troubleshooting
3. Verify cluster health: `kubectl get nodes`, `kubectl get pods -A`
4. Document all investigation steps and findings
5. Verify the issue is resolved or properly escalated

## Common Issues

- symptoms: "General cluster instability — multiple components failing"
  diagnosis: "Often caused by management cluster resource exhaustion or network issues."
  resolution: "Check management cluster node resources and network connectivity first."

## Output Format

```yaml
root_cause: "general_eks_anywhere — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and alerting"
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
