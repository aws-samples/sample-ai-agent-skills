---
title: "B2 — Forecast Accuracy"
description: "Diagnose budget and cost forecast inaccuracy"
status: active
severity: MEDIUM
triggers:
  - "forecast inaccurate"
  - "budget forecast wrong"
  - "cost prediction off"
  - "forecast not matching"
owner: devops-agent
objective: "Understand and improve cost forecast accuracy"
context: "Cost Explorer forecasts use historical spending patterns. Accuracy depends on spending consistency. New services, usage changes, or one-time charges reduce accuracy. Forecasts need 2-4 weeks of stable data."
---

## Phase 1 — Triage

MUST:
- Get current forecast: `aws ce get-cost-forecast --time-period Start=<start>,End=<end> --metric UNBLENDED_COST --granularity MONTHLY`
- Compare forecast with actual costs for recent periods
- Check for recent spending pattern changes
- Verify forecast parameters (metric, granularity, filters)

SHOULD:
- Check if new services were recently adopted
- Review one-time charges (RI purchases, support plan changes)
- Analyze spending variability over the past 3 months

MAY:
- Compare forecasts at different granularities (daily vs monthly)
- Check if organizational changes affected spending patterns

## Phase 2 — Remediate

MUST:
- Understand that forecasts are estimates, not guarantees
- Account for known upcoming changes (new projects, decommissions)
- Use forecast confidence intervals for budget planning

SHOULD:
- Exclude one-time charges when evaluating forecast accuracy
- Use budget forecasted alerts in addition to actual alerts
- Review and adjust budgets quarterly based on actual trends

MAY:
- Build custom forecasting using CUR data and external tools
- Implement tagging to improve per-project forecast accuracy

## Common Issues

- symptoms: "Forecast significantly higher than actual"
  diagnosis: "Recent cost reduction (RI purchase, optimization) not yet reflected in baseline."
  resolution: "Wait 2-4 weeks for forecast to adjust to new spending pattern."

- symptoms: "Forecast significantly lower than actual"
  diagnosis: "New services or scaling events not in historical baseline."
  resolution: "Manually adjust budget to account for known upcoming changes."

- symptoms: "Forecast unavailable"
  diagnosis: "Insufficient historical data (less than 2 weeks of spending)."
  resolution: "Wait for sufficient data accumulation. Use manual estimates in the interim."

## Output Format

```yaml
root_cause: "forecast_accuracy — <specific_cause>"
evidence:
  - type: forecast_data
    content: "<forecast vs actual comparison>"
  - type: spending_pattern
    content: "<recent spending trend analysis>"
severity: MEDIUM
mitigation:
  immediate: "Adjust budgets for known changes"
  long_term: "Implement regular budget review and custom forecasting"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Forecasts reveal expected spending patterns |
| Blast Radius | LOW — Forecasts are advisory; no infrastructure changes |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | LOW — Forecasts are planning tools, not compliance controls |

## Escalation Conditions

- Escalate to finance if forecast inaccuracy affects budget planning
- Escalate to management if actual costs significantly exceed forecasts

## Data Sensitivity

- **Forecast data** reveals expected spending — treat as financial planning data
- **Confidence intervals** indicate forecast reliability

## PROHIBITED Actions

- Do NOT present forecasts as guaranteed cost projections to stakeholders
- Do NOT make financial commitments (RI/SP purchases) based solely on forecasts without usage analysis

## Phase 3 — Rollback

- No rollback needed — forecasts are read-only advisory data
- If budget decisions were based on inaccurate forecasts, reassess and adjust budgets

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
