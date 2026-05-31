---
title: "E1 — Regional Availability"
description: "Diagnose control regional availability issues"
status: active
severity: MEDIUM
triggers: ["control not available in region", "regional support", "region-specific"]
owner: devops-agent
objective: "Resolve regional availability issues"
context: "Not all controls are available in all regions. Check control details for regional support before implementation."
---

## Phase 1 — Triage

MUST:
- Get control details: `aws controlcatalog get-control --control-arn <arn>`
- Check regional availability in control properties
- Verify target region is supported

SHOULD:
- List controls available in specific region
- Check if alternative controls exist for unsupported regions

MAY:
- Compare regional availability across controls

## Phase 2 — Remediate

MUST:
- Use controls available in your target region
- Find alternative controls for unsupported regions

SHOULD:
- Document regional control availability for your environment

MAY:
- Request control availability in additional regions via AWS support

## Common Issues

- symptoms: "Control available in us-east-1 but not in eu-west-1"
  diagnosis: "Control has limited regional availability."
  resolution: "Check control details for supported regions. Use alternative controls or Config rules for unsupported regions."

## Output Format

```yaml
root_cause: "regional_availability — <specific_cause>"
evidence:
  - type: availability_details
    content: "<control regional availability>"
severity: MEDIUM
mitigation:
  immediate: "Use available controls"
  long_term: "Plan for regional control gaps"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Regional availability is public reference data |
| Blast Radius | MEDIUM — Missing controls in a region creates compliance gaps |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | HIGH — Regional control gaps affect compliance in those regions |

## Escalation Conditions

- Escalate to compliance team if required controls are unavailable in a target region
- Escalate to security architects to design alternative controls for unsupported regions

## Data Sensitivity

- **Regional availability data** is public reference information
- **Regional deployment plans** may be organization-sensitive

## PROHIBITED Actions

- Do NOT deploy workloads to regions without verifying required controls are available
- Do NOT assume controls available in one region are available in all regions

## Phase 3 — Rollback

- No rollback needed — read-only query
- If workloads were deployed to regions without required controls, implement alternative controls or migrate

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
