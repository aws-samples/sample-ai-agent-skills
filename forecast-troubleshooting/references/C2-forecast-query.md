---
title: "C2 â€” Forecast Query Issues"
description: "Diagnose forecast query problems"
status: active
severity: MEDIUM
triggers: ["QueryForecast error", "no forecast data", "query returns empty"]
owner: devops-agent
objective: "Identify and resolve forecast query issues"
context: "QueryForecast retrieves predictions for specific items. Requires ACTIVE forecast. Returns quantile predictions with timestamps."
---

## Phase 1 â€” Triage

MUST:
- Verify forecast is ACTIVE: `aws forecast describe-forecast --forecast-arn <arn>`
- Test query: `aws forecastquery query-forecast --forecast-arn <arn> --filters '{"item_id":"<id>"}'`
- Verify item_id exists in dataset
- Check filter syntax

SHOULD:
- Test with known item from training data
- Check forecast horizon dates
- Verify quantile types in response

MAY:
- Query multiple items for comparison
- Check for query throttling

## Phase 2 â€” Remediate

MUST:
- Ensure forecast is ACTIVE
- Use valid item_id from dataset
- Use correct filter syntax

SHOULD:
- Handle all quantile predictions in response
- Cache frequently queried forecasts

MAY:
- Implement forecast query API

## Common Issues

- symptoms: "QueryForecast returns empty"
  diagnosis: "Item not in dataset or forecast not ACTIVE."
  resolution: "Verify item exists. Check forecast status."

- symptoms: "ResourceNotFoundException"
  diagnosis: "Forecast ARN incorrect or forecast deleted."
  resolution: "Verify forecast ARN. Check forecast exists."

## Output Format

```yaml
root_cause: "forecast_query â€” <specific_cause>"
evidence:
  - type: query_result
    content: "<QueryForecast response>"
severity: MEDIUM
mitigation:
  immediate: "Fix query parameters"
  long_term: "Implement forecast query caching"
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
