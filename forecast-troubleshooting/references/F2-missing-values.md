---
title: "F2 â€” Missing Values Issues"
description: "Diagnose missing value handling problems"
status: active
severity: MEDIUM
triggers: ["missing values", "gaps in data", "fill strategy"]
owner: devops-agent
objective: "Identify and resolve missing value issues"
context: "Missing values in time series reduce accuracy. Forecast can fill gaps using frontfill, backfill, middlefill, or value strategies. Excessive gaps degrade quality."
---

## Phase 1 â€” Triage

MUST:
- Check for gaps in time series data
- Review fill strategy configuration in predictor
- Assess percentage of missing values
- Check if missing values are random or systematic

SHOULD:
- Identify items with most missing values
- Check if gaps align with known events
- Review fill strategy impact on accuracy

MAY:
- Compare accuracy with different fill strategies
- Analyze missing value patterns

## Phase 2 â€” Remediate

MUST:
- Choose appropriate fill strategy (frontfill, backfill, middlefill, value)
- Minimize missing values in source data
- Handle systematic gaps differently from random gaps

SHOULD:
- Use domain knowledge for fill strategy selection
- Pre-fill data before import when possible

MAY:
- Implement missing value detection and reporting

## Common Issues

- symptoms: "Forecast shows unexpected spikes or drops"
  diagnosis: "Missing values filled with inappropriate strategy."
  resolution: "Change fill strategy. Pre-fill data with domain-appropriate values."

- symptoms: "Training fails due to excessive missing values"
  diagnosis: "Too many gaps in time series."
  resolution: "Fill gaps before import. Remove items with excessive missing data."

## Output Format

```yaml
root_cause: "missing_values â€” <specific_cause>"
evidence:
  - type: data_gaps
    content: "<missing value analysis>"
severity: MEDIUM
mitigation:
  immediate: "Fix fill strategy or pre-fill data"
  long_term: "Implement data completeness monitoring"
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
