---
title: "D2 — Insights Generation"
description: "Diagnose Supply Chain insights and recommendation issues"
status: active
severity: MEDIUM
triggers: ["no insights generated", "insights inaccurate", "recommendations missing"]
owner: devops-agent
objective: "Resolve insights generation issues in Supply Chain"
context: "Insights require complete data ingestion and processing. They are generated asynchronously after data analysis."
---

## Phase 1 — Triage

MUST:
- Verify data lake datasets are populated: `aws scn list-data-lake-datasets --instance-id <instance-id> --namespace asc`
- Check data integration flow completion status
- Verify instance is ACTIVE: `aws scn get-instance --instance-id <instance-id>`

SHOULD:
- Check data completeness across required datasets
- Verify data freshness (last update timestamps)

MAY:
- Review CloudTrail for insights-related API calls
- Check CloudWatch metrics for processing status

## Phase 2 — Remediate

MUST:
- Ensure all required datasets are populated with sufficient data
- Fix data quality issues that prevent insight generation
- Allow processing time after data ingestion

SHOULD:
- Document minimum data requirements for each insight type
- Implement data completeness monitoring

MAY:
- Set up notifications for new insights
- Build insight tracking dashboards

## Safety Ratings

```
safety_ratings:
  - "list-data-lake-datasets: GREEN — read-only dataset enumeration"
  - "get-instance: GREEN — read-only instance check"
  - "Data re-ingestion: YELLOW — updates data lake contents"
  - "Insight configuration changes: YELLOW — affects recommendation generation"
```

## Escalation Conditions

- No insights generated despite complete data ingestion
- Insights consistently contradict known business conditions
- Processing time exceeds expected duration
- Insight generation blocked by service issues

## Data Sensitivity

- Insights and recommendations: HIGH — contain business-sensitive analysis
- Data lake contents: HIGH — proprietary supply chain data
- Processing metrics: LOW — operational metadata

## Prohibited Actions

- NEVER share insights outside authorized business units
- NEVER modify data to influence insight generation
- NEVER rely on insights from incomplete data
- NEVER dismiss insights without investigation

## Phase 3 — Rollback

1. If data was re-ingested incorrectly: revert data integration flows
2. If insight configuration was changed: restore previous settings
3. If processing was interrupted: allow it to complete naturally
4. Verify insights are consistent after data changes
5. Document investigation for business operations team

## Common Issues

- symptoms: "No insights generated after data ingestion"
  diagnosis: "Insufficient data or processing still in progress."
  resolution: "Verify all required datasets are populated. Allow processing time (hours to days for initial insights)."

- symptoms: "Insights seem outdated or stale"
  diagnosis: "Data integration flows have not run recently."
  resolution: "Check flow execution status and frequency. Re-run flows to refresh data."

## Output Format

```yaml
root_cause: "insights_generation — <specific_cause>"
evidence:
  - type: insights_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix data completeness"
  long_term: "Implement data freshness monitoring"
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
