---
title: "E2 — Region-Specific Controls"
description: "Diagnose region-specific control issues"
status: active
severity: LOW
triggers: ["region-specific", "control in specific region", "regional controls"]
owner: devops-agent
objective: "Manage region-specific control requirements"
context: "Some controls are region-specific due to service availability or regulatory requirements."
---

## Phase 1 — Triage

MUST:
- Identify which controls are needed per region
- Check control availability in each target region
- Verify Control Tower is set up in target regions

SHOULD:
- Map regional compliance requirements to available controls
- Identify gaps per region

MAY:
- Create region-specific control implementation plans

## Phase 2 — Remediate

MUST:
- Implement available controls per region
- Use alternative controls for gaps

SHOULD:
- Document regional control strategy

MAY:
- Use Config rules as alternatives for unavailable catalog controls

## Common Issues

- symptoms: "Need same control in all regions but only available in some"
  diagnosis: "Control has limited regional availability."
  resolution: "Implement catalog control where available. Use Config custom rules for other regions."

## Output Format

```yaml
root_cause: "region_specific — <specific_cause>"
evidence:
  - type: regional_analysis
    content: "<per-region control availability>"
severity: LOW
mitigation:
  immediate: "Implement available controls"
  long_term: "Maintain regional control strategy"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Region-specific control data is public reference information |
| Blast Radius | MEDIUM — Missing region-specific controls create compliance gaps |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | HIGH — Region-specific regulatory requirements may mandate specific controls |

## Escalation Conditions

- Escalate to compliance if regulatory requirements in a region cannot be met with available controls
- Escalate to security architects for alternative control design

## Data Sensitivity

- **Regional control strategy** documents may be organization-sensitive
- **Regulatory requirements** per region should be handled per compliance policy

## PROHIBITED Actions

- Do NOT operate in regulated regions without documenting control coverage
- Do NOT assume Config rules are equivalent to catalog controls without validation

## Phase 3 — Rollback

- No rollback needed — read-only analysis
- If alternative controls were implemented incorrectly, revert and redesign

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
