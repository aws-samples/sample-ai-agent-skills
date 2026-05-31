---
title: "B1 — Recipe Job Failures"
description: "Diagnose recipe job execution failures"
status: active
severity: HIGH
triggers:
  - "recipe job failed"
  - "job run error"
  - "transformation failed"
owner: devops-agent
objective: "Resolve recipe job execution failures"
context: "Recipe jobs apply transformation steps to input data and write results to output. Failures can be IAM, data format, transformation logic, or output configuration issues."
---

## Phase 1 — Triage

MUST:
- List recent job runs: `aws databrew list-job-runs --name <job-name> --max-results 5`
- Get failed run details: `aws databrew describe-job-run --name <job-name> --run-id <run-id>`
- Check error message in job run output
- Check CloudWatch Logs: `aws logs filter-log-events --log-group-name /aws-glue-databrew/<job-name> --limit 20`

SHOULD:
- Verify job IAM role has required permissions
- Check input data format matches dataset configuration
- Verify output S3 location is writable

## Phase 2 — Remediate

MUST:
- Fix the identified error (IAM, data format, or recipe step)
- Re-run job: `aws databrew start-job-run --name <job-name>`
- Verify successful completion: `aws databrew list-job-runs --name <job-name> --max-results 1`

## Common Issues

- symptoms: "AccessDenied writing to output S3 bucket"
  diagnosis: "Job IAM role lacks s3:PutObject on the output location."
  resolution: "Add s3:PutObject permission for the output S3 path to the job role."

- symptoms: "Job fails on specific transformation step"
  diagnosis: "Data type mismatch or null values in transformation input."
  resolution: "Add a null-handling step before the failing transformation or fix data types."

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
