---
title: "Z1 — General DevOps Guru Troubleshooting"
description: "Catch-all for DevOps Guru issues"
status: active
severity: MEDIUM
triggers: ["DevOps Guru error", "something wrong", "not working"]
owner: devops-agent
objective: "Systematic triage for unclassified issues"
context: "Use when symptoms don't map to a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check account health: `aws devops-guru describe-account-health`
- Verify IAM permissions
- Check service health dashboard
- Review CloudTrail for errors

## Phase 2 — Route to specific runbook

Based on findings, route to A1-A3, B1-B2, C1-C2, D1-D2, E1-E2, or F1-F2.

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: service_status
    content: "<service health>"
severity: MEDIUM
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — General troubleshooting may expose operational monitoring configuration |
| Blast Radius | LOW — Diagnostic commands are read-only |
| Rollback Complexity | LOW — No changes during triage |
| Compliance Scope | MEDIUM — DevOps Guru supports operational monitoring requirements |

## Escalation Conditions

- Escalate to the specific runbook owner once the problem domain is identified
- Escalate to AWS Support if systematic issues affect DevOps Guru operations

## Data Sensitivity

- **Account health data** reveals operational monitoring status
- **CloudTrail events** contain API call details

## PROHIBITED Actions

- Do NOT modify coverage or notification settings during initial triage
- Do NOT share operational monitoring details outside the operations team

## Phase 3 — Rollback

- No rollback needed for diagnostic operations
- Route to the appropriate specific runbook for remediation

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
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
