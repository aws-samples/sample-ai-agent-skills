---
title: "A2 — Data Ingestion Errors"
description: "Diagnose Amazon Lookout for Equipment data ingestion failures"
status: active
severity: HIGH
triggers: ["ingestion failed", "data ingestion error", "StartDataIngestionJob failed", "ingestion job stuck"]
owner: devops-agent
objective: "Resolve data ingestion failures and ensure sensor data is properly loaded into datasets"
context: "Data ingestion loads CSV sensor data from S3 into a dataset. Data must follow strict format requirements including timestamps, column structure, and file organization."
---

## Phase 1 — Triage

MUST:
- Check ingestion job status: `aws lookoutequipment describe-data-ingestion-job --job-id <job-id>`
- List recent ingestion jobs: `aws lookoutequipment list-data-ingestion-jobs --dataset-name <name>`
- Verify S3 data path exists and contains CSV files: `aws s3 ls s3://<bucket>/<prefix>/ --recursive`
- Verify the IAM role can access the S3 bucket: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::<bucket>/<prefix>/*`

SHOULD:
- Check that CSV files have correct timestamp format (ISO 8601)
- Verify each component has its own subfolder with matching CSV files
- Check S3 bucket is in the same region as the dataset

MAY:
- Download and inspect a sample CSV file for format issues
- Check CloudWatch Logs for detailed ingestion error messages

## Phase 2 — Remediate

MUST:
- Fix CSV format: timestamps as first column in ISO 8601, consistent column headers
- Ensure one CSV file per component in a named subfolder matching the schema component name
- Fix IAM role trust policy to allow lookoutequipment.amazonaws.com to assume the role
- Ensure S3 bucket policy does not deny access to the Lookout for Equipment service role

SHOULD:
- Validate data completeness — minimum 14 days of continuous data for training
- Remove duplicate timestamps and ensure monotonically increasing order
- Verify sampling rate consistency within each component file

MAY:
- Use S3 event notifications to trigger validation before ingestion
- Implement pre-ingestion data quality checks in a Lambda function

## Common Issues

- symptoms: "Ingestion job status is FAILED with 'Unable to access S3' error"
  diagnosis: "The IAM role cannot read from the specified S3 path. Either the role lacks s3:GetObject permission or the trust policy is missing."
  resolution: "Update the IAM role with s3:GetObject, s3:ListBucket, s3:GetBucketLocation permissions. Add lookoutequipment.amazonaws.com to the trust policy."

- symptoms: "Ingestion job fails with schema mismatch error"
  diagnosis: "CSV column headers do not match the dataset schema definition."
  resolution: "Compare CSV headers against the dataset schema. Column names must match exactly including case sensitivity."

- symptoms: "Ingestion job stays in IN_PROGRESS for hours"
  diagnosis: "Large datasets take time to ingest. This is normal for datasets with millions of rows."
  resolution: "Wait for completion. Check status periodically. Ingestion of very large datasets can take several hours."

## Safety Ratings

```
safety_ratings:
  - "describe-data-ingestion-job: GREEN — read-only status check"
  - "list-data-ingestion-jobs: GREEN — read-only enumeration"
  - "start-data-ingestion-job: YELLOW — initiates data processing, consumes compute resources"
  - "s3 ls: GREEN — read-only listing of S3 objects"
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
```

## Escalation Conditions

- Ingestion fails repeatedly with internal service errors after data format fixes
- Ingestion job stuck in IN_PROGRESS for more than 24 hours
- S3 access issues persist despite correct IAM configuration (possible VPC endpoint or bucket policy conflict)
- Data format requirements are unclear for complex multi-component equipment

## Data Sensitivity

- `aws lookoutequipment describe-data-ingestion-job`: LOW — returns job metadata and status
- `aws s3 ls`: MEDIUM — reveals S3 object paths which may contain environment or project names
- Sensor CSV data: MEDIUM to HIGH — may contain proprietary equipment telemetry data
- `aws iam simulate-principal-policy`: LOW — tests permissions without making changes

## Prohibited Actions

- NEVER modify or delete source CSV files in S3 during an active ingestion job
- NEVER suggest ingesting data with fewer than 14 days of timestamps for training purposes
- NEVER change the dataset schema after initial ingestion — schemas are immutable
- NEVER expose raw sensor data values in troubleshooting logs without confirming data classification

## Phase 3 — Rollback

1. If ingestion loaded incorrect data, the dataset may need to be deleted and recreated (no selective data removal)
2. If the IAM role was modified, restore the previous role policy from version history: `aws iam list-role-policies --role-name <name>`
3. If S3 data was accidentally modified, restore from S3 versioning: `aws s3api list-object-versions --bucket <bucket> --prefix <prefix>`
4. Document the correct data format and S3 structure for future ingestion jobs
5. Notify the data pipeline team to pause upstream data writes during troubleshooting

## Output Format

```yaml
root_cause: "data_ingestion — <specific_cause>"
evidence:
  - type: ingestion_error
    content: "<error details from describe-data-ingestion-job>"
severity: HIGH
mitigation:
  immediate: "Fix data format or access issues and re-run ingestion"
  long_term: "Implement automated data validation before ingestion"
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
