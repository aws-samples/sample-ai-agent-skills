---
title: "H1 — Compliance Scoring Issues"
description: "Diagnose Security Hub compliance score calculation and accuracy issues"
status: active
severity: MEDIUM
triggers:
  - "compliance score"
  - "security score"
  - "score incorrect"
  - "score not updating"
  - "passing percentage"
owner: devops-agent
objective: "Identify and resolve compliance score calculation and accuracy issues"
context: "The compliance score for each standard is calculated as (passed controls / total enabled controls) × 100. Disabled controls are excluded. Controls with NO_DATA count as not passed. Understanding the calculation helps explain unexpected scores."
---

## Phase 1 — Triage

MUST:
- Check enabled standards and their scores: `aws securityhub get-enabled-standards`
- List controls and their status: `aws securityhub describe-standards-controls --standards-subscription-arn <standard-arn>`
- Count PASSED, FAILED, and NO_DATA controls
- Verify the score calculation: passed / (passed + failed + no_data) × 100
- Check for disabled controls that are excluded from the calculation

SHOULD:
- Identify controls with NO_DATA status (these reduce the score)
- Check if Config is enabled for NO_DATA controls
- Compare the displayed score with the manual calculation

MAY:
- Track score changes over time
- Identify the controls with the most impact on the score

## Phase 2 — Remediate

MUST:
- Fix NO_DATA controls by enabling Config with required resource types
- Remediate FAILED controls to improve the score
- Verify the score updates after remediation

SHOULD:
- Prioritize high-impact controls (those affecting many resources)
- Address NO_DATA controls first as they are often Config dependency issues
- Document the score improvement plan

MAY:
- Create a compliance dashboard with score trends
- Set up alerts for score drops

## Common Issues

- symptoms: "Compliance score lower than expected despite fixing many controls"
  diagnosis: "Controls with NO_DATA status count as not passed, reducing the score significantly."
  resolution: "Enable Config with all resource types to resolve NO_DATA controls. These often represent a large portion of the score."

- symptoms: "Score not updating after remediating a control"
  diagnosis: "Control re-evaluation takes time. The score updates after the underlying Config rule re-evaluates."
  resolution: "Wait for Config rule re-evaluation (up to 24 hours). Force re-evaluation if needed."

- symptoms: "Score improved after disabling failing controls"
  diagnosis: "Disabled controls are excluded from the score calculation, artificially improving the score."
  resolution: "This is expected behavior but does not improve actual security. Re-enable and fix controls for genuine improvement."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix NO_DATA controls | YELLOW | Config changes; reversible |
| Remediate FAILED controls | YELLOW | Resource changes; varies by control |

## Escalation Conditions

- Standard enablement/disablement
- Cross-account configuration changes
- Bulk control remediation

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-enabled-standards` | LOW | Standard scores |
| `describe-standards-controls` | LOW | Control status breakdown |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings
- NEVER disable controls to artificially improve compliance score

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Config change | Revert Config configuration |
| Resource remediation | Revert resource to previous configuration |

## Output Format

```yaml
root_cause: "compliance_scoring — <specific_cause>"
evidence:
  - type: standards
    content: "<standard scores>"
  - type: controls
    content: "<control status breakdown (passed/failed/no_data/disabled)>"
  - type: calculation
    content: "<score calculation verification>"
severity: MEDIUM
mitigation:
  immediate: "Fix NO_DATA and FAILED controls"
  long_term: "Monitor score trends and implement continuous improvement"
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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
