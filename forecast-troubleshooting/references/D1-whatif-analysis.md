---
title: "D1 â€” What-If Analysis Issues"
description: "Diagnose what-if analysis creation problems"
status: active
severity: MEDIUM
triggers: ["what-if analysis failed", "CreateWhatIfAnalysis error"]
owner: devops-agent
objective: "Identify and resolve what-if analysis issues"
context: "What-if analysis models scenarios by modifying input data. Requires existing forecast as baseline."
---

## Phase 1 â€” Triage

MUST:
- Check analysis: `aws forecast describe-what-if-analysis --what-if-analysis-arn <arn>`
- Verify baseline forecast is ACTIVE
- Review status and failure reason

SHOULD:
- Check time series transformations
- Verify data modifications are valid

MAY:
- Compare with baseline forecast

## Phase 2 â€” Remediate

MUST:
- Ensure baseline forecast is ACTIVE
- Provide valid time series transformations

SHOULD:
- Test with simple scenarios first

MAY:
- Implement scenario management pipeline

## Common Issues

- symptoms: "What-if analysis fails"
  diagnosis: "Baseline forecast not ACTIVE or invalid transformations."
  resolution: "Verify forecast status. Fix transformation definitions."

## Output Format

```yaml
root_cause: "whatif_analysis â€” <specific_cause>"
evidence:
  - type: analysis_status
    content: "<what-if analysis details>"
severity: MEDIUM
mitigation:
  immediate: "Fix baseline forecast or transformations"
  long_term: "Implement scenario analysis pipeline"
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
