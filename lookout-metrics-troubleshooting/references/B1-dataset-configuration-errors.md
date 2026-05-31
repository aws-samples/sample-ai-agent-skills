---
title: "B1 — Dataset Configuration Errors"
description: "Diagnose and resolve metric set and dataset configuration issues"
status: active
severity: HIGH
triggers: ["metric set error", "dataset configuration failed", "invalid schema", "timestamp format error"]
owner: devops-agent
objective: "Correct metric set configuration to enable proper data ingestion"
context: "Metric sets define measures, dimensions, timestamps, and data source connections. Configuration errors are the most common cause of detector failures. Timestamp format mismatches and schema issues account for over 60% of dataset problems."
---

## Phase 1 — Triage

MUST:
- Describe the metric set: `aws lookoutmetrics describe-metric-set --metric-set-arn <arn>`
- Verify timestamp format matches source data format exactly
- Check measure columns are numeric and dimension columns are strings

SHOULD:
- Validate the metric set against a sample data file
- Check data quality metrics: `aws lookoutmetrics get-data-quality-metrics --anomaly-detector-arn <arn>`
- Review the metric set frequency matches data delivery cadence

MAY:
- Compare metric set schema against the actual data source schema
- Check for encoding issues in CSV/JSON data files

## Phase 2 — Remediate

MUST:
- Fix timestamp format to match source data exactly (e.g., `yyyy-MM-dd HH:mm:ss` vs `yyyyMMdd'T'HHmmss`)
- Correct measure and dimension column mappings
- Ensure the metric set frequency aligns with data availability

SHOULD:
- Deactivate detector before modifying metric set: `aws lookoutmetrics deactivate-anomaly-detector --anomaly-detector-arn <arn>`
- Update metric set: `aws lookoutmetrics update-metric-set --metric-set-arn <arn> --timestamp-column '{"ColumnName":"timestamp","ColumnFormat":"yyyy-MM-dd HH:mm:ss"}'`

MAY:
- Create a new metric set if the existing one has fundamental schema issues
- Add offset configuration if data arrives with a delay

## Safety Ratings

```
safety_ratings:
  - "describe-metric-set: GREEN — read-only configuration check"
  - "update-metric-set: YELLOW — modifies configuration, requires detector deactivation"
  - "deactivate-anomaly-detector: YELLOW — stops detection, resets learning period"
  - "get-data-quality-metrics: GREEN — read-only quality check"
```

## Escalation Conditions

- Metric set update fails with ValidationException despite correct parameters
- Timestamp format cannot be determined from the source data
- Data quality metrics show persistent issues after configuration fixes
- Multiple metric sets in the same detector have conflicting configurations

## Data Sensitivity

- `aws lookoutmetrics describe-metric-set`: MEDIUM — contains data source details and schema
- Metric set configuration: MEDIUM — reveals business metric names and data structure
- Sample data files: HIGH — may contain actual business metrics and values
- Data quality metrics: LOW — statistical summaries only

## Prohibited Actions

- NEVER modify a metric set on an active production detector without stakeholder approval
- NEVER delete a metric set that has historical anomaly data without backing up results
- NEVER configure measures on non-numeric columns — this causes silent data loss
- NEVER assume timestamp format without verifying against actual source data

## Phase 3 — Rollback

1. If metric set was modified incorrectly, revert to previous configuration using saved settings
2. If detector was deactivated for metric set changes, reactivate after confirming correct configuration
3. If a new metric set was created in error, delete it: `aws lookoutmetrics delete-anomaly-detector --anomaly-detector-arn <arn>` (only if detector is not needed)
4. Document all configuration changes for audit trail
5. Notify stakeholders about learning period reset if detector was deactivated

## Common Issues

- symptoms: "Detector shows zero data points after activation"
  diagnosis: "Timestamp format mismatch between metric set configuration and source data."
  resolution: "Check actual timestamp format in source data. Update metric set with correct format string."

- symptoms: "ValidationException when creating metric set"
  diagnosis: "Invalid column names, unsupported timestamp format, or exceeding quota limits."
  resolution: "Verify column names match source data exactly. Check timestamp format is supported. Ensure within 5 measures and 5 dimensions limit."

## Output Format

```yaml
root_cause: "dataset_configuration — <specific_cause>"
evidence:
  - type: configuration_error
    content: "<metric set details and mismatch>"
severity: HIGH
mitigation:
  immediate: "Fix metric set configuration"
  long_term: "Implement schema validation in data pipeline"
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
