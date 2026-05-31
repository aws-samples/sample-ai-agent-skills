---
title: "D2 — Dataset Connection Issues"
description: "Diagnose dataset connection and access problems"
status: active
severity: HIGH
triggers:
  - "dataset connection failed"
  - "cannot access data source"
  - "JDBC connection error"
  - "Data Catalog access denied"
owner: devops-agent
objective: "Resolve dataset connection and data access issues"
context: "Datasets can connect to S3, JDBC databases (via Glue connections), or Glue Data Catalog tables. Connection issues involve network access, credentials, or permissions."
---

## Phase 1 — Triage

MUST:
- Describe dataset: `aws databrew describe-dataset --name <dataset-name>`
- For S3: verify bucket access: `aws s3 ls <bucket>/<prefix>`
- For Data Catalog: verify table: `aws glue get-table --database-name <db> --name <table>`
- Check IAM role permissions for the data source

SHOULD:
- For JDBC: verify Glue connection: `aws glue get-connection --name <connection-name>`
- Check VPC/subnet configuration for database connections
- Verify security group allows outbound to database port

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for the data source
- For JDBC: fix connection credentials or network configuration
- For S3: ensure bucket policy allows access from the DataBrew role

## Common Issues

- symptoms: "AccessDenied reading S3 data"
  diagnosis: "DataBrew role lacks s3:GetObject or bucket policy denies access."
  resolution: "Add s3:GetObject and s3:ListBucket to the role. Check bucket policy for explicit denies."

- symptoms: "JDBC connection timeout"
  diagnosis: "Security group or NACL blocks outbound traffic to database."
  resolution: "Allow outbound traffic on the database port in the security group."

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
