---
title: "C2 — EKS Protection Issues"
description: "Diagnose GuardDuty EKS protection and Kubernetes threat detection issues"
status: active
severity: HIGH
triggers:
  - "EKS protection"
  - "EKS findings"
  - "Kubernetes threats"
  - "EKS audit logs"
  - "EKS runtime"
owner: devops-agent
objective: "Identify and resolve EKS protection issues for Kubernetes threat detection"
context: "GuardDuty EKS Protection has two components: EKS Audit Log Monitoring (analyzes Kubernetes audit logs) and EKS Runtime Monitoring (detects OS/container-level threats via agent). These are separate features with separate enablement and pricing."
---

## Phase 1 — Triage

MUST:
- Check EKS protection status: `aws guardduty get-detector --detector-id <detector-id>` (check EKS_AUDIT_LOGS and EKS_RUNTIME_MONITORING features)
- List EKS-related findings: `aws guardduty list-findings --detector-id <detector-id> --finding-criteria '{"Criterion":{"type":{"Prefix":"Kubernetes"}}}' --max-results 10`
- Verify EKS clusters exist: `aws eks list-clusters`
- Check runtime coverage for EKS: `aws guardduty list-coverage --detector-id <detector-id> --filter-criteria '{"FilterCriterion":[{"CriterionKey":"RESOURCE_TYPE","FilterCondition":{"EqualsValue":"EKS"}}]}'`

SHOULD:
- Check EKS cluster details: `aws eks describe-cluster --name <cluster-name>`
- Verify the GuardDuty EKS add-on is installed: `aws eks list-addons --cluster-name <cluster-name>`
- Check coverage statistics: `aws guardduty get-coverage-statistics --detector-id <detector-id> --statistics-type COUNT_BY_COVERAGE_STATUS`

MAY:
- Check EKS node group status for agent deployment
- Verify the EKS cluster security group allows agent communication

## Phase 2 — Remediate

MUST:
- Enable EKS Audit Log Monitoring if disabled
- Enable EKS Runtime Monitoring if runtime findings are needed
- Deploy the GuardDuty agent for runtime monitoring

SHOULD:
- Enable auto-management for the GuardDuty EKS add-on
- Verify coverage status shows HEALTHY for EKS resources

MAY:
- Set up EventBridge rules for Kubernetes finding response
- Configure EKS-specific suppression rules for expected activity

## Common Issues

- symptoms: "No Kubernetes findings despite EKS clusters running"
  diagnosis: "EKS Audit Log Monitoring is not enabled in the detector."
  resolution: "Enable EKS_AUDIT_LOGS feature in the detector configuration."

- symptoms: "EKS audit log findings work but no runtime findings"
  diagnosis: "EKS Runtime Monitoring is not enabled or the GuardDuty agent is not deployed."
  resolution: "Enable EKS_RUNTIME_MONITORING and deploy the GuardDuty EKS add-on to clusters."

- symptoms: "Runtime coverage shows UNHEALTHY for EKS clusters"
  diagnosis: "The GuardDuty agent (EKS add-on) failed to deploy or is not running on nodes."
  resolution: "Check EKS add-on status. Verify node IAM role has required permissions. Check agent pod logs."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable EKS protection features | YELLOW | Configuration change; reversible |
| Deploy GuardDuty EKS add-on | YELLOW | Add-on deployment; can be removed |
| Enable auto-management | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Detector configuration changes
- EKS add-on deployment to production clusters

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-detector` (EKS features) | LOW | Feature status |
| `list-findings` (Kubernetes) | HIGH | Kubernetes threat details |
| `list-coverage` | MEDIUM | Resource coverage details |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| EKS protection enablement | Disable via `update-detector` (not recommended) |
| EKS add-on deployment | Remove via `delete-addon` |
| Auto-management enablement | Disable auto-management |

## Output Format

```yaml
root_cause: "eks_protection — <specific_cause>"
evidence:
  - type: detector_features
    content: "<EKS feature status>"
  - type: coverage
    content: "<EKS runtime coverage>"
  - type: clusters
    content: "<EKS cluster and add-on status>"
severity: HIGH
mitigation:
  immediate: "Enable EKS protection features and deploy agent"
  long_term: "Monitor coverage status and automate agent deployment"
```

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
  - command: "get-findings"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-detectors"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-threat-intel-set"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling GuardDuty to stop alerts"
  - "NEVER suggest archiving findings without investigation"
  - "NEVER expose finding details in public logs"
