---
title: "C1 — Profile Job Failures"
description: "Diagnose profile job execution and statistics generation failures"
status: active
severity: HIGH
triggers:
  - "profile job failed"
  - "profiling error"
  - "statistics not generated"
owner: devops-agent
objective: "Resolve profile job execution failures"
context: "Profile jobs analyze data to produce statistics (null counts, distributions, patterns, correlations). They require read access to input data and write access to output location for results. Profile jobs do NOT transform data."
---

## Phase 1 — Triage

MUST:
- List job runs: `aws databrew list-job-runs --name <profile-job-name> --max-results 5`
- Get run details: `aws databrew describe-job-run --name <profile-job-name> --run-id <run-id>`
- Check job configuration: `aws databrew describe-job --name <profile-job-name>`
- Check CloudWatch Logs for errors

SHOULD:
- Verify IAM role has read access to dataset and write access to output
- Check dataset size vs job timeout settings
- Verify output S3 location exists

## Phase 2 — Remediate

MUST:
- Fix IAM permissions or dataset access issues
- Re-run profile job: `aws databrew start-job-run --name <profile-job-name>`

## Common Issues

- symptoms: "Profile job times out on large dataset"
  diagnosis: "Dataset too large for default timeout or configuration."
  resolution: "Increase job timeout or use sampling to reduce data volume."

- symptoms: "Profile output missing columns"
  diagnosis: "Column configuration excludes certain columns from profiling."
  resolution: "Update profile configuration to include all required columns."

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
