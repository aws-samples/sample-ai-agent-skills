---
title: "E1 — S3 Data Source"
description: "Diagnose and resolve S3 data source ingestion issues for Lookout for Metrics"
status: active
severity: HIGH
triggers: ["S3 data not ingested", "S3 path template error", "S3 data source failed", "no data from S3"]
owner: devops-agent
objective: "Ensure reliable data ingestion from S3 data sources into Lookout for Metrics detectors"
context: "S3 data sources require specific folder structures with date-partitioned paths, CSV or JSON Lines format, and proper IAM permissions. Path template mismatches and timestamp format errors are the most common issues."
---

## Phase 1 — Triage

MUST:
- Describe metric set to get S3 configuration: `aws lookoutmetrics describe-metric-set --metric-set-arn <arn>`
- Verify S3 bucket and prefix exist: `aws s3 ls s3://<bucket>/<prefix>/`
- Check that data files exist in the expected date-partitioned folders: `aws s3 ls s3://<bucket>/<prefix>/<date_partition>/`

SHOULD:
- Verify the path template matches actual folder structure
- Check data file format (CSV or JSON Lines) and encoding
- Verify data quality metrics: `aws lookoutmetrics get-data-quality-metrics --anomaly-detector-arn <arn>`

MAY:
- Download and inspect a sample data file for schema correctness
- Check S3 bucket versioning and lifecycle policies that might affect data availability

## Phase 2 — Remediate

MUST:
- Fix S3 path template to match actual folder structure (e.g., `s3://bucket/prefix/{{yyyyMMdd}}/{{HHmm}}/`)
- Ensure data files are in CSV or JSON Lines format with correct column names
- Fix IAM role permissions: `s3:GetObject` and `s3:ListBucket` on the bucket and prefix

SHOULD:
- Verify timestamp column values in data files match the configured timestamp format
- Ensure data files are delivered before the detector's evaluation window
- Check for S3 bucket policy restrictions or KMS encryption requirements

MAY:
- Set up S3 event notifications to monitor data delivery
- Implement data validation in the ETL pipeline before writing to S3

## Safety Ratings

```
safety_ratings:
  - "s3 ls: GREEN — read-only bucket listing"
  - "describe-metric-set: GREEN — read-only configuration check"
  - "get-data-quality-metrics: GREEN — read-only quality check"
  - "S3 bucket policy modification: YELLOW — affects access control"
```

## Escalation Conditions

- S3 data files are present and correctly formatted but detector still shows no data
- S3 bucket is in a different account and cross-account access cannot be configured
- KMS encryption on S3 objects prevents Lookout for Metrics from reading data
- Data pipeline delivers files after the detector's evaluation window consistently

## Data Sensitivity

- `aws s3 ls`: MEDIUM — reveals bucket contents and folder structure
- S3 data files: HIGH — contain actual business metrics and values
- `aws lookoutmetrics describe-metric-set`: MEDIUM — contains S3 path and schema details
- S3 bucket policy: LOW — access control configuration only

## Prohibited Actions

- NEVER grant `s3:*` permissions to the Lookout for Metrics execution role
- NEVER modify S3 bucket policies without understanding impact on other consumers
- NEVER delete S3 data files during troubleshooting — they may be needed for reprocessing
- NEVER expose S3 data file contents containing sensitive business metrics in logs

## Phase 3 — Rollback

1. If S3 path template was changed incorrectly, revert metric set configuration (requires detector deactivation)
2. If S3 bucket policy was modified, revert to previous policy
3. If IAM permissions were broadened, revert to least-privilege after resolution
4. Document all S3 configuration changes and their impact
5. Verify data ingestion resumes by checking data quality metrics after fixes

## Common Issues

- symptoms: "Detector shows zero data points despite files in S3"
  diagnosis: "S3 path template does not match actual folder structure. Common: missing trailing slash, wrong date format."
  resolution: "Compare configured path template with actual S3 folder names. Fix template to match exactly."

- symptoms: "Intermittent data gaps in detection results"
  diagnosis: "Data files arriving after the detector's evaluation window."
  resolution: "Adjust data pipeline timing or configure offset in metric set to allow for delivery delay."

## Output Format

```yaml
root_cause: "s3_data_source — <specific_cause>"
evidence:
  - type: s3_configuration
    content: "<path template, actual S3 structure, and data format details>"
severity: HIGH
mitigation:
  immediate: "Fix S3 path template or data format"
  long_term: "Implement S3 data delivery monitoring and validation"
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
