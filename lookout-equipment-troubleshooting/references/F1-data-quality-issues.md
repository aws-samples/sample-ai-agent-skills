---
title: "F1 — Data Quality Issues"
description: "Diagnose Amazon Lookout for Equipment data quality problems affecting model training and inference"
status: active
severity: MEDIUM
triggers: ["poor data quality", "missing sensor data", "inconsistent timestamps", "data gaps", "noisy data"]
owner: devops-agent
objective: "Identify and resolve data quality issues that degrade model performance"
context: "Data quality directly impacts model accuracy. Common issues include missing values, timestamp gaps, inconsistent sampling rates, and sensor noise."
---

## Phase 1 — Triage

MUST:
- Check dataset status and ingestion history: `aws lookoutequipment describe-dataset --dataset-name <name>`
- List recent ingestion jobs for errors: `aws lookoutequipment list-data-ingestion-jobs --dataset-name <name>`
- Verify data file structure in S3: `aws s3 ls s3://<bucket>/<prefix>/ --recursive`
- Check model training metrics for quality indicators: `aws lookoutequipment describe-model --model-name <name>`

SHOULD:
- Download and inspect sample CSV files for format issues
- Check for missing values (empty cells, NaN, null) in sensor columns
- Verify timestamp consistency and sampling rate across all component files

MAY:
- Use Amazon Athena to query the CSV data for statistical analysis
- Check sensor value ranges for outliers that may indicate calibration issues

## Phase 2 — Remediate

MUST:
- Fill or interpolate missing sensor values using appropriate methods (forward-fill, linear interpolation)
- Fix timestamp gaps by either filling with interpolated data or removing the affected time range
- Ensure consistent sampling rates across all component files
- Remove or fix rows with corrupted or invalid sensor values

SHOULD:
- Implement data validation in the ingestion pipeline before writing to S3
- Set up monitoring for data quality metrics (completeness, consistency, timeliness)
- Document acceptable data quality thresholds for each sensor

MAY:
- Use AWS Glue DataBrew for data profiling and quality assessment
- Implement automated data quality checks with AWS Lambda triggered by S3 events

## Common Issues

- symptoms: "Model training succeeds but produces poor anomaly detection results"
  diagnosis: "Training data contains too many missing values or inconsistent sampling rates, causing the model to learn noise patterns."
  resolution: "Clean the training data: fill missing values, ensure consistent sampling, remove corrupted rows. Reingest and retrain."

- symptoms: "Inference results show anomalies during known-normal operation periods"
  diagnosis: "Inference input data has different quality characteristics than training data (different sampling rate, missing sensors)."
  resolution: "Ensure inference data pipeline produces data with the same quality as training data. Match sampling rates and sensor coverage."

- symptoms: "Data ingestion succeeds but model training fails with data quality error"
  diagnosis: "The data passes format validation but has insufficient variation or too many constant-value sensors."
  resolution: "Remove sensors with constant values. Ensure training data covers normal operational variation. Add more diverse operating condition data."

## Safety Ratings

```
safety_ratings:
  - "describe-dataset: GREEN — read-only metadata"
  - "list-data-ingestion-jobs: GREEN — read-only job listing"
  - "s3 ls: GREEN — read-only file listing"
  - "Data modification in S3: YELLOW — changes source data, may affect other consumers"
  - "Re-ingestion after data fixes: YELLOW — replaces dataset contents"
```

## Escalation Conditions

- Data quality issues originate from upstream sensor hardware failures
- Multiple sensors consistently produce invalid readings requiring physical inspection
- Data pipeline team cannot meet quality requirements within the project timeline
- Data quality degradation is gradual and requires trend analysis to detect

## Data Sensitivity

- Sensor data files: MEDIUM to HIGH — contain proprietary equipment telemetry
- Data quality metrics: LOW — statistical measures without raw values
- Sensor calibration data: MEDIUM — reveals equipment configuration details
- Data pipeline configurations: LOW — technical pipeline settings

## Prohibited Actions

- NEVER delete original sensor data files without creating backups first
- NEVER fabricate sensor values to fill gaps without documenting the interpolation method
- NEVER modify production inference input data without coordinating with the operations team
- NEVER assume data quality issues are resolved without verifying with a test ingestion and model training

## Phase 3 — Rollback

1. If data was incorrectly modified in S3, restore from S3 versioning: `aws s3api list-object-versions --bucket <bucket> --prefix <prefix>`
2. If re-ingestion loaded bad data, the dataset may need to be recreated with clean data
3. If data quality fixes broke the data pipeline, revert pipeline changes and restore original data
4. Document all data transformations applied for audit and reproducibility
5. Notify the data engineering team of any changes to data format or quality requirements

## Output Format

```yaml
root_cause: "data_quality — <specific_cause>"
evidence:
  - type: quality_issue
    content: "<specific data quality problem identified>"
severity: MEDIUM
mitigation:
  immediate: "Fix data quality issues and re-ingest"
  long_term: "Implement automated data quality monitoring in the ingestion pipeline"
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
