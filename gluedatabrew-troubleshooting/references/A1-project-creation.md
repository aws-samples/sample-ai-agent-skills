---
title: "A1 — Project Creation"
description: "Diagnose DataBrew project creation and session start failures"
status: active
severity: HIGH
triggers:
  - "project creation failed"
  - "cannot create project"
  - "session start error"
owner: devops-agent
objective: "Resolve DataBrew project creation issues"
context: "A DataBrew project links a dataset to an interactive session for recipe development. Projects require an IAM role and a valid dataset. Session start failures are often IAM or dataset issues."
---

## Phase 1 — Triage

MUST:
- List projects: `aws databrew list-projects`
- Describe project: `aws databrew describe-project --name <project-name>`
- Check dataset: `aws databrew describe-dataset --name <dataset-name>`
- Verify IAM role: `aws iam get-role --role-name <role-name>`
- Check role trust policy includes `databrew.amazonaws.com`

SHOULD:
- Verify S3 access from the role
- Check CloudTrail for creation errors

## Phase 2 — Remediate

MUST:
- Create project with valid dataset and role: `aws databrew create-project --name <name> --dataset-name <dataset> --recipe-name <recipe> --role-arn <role-arn>`
- Ensure IAM role has S3 read access to dataset location

## Common Issues

- symptoms: "Session fails to start"
  diagnosis: "IAM role cannot access the dataset's S3 location."
  resolution: "Add s3:GetObject and s3:ListBucket permissions to the role for the dataset's S3 path."

- symptoms: "Dataset not found"
  diagnosis: "Dataset name doesn't exist or was deleted."
  resolution: "Create the dataset first, then create the project referencing it."

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
