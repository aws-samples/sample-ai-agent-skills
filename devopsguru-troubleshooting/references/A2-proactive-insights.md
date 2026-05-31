---
title: "A2 — Proactive Insight Issues"
description: "Diagnose proactive insight generation and quality"
status: active
severity: MEDIUM
triggers: ["no proactive insights", "prediction wrong", "proactive not working"]
owner: devops-agent
objective: "Resolve proactive insight issues"
context: "Proactive insights predict future issues. Require weeks of historical data."
---

## Phase 1 — Triage

MUST:
- List proactive insights: `aws devops-guru list-insights --status-filter ONGOING --type PROACTIVE`
- Verify sufficient historical data (weeks)
- Check resource coverage

SHOULD:
- Review proactive insight accuracy
- Check for seasonal patterns

## Phase 2 — Remediate

MUST:
- Ensure sufficient data history
- Verify coverage includes target resources

## Common Issues

- symptoms: "No proactive insights generated"
  diagnosis: "Insufficient historical data."
  resolution: "Wait for weeks of data collection. Verify coverage."

## Output Format

```yaml
root_cause: "proactive_insights — <specific_cause>"
evidence:
  - type: insight_status
    content: "<proactive insight details>"
severity: MEDIUM
mitigation:
  immediate: "Wait for data accumulation"
  long_term: "Monitor prediction accuracy"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Proactive insights are predictive analysis |
| Blast Radius | LOW — Predictions are advisory only |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | LOW — Proactive insights are operational tools |

## Escalation Conditions

- Escalate to operations if proactive insights predict imminent service issues
- Escalate to capacity planning if predictions indicate resource exhaustion

## Data Sensitivity

- **Proactive predictions** are operational forecasts — low sensitivity
- **Resource utilization trends** are standard metrics

## PROHIBITED Actions

- Do NOT ignore proactive insights predicting critical resource exhaustion
- Do NOT treat predictions as guaranteed outcomes — they are estimates

## Phase 3 — Rollback

- No rollback needed — proactive insights are advisory
- If actions were taken based on incorrect predictions, revert those actions

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
