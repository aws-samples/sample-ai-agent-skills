---
title: "A1 — Dataset Import Failures"
description: "Diagnose dataset import job failures"
status: active
severity: HIGH
triggers: ["dataset import failed", "import job error", "CSV import error"]
owner: devops-agent
objective: "Identify and resolve dataset import failures"
context: "Dataset imports load CSV data from S3 into Personalize datasets. Requires matching schema, proper S3 access, and correct CSV format."
---

## Phase 1 — Triage

MUST:
- Check import job: `aws personalize describe-dataset-import-job --dataset-import-job-arn <arn>`
- Review failureReason in job details
- Verify S3 file exists and is accessible
- Check CSV matches dataset schema

SHOULD:
- Verify CSV column order matches schema definition
- Check for encoding issues (UTF-8 required)
- Verify required columns present (USER_ID, ITEM_ID, TIMESTAMP for interactions)

MAY:
- Check S3 bucket policy
- Review import job metrics

## Phase 2 — Remediate

MUST:
- Fix CSV to match schema exactly (column names, order, types)
- Grant S3 read access to Personalize IAM role
- Use UTF-8 encoding without BOM

SHOULD:
- Validate CSV before import
- Use FULL or INCREMENTAL mode appropriately

MAY:
- Implement data validation pipeline

## Common Issues

- symptoms: "Import fails with schema mismatch"
  diagnosis: "CSV columns don't match dataset schema."
  resolution: "Align CSV columns with schema definition. Check column order."

- symptoms: "Import fails with S3 access error"
  diagnosis: "IAM role lacks S3 read permissions."
  resolution: "Add s3:GetObject permission to Personalize service role."

## Output Format

```yaml
root_cause: "dataset_import — <specific_cause>"
evidence:
  - type: import_job
    content: "<import job details>"
severity: HIGH
mitigation:
  immediate: "Fix CSV format or S3 permissions"
  long_term: "Implement data validation before import"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — datasets contain user behavior and interaction data
- Service Disruption Risk: LOW — import failures do not affect running campaigns
- Compliance Impact: HIGH — user data import must comply with privacy regulations

## Escalation Conditions
- Dataset import fails with persistent format errors
- Import job stalls without progress
- Imported data contains PII that should have been anonymized
- S3 access issues prevent import completion
- Import causes dataset group corruption

## Data Sensitivity
- User interaction data — RESTRICTED
- Item metadata — CONFIDENTIAL
- User metadata — RESTRICTED
- S3 bucket paths — INTERNAL
- Never expose user behavioral data in troubleshooting logs

## Prohibited Actions
- Do NOT import datasets containing unmasked PII
- Do NOT import data from untrusted or unvalidated sources
- Do NOT skip data quality validation before import
- Do NOT grant import roles access to unrelated S3 buckets
- Do NOT import data without data governance review
- Do NOT log dataset contents during troubleshooting

## Phase 3 — Rollback
- Delete incorrectly imported datasets
- Restore previous dataset versions
- Revert IAM role permissions added for import
- Verify dataset integrity after rollback
- Re-import with corrected data and configurations
- Document import issues and data quality findings

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
