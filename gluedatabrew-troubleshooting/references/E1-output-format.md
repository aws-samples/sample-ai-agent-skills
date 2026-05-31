---
title: "E1 — Output Format"
description: "Diagnose output format and compression configuration issues"
status: active
severity: MEDIUM
triggers:
  - "output format error"
  - "wrong output format"
  - "compression issue"
  - "output file corrupt"
owner: devops-agent
objective: "Resolve output format and compression configuration problems"
context: "Job output configuration includes format (CSV, JSON, Parquet, Avro, ORC, XML, GLUEPARQUET), compression (GZIP, SNAPPY, BZIP2, etc.), and S3 location. Each job configures output independently."
---

## Phase 1 — Triage

MUST:
- Describe job: `aws databrew describe-job --name <job-name>`
- Check output configuration (format, compression, S3 location)
- Verify output files: `aws s3 ls <output-s3-path>`
- Check job run for output-related errors

SHOULD:
- Verify downstream systems can read the output format
- Check file size and partitioning settings

## Phase 2 — Remediate

MUST:
- Update job output: `aws databrew update-recipe-job --name <name> --outputs '[{"location":{"bucket":"<bucket>","key":"<prefix>"},"format":"PARQUET","compression-format":"SNAPPY","overwrite":true}]'`

## Common Issues

- symptoms: "Output files unreadable by downstream system"
  diagnosis: "Format or compression mismatch with consumer expectations."
  resolution: "Change output format/compression to match downstream requirements."

- symptoms: "Output overwrites previous results"
  diagnosis: "Overwrite is set to true without partitioning by date."
  resolution: "Use dynamic output paths with date partitions or set overwrite to false."

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
