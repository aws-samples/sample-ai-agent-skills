---
title: "A3 — Insight Quality Issues"
description: "Diagnose insight relevance and recommendation quality"
status: active
severity: LOW
triggers: ["irrelevant insight", "wrong recommendation", "insight noise"]
owner: devops-agent
objective: "Improve insight quality"
context: "Insight quality depends on coverage, data quality, and resource behavior patterns."
---

## Phase 1 — Triage

MUST:
- Review insight details: `aws devops-guru describe-insight --id <id>`
- Check anomaly details within insight
- Review recommendations
- Verify resource context

SHOULD:
- Check for noisy resources generating false insights
- Review insight severity distribution

## Phase 2 — Remediate

MUST:
- Close irrelevant insights with feedback
- Adjust coverage to reduce noise

SHOULD:
- Provide feedback on insight quality
- Review insights regularly

## Common Issues

- symptoms: "Many low-value insights"
  diagnosis: "Coverage too broad or noisy resources."
  resolution: "Narrow coverage. Remove noisy resources. Provide feedback."

## Output Format

```yaml
root_cause: "insight_quality — <specific_cause>"
evidence:
  - type: insight_details
    content: "<insight analysis>"
severity: LOW
mitigation:
  immediate: "Close irrelevant insights"
  long_term: "Tune coverage"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Insight quality tuning is operational |
| Blast Radius | LOW — Quality adjustments affect alerting only |
| Rollback Complexity | LOW — Coverage changes can be reverted |
| Compliance Scope | LOW — Insight quality is an operational concern |

## Escalation Conditions

- Escalate to operations if insight noise overwhelms the team
- Escalate to application owners to validate insight relevance

## Data Sensitivity

- **Insight feedback** is operational metadata
- **Coverage adjustments** are operational settings

## PROHIBITED Actions

- Do NOT remove critical resources from coverage to reduce noise
- Do NOT close insights without providing feedback to improve the model

## Phase 3 — Rollback

- Restore previous coverage scope if it was narrowed too aggressively
- Reopen insights that were incorrectly closed

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
