---
title: "F2 — Recommendation Issues"
description: "Diagnose insight recommendation quality and applicability"
status: active
severity: LOW
triggers: ["recommendation unclear", "wrong recommendation", "how to implement"]
owner: devops-agent
objective: "Resolve recommendation issues"
context: "Recommendations are contextual to specific insights."
---

## Phase 1 — Triage

MUST:
- List recommendations: `aws devops-guru list-recommendations --insight-id <id>`
- Review recommendation context
- Check related anomalies

## Phase 2 — Remediate

MUST:
- Follow recommendations in context of the insight
- Verify fix addresses root cause

## Common Issues

- symptoms: "Recommendation doesn't seem relevant"
  diagnosis: "Recommendation context not fully understood."
  resolution: "Review all anomalies in the insight. Recommendation addresses the combined pattern."

## Output Format

```yaml
root_cause: "recommendations — <specific_cause>"
evidence:
  - type: recommendation_details
    content: "<recommendation analysis>"
severity: LOW
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Recommendations are operational guidance |
| Blast Radius | LOW — Recommendations are advisory only |
| Rollback Complexity | LOW — No changes from viewing recommendations |
| Compliance Scope | LOW — Recommendations are operational tools |

## Escalation Conditions

- Escalate to application owners if recommendations suggest significant changes
- Escalate to operations if recommendations address active incidents

## Data Sensitivity

- **Recommendation details** are operational guidance — low sensitivity

## PROHIBITED Actions

- Do NOT apply recommendations to production without testing
- Do NOT ignore recommendations for active incidents

## Phase 3 — Rollback

- If actions were taken based on recommendations and caused issues, revert those specific changes
- Document which recommendations were applied and their outcomes

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
