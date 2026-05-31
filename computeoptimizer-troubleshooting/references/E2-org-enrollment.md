---
title: "E2 — Organization Enrollment Issues"
description: "Diagnose organization-wide enrollment"
status: active
severity: MEDIUM
triggers: ["org enrollment", "member account", "organization opt-in"]
owner: devops-agent
objective: "Resolve organization enrollment issues"
context: "Organization enrollment requires management account action."
---

## Phase 1 — Triage

MUST:
- Check org enrollment: `aws compute-optimizer get-enrollment-statuses-for-organization`
- Verify management account permissions
- Check delegated administrator

## Phase 2 — Remediate

MUST:
- Enroll from management account
- Configure delegated administrator if needed

## Common Issues

- symptoms: "Member accounts not enrolled"
  diagnosis: "Organization enrollment not configured."
  resolution: "Enroll from management account with includeMemberAccounts."

## Output Format

```yaml
root_cause: "org_enrollment — <specific_cause>"
evidence:
  - type: org_status
    content: "<organization enrollment>"
severity: MEDIUM
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Organization enrollment is administrative metadata |
| Blast Radius | MEDIUM — Org enrollment affects all member accounts |
| Rollback Complexity | LOW — Can opt out member accounts individually |
| Compliance Scope | LOW — No compliance impact from enrollment |

## Escalation Conditions

- Escalate to organization administrators if member accounts should not be enrolled
- Escalate to management account owner for enrollment authorization

## Data Sensitivity

- **Organization structure** information is exposed during enrollment — handle per org policy
- **Member account IDs** are visible in enrollment status

## PROHIBITED Actions

- Do NOT enroll the organization without management account owner approval
- Do NOT assume all member accounts should be enrolled — some may have restrictions

## Phase 3 — Rollback

- Opt out individual member accounts or the entire organization
- Verify member account enrollment status after changes

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
