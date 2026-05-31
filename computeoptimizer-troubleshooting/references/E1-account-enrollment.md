---
title: "E1 — Account Enrollment Issues"
description: "Diagnose Compute Optimizer enrollment and opt-in"
status: active
severity: HIGH
triggers: ["not enrolled", "opt-in failed", "enrollment error"]
owner: devops-agent
objective: "Resolve enrollment issues"
context: "Enrollment is required before any recommendations are generated."
---

## Phase 1 — Triage

MUST:
- Check enrollment: `aws compute-optimizer get-enrollment-status`
- Verify IAM permissions for enrollment
- Check for organization-level enrollment

## Phase 2 — Remediate

MUST:
- Enroll: `aws compute-optimizer update-enrollment-status --status Active`
- Fix IAM permissions if enrollment fails

## Common Issues

- symptoms: "get-enrollment-status returns Inactive"
  diagnosis: "Account not enrolled."
  resolution: "Run update-enrollment-status with Active. Wait for recommendations."

## Output Format

```yaml
root_cause: "enrollment — <specific_cause>"
evidence:
  - type: enrollment_status
    content: "<enrollment state>"
severity: HIGH
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Enrollment status is account-level metadata |
| Blast Radius | LOW — Enrollment enables read-only analysis; no infrastructure changes |
| Rollback Complexity | LOW — Can opt out at any time |
| Compliance Scope | LOW — Enrollment does not affect compliance posture |

## Escalation Conditions

- Escalate to account administrators if enrollment fails due to IAM restrictions
- Escalate to organization management if org-level enrollment is required

## Data Sensitivity

- **Enrollment status** is account metadata — low sensitivity
- Enrollment grants Compute Optimizer read access to CloudWatch metrics

## PROHIBITED Actions

- Do NOT enroll accounts without authorization from the account owner
- Do NOT disable enrollment on accounts that depend on recommendations for capacity planning

## Phase 3 — Rollback

- Opt out: `aws compute-optimizer update-enrollment-status --status Inactive`
- Existing recommendations will no longer be updated after opt-out

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
