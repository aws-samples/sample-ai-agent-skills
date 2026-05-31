---
title: "B1 — Control Details"
description: "Diagnose control detail retrieval issues"
status: active
severity: MEDIUM
triggers: ["cannot get control", "control not found", "get-control error"]
owner: devops-agent
objective: "Resolve control detail retrieval issues"
context: "Control details include description, behavior, regional availability, and objective mappings."
---

## Phase 1 — Triage

MUST:
- Try getting control: `aws controlcatalog get-control --control-arn <arn>`
- Verify control ARN format is correct
- Check IAM permissions for controlcatalog:GetControl

SHOULD:
- Verify control exists by listing controls first
- Check if control ARN is from the correct region

MAY:
- Compare with known valid control ARNs

## Phase 2 — Remediate

MUST:
- Fix control ARN format
- Add controlcatalog:GetControl permission

SHOULD:
- List controls first to get valid ARNs

MAY:
- Cache control details for repeated access

## Common Issues

- symptoms: "get-control returns ResourceNotFoundException"
  diagnosis: "Control ARN is invalid or control doesn't exist."
  resolution: "List controls to get valid ARNs: aws controlcatalog list-controls."

## Output Format

```yaml
root_cause: "control_details — <specific_cause>"
evidence:
  - type: api_error
    content: "<get-control error>"
severity: MEDIUM
mitigation:
  immediate: "Fix control ARN"
  long_term: "Maintain control ARN inventory"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Control details are public AWS reference data |
| Blast Radius | LOW — Read-only API |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | MEDIUM — Control details inform compliance implementation decisions |

## Escalation Conditions

- Escalate to compliance team if control details are needed for audit deadlines
- Escalate to AWS Support if specific controls return ResourceNotFoundException unexpectedly

## Data Sensitivity

- **Control details** including descriptions and implementation guidance are public reference data
- **Control ARNs** are stable identifiers — low sensitivity

## PROHIBITED Actions

- Do NOT implement controls based solely on catalog descriptions without reviewing full AWS documentation
- Do NOT cache control details indefinitely — refresh periodically for updates

## Phase 3 — Rollback

- No rollback needed — read-only API
- If control implementation was based on incorrect details, review and correct the implementation

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
