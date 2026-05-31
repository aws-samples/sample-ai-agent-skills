---
title: "F1 â€” Time Series Quality Issues"
description: "Diagnose time series data quality problems"
status: active
severity: MEDIUM
triggers: ["data quality", "noisy data", "outliers", "insufficient history"]
owner: devops-agent
objective: "Identify and resolve time series quality issues"
context: "Forecast quality depends on data quality. Sufficient history, consistent frequency, and clean data produce better forecasts."
---

## Phase 1 â€” Triage

MUST:
- Check data volume (2-3x forecast horizon recommended)
- Verify data frequency consistency
- Check for outliers and anomalies
- Review data coverage across items

SHOULD:
- Check for seasonal patterns in data
- Verify data recency
- Analyze data distribution

MAY:
- Compare data quality metrics across items
- Check for data drift

## Phase 2 â€” Remediate

MUST:
- Provide sufficient historical data
- Ensure consistent data frequency
- Handle outliers appropriately

SHOULD:
- Clean data before import
- Include at least 2-3x forecast horizon of history

MAY:
- Implement data quality monitoring

## Common Issues

- symptoms: "Poor forecast accuracy despite sufficient data"
  diagnosis: "Outliers or data quality issues."
  resolution: "Clean outliers. Verify data consistency."

- symptoms: "Training fails with insufficient data"
  diagnosis: "Not enough historical data points."
  resolution: "Add more history. Need 2-3x forecast horizon minimum."

## Output Format

```yaml
root_cause: "timeseries_quality â€” <specific_cause>"
evidence:
  - type: data_quality
    content: "<data quality assessment>"
severity: MEDIUM
mitigation:
  immediate: "Clean data and add more history"
  long_term: "Implement data quality pipeline"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | MEDIUM — Forecast generates predictions used for business planning and resource allocation |
| Data Risk | HIGH — Training datasets may contain sensitive business metrics and demand signals |
| Recovery Complexity | MEDIUM — Predictors can be retrained but require time and compute resources |
| Blast Radius | MEDIUM — Incorrect forecasts can lead to poor business decisions if consumed without validation |

## Escalation Conditions

- Predictor training failures on critical business forecasting pipelines
- Forecast accuracy metrics significantly degraded compared to previous predictor versions
- Dataset import failures blocking time-sensitive forecast generation cycles
- What-if analysis producing unrealistic scenarios affecting planning decisions
- Explainability results showing unexpected feature importance shifts
- Service quota limits blocking forecast generation for business-critical use cases

## Data Sensitivity

- Time series datasets may contain proprietary business metrics (sales, demand, revenue)
- Forecast outputs reveal business projections that may be market-sensitive
- What-if scenarios expose strategic planning assumptions and business variables
- Explainability results reveal which factors drive business outcomes
- Dataset schemas expose business data taxonomy and measurement categories

## PROHIBITED Actions

- NEVER suggest deleting predictors that are actively used for production forecast generation
- NEVER modify dataset schemas without verifying impact on dependent predictors and workflows
- NEVER share forecast outputs containing sensitive business projections without authorization
- NEVER delete historical datasets required for predictor retraining or audit compliance
- NEVER change forecast horizon or granularity on production predictors without stakeholder approval
- NEVER ignore data quality warnings during dataset import for business-critical forecasts

## Phase 3 — Rollback

- If predictor was deleted: retrain using the same dataset group and algorithm configuration
- If dataset import corrupted data: delete the bad import and re-import from verified source
- If forecast parameters were changed: regenerate forecast with previous predictor and parameters
- If what-if analysis produced bad scenarios: delete incorrect analysis and recreate with corrected inputs
- If schema mapping was modified: revert to previous schema and re-import affected datasets
- Document all rollback actions with predictor ARN, dataset group, and forecast generation timestamps
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
