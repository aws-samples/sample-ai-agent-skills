---
title: "C1 — Demand Planning Errors"
description: "Diagnose demand planning and forecasting issues"
status: active
severity: HIGH
triggers: ["demand planning error", "forecast inaccurate", "planning model failure", "demand forecast"]
owner: devops-agent
objective: "Resolve demand planning and forecasting issues in Supply Chain"
context: "Demand planning requires sufficient historical data. Models need weeks to months of data for accurate forecasts."
---

## Phase 1 — Triage

MUST:
- Verify instance is ACTIVE: `aws scn get-instance --instance-id <instance-id>`
- Check data lake datasets for demand data: `aws scn list-data-lake-datasets --instance-id <instance-id> --namespace asc`
- Verify historical data volume and date range coverage

SHOULD:
- Check data integration flow status for demand data sources
- Verify data quality (missing values, outliers, format issues)

MAY:
- Review CloudTrail for planning-related API calls
- Check CloudWatch metrics for planning execution

## Phase 2 — Remediate

MUST:
- Ensure sufficient historical data (3-6 months minimum recommended)
- Fix data quality issues in source datasets
- Re-run data integration flows after fixing source data

SHOULD:
- Implement data quality checks before planning execution
- Document data requirements for demand planning

MAY:
- Set up automated data quality monitoring
- Implement forecast accuracy tracking

## Safety Ratings

```
safety_ratings:
  - "get-instance: GREEN — read-only instance check"
  - "list-data-lake-datasets: GREEN — read-only dataset enumeration"
  - "Data integration re-execution: YELLOW — overwrites existing data lake data"
  - "Planning model retraining: YELLOW — consumes compute resources"
```

## Escalation Conditions

- Demand planning consistently produces inaccurate forecasts despite sufficient data
- Planning model training fails with internal errors
- Data quality issues in source systems cannot be resolved
- Forecast accuracy requirements exceed model capabilities

## Data Sensitivity

- `aws scn get-instance`: LOW — returns instance metadata
- `aws scn list-data-lake-datasets`: LOW — returns dataset metadata
- Demand data: HIGH — contains proprietary sales and demand information
- Forecast results: HIGH — contains business-sensitive predictions

## Prohibited Actions

- NEVER share demand forecasts outside authorized business units
- NEVER modify historical demand data to improve forecast appearance
- NEVER rely on forecasts from models with insufficient training data
- NEVER bypass data quality checks for planning execution

## Phase 3 — Rollback

1. If data integration flows were modified: revert to previous configuration
2. If source data was corrected: verify corrections are accurate before re-ingestion
3. If planning configuration was changed: restore previous settings
4. Verify downstream supply planning is not affected by changes
5. Document all changes for business operations team

## Common Issues

- symptoms: "Demand forecast shows zero or flat predictions"
  diagnosis: "Insufficient historical data or data format issues."
  resolution: "Verify at least 3-6 months of historical demand data is ingested. Check data format matches expected schema."

- symptoms: "Forecast accuracy is very low"
  diagnosis: "Data quality issues or insufficient data variety."
  resolution: "Check for missing values, outliers, and data gaps. Ensure data covers seasonal patterns and trends."

## Output Format

```yaml
root_cause: "demand_planning — <specific_cause>"
evidence:
  - type: planning_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix data quality and re-run planning"
  long_term: "Implement data quality monitoring"
```

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
