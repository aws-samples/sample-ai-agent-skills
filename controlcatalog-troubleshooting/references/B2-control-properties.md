---
title: "B2 — Control Properties"
description: "Diagnose control property interpretation issues"
status: active
severity: LOW
triggers: ["control behavior unclear", "control properties", "what does control do"]
owner: devops-agent
objective: "Interpret control properties correctly"
context: "Control properties include behavior type (preventive, detective, proactive), implementation details, and framework mappings."
---

## Phase 1 — Triage

MUST:
- Get control details: `aws controlcatalog get-control --control-arn <arn>`
- Review behavior type (PREVENTIVE, DETECTIVE, PROACTIVE)
- Check control description and implementation guidance

SHOULD:
- Verify control maps to your compliance requirements
- Check regional availability

MAY:
- Compare with similar controls in the catalog

## Phase 2 — Remediate

MUST:
- Understand control behavior before implementation
- Verify control is available in target region

SHOULD:
- Map controls to compliance framework requirements
- Document control selection rationale

MAY:
- Create control implementation plan

## Common Issues

- symptoms: "Unclear if control is preventive or detective"
  diagnosis: "Control behavior type not checked."
  resolution: "Check behavior field in get-control response. Preventive blocks actions, detective reports violations."

## Output Format

```yaml
root_cause: "control_properties — <specific_cause>"
evidence:
  - type: control_info
    content: "<control properties and behavior>"
severity: LOW
mitigation:
  immediate: "Review control properties"
  long_term: "Document control selection criteria"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Control properties are public reference data |
| Blast Radius | LOW — Read-only interpretation; no changes |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | HIGH — Misinterpreting control behavior types can lead to compliance gaps |

## Escalation Conditions

- Escalate to compliance team if control behavior type is unclear for a compliance requirement
- Escalate to security architects if preventive vs detective control selection affects security posture

## Data Sensitivity

- **Control behavior types** (preventive, detective, proactive) are public reference data
- No sensitive data involved

## PROHIBITED Actions

- Do NOT implement a detective control where a preventive control is required by the compliance framework
- Do NOT assume control behavior without checking the behavior field in the API response

## Phase 3 — Rollback

- No rollback needed — read-only interpretation
- If controls were implemented based on incorrect behavior understanding, reassess and correct

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
