---
title: "D1 — Dataset Creation"
description: "Diagnose dataset creation and configuration issues"
status: active
severity: HIGH
triggers:
  - "dataset creation failed"
  - "cannot create dataset"
  - "dataset format error"
owner: devops-agent
objective: "Resolve dataset creation and format configuration problems"
context: "Datasets define the input data source (S3, JDBC, Data Catalog), format (CSV, JSON, Parquet, Excel), and connection details. Datasets are referenced by projects and jobs."
---

## Phase 1 — Triage

MUST:
- List datasets: `aws databrew list-datasets`
- Describe dataset: `aws databrew describe-dataset --name <dataset-name>`
- Verify S3 path exists: `aws s3 ls <s3-path>`
- Check data format matches dataset configuration

SHOULD:
- Verify file encoding (UTF-8 expected)
- Check CSV delimiter configuration
- Verify JSON structure (array of objects vs JSON lines)

## Phase 2 — Remediate

MUST:
- Create dataset with correct format: `aws databrew create-dataset --name <name> --input '{"s3InputDefinition":{"bucket":"<bucket>","key":"<prefix>"}}' --format-options '{"csv":{"delimiter":",","headerRow":true}}'`

## Common Issues

- symptoms: "Dataset creation succeeds but data preview fails"
  diagnosis: "Format mismatch — data is tab-delimited but configured as comma-delimited."
  resolution: "Update format options to match actual data format."

- symptoms: "S3 path not found"
  diagnosis: "S3 bucket or key prefix doesn't exist or is misspelled."
  resolution: "Verify S3 path with aws s3 ls. Check bucket name and key prefix."

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete datasets that are referenced by active projects or jobs
- Never modify recipe steps on production jobs without testing in a separate project
- Never change output S3 paths for scheduled jobs without updating downstream consumers
- Never delete or overwrite profile job results used for data quality validation
- Never run untested recipe transformations against production data

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Revert recipe changes by republishing the previous recipe version
- Restore dataset configurations from documented state if connections break
- Re-enable previous schedule settings if job scheduling fails after changes

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
  - "NEVER suggest disabling encryption for Glue jobs"
  - "NEVER suggest overly broad Glue service role"
  - "NEVER suggest public S3 access for data catalog"
