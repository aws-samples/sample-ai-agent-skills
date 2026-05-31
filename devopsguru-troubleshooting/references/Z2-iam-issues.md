---
title: "Z2 — DevOps Guru IAM Issues"
description: "Diagnose IAM permission errors"
status: active
severity: HIGH
triggers: ["AccessDeniedException", "permission denied", "not authorized"]
owner: devops-agent
objective: "Resolve IAM permission issues"
context: "DevOps Guru requires specific IAM permissions."
---

## Phase 1 — Triage

MUST:
- Check error message for denied action
- Verify IAM policy includes devops-guru permissions
- Check for SCPs blocking access

## Phase 2 — Remediate

MUST:
- Add missing permissions
- Fix SCP conflicts

## Common Issues

- symptoms: "AccessDeniedException on describe-account-health"
  diagnosis: "Missing devops-guru:DescribeAccountHealth permission."
  resolution: "Add devops-guru:* or specific actions to IAM policy."

## Output Format

```yaml
root_cause: "iam — <specific_cause>"
evidence:
  - type: iam_policy
    content: "<policy evaluation>"
severity: HIGH
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — IAM permissions control access to operational monitoring |
| Blast Radius | MEDIUM — Permission changes affect who can view operational insights |
| Rollback Complexity | LOW — IAM policy changes can be reverted |
| Compliance Scope | MEDIUM — Access to operational monitoring should be controlled |

## Escalation Conditions

- Escalate to IAM administrators if permission issues block operational monitoring
- Escalate to security if SCPs are blocking DevOps Guru operations

## Data Sensitivity

- **IAM policies** for DevOps Guru control access to operational data
- **SCP configurations** may affect multiple accounts

## PROHIBITED Actions

- Do NOT grant devops-guru:* to users who don't need operational monitoring access
- Do NOT modify SCPs without understanding the impact on all accounts

## Phase 3 — Rollback

- Revert IAM policy changes to previous version
- Restore SCP settings if they were modified
- Verify DevOps Guru access is restored for authorized users

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
