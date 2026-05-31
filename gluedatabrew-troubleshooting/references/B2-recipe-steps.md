---
title: "B2 — Recipe Steps"
description: "Diagnose recipe step configuration and ordering issues"
status: active
severity: MEDIUM
triggers:
  - "recipe step error"
  - "transformation not applied"
  - "wrong recipe output"
owner: devops-agent
objective: "Resolve recipe step configuration problems"
context: "Recipe steps execute sequentially. Each step has a type (FILTER, RENAME, FORMAT, etc.) and parameters. Step order affects results. Recipes can be published as versions."
---

## Phase 1 — Triage

MUST:
- Get recipe: `aws databrew describe-recipe --name <recipe-name>`
- List recipe versions: `aws databrew list-recipe-versions --name <recipe-name>`
- Check step order and parameters
- Verify the job uses the correct recipe version

SHOULD:
- Test recipe in interactive session before running as job
- Check for conflicting steps (e.g., rename then reference old name)

## Phase 2 — Remediate

MUST:
- Fix step configuration or ordering
- Publish new recipe version: `aws databrew publish-recipe --name <recipe-name>`
- Update job to use latest recipe version if needed

## Common Issues

- symptoms: "Column not found in transformation step"
  diagnosis: "Previous step renamed or removed the column."
  resolution: "Reorder steps or update column references to match upstream changes."

- symptoms: "Job uses old recipe version"
  diagnosis: "Job references a specific recipe version, not the latest."
  resolution: "Update job to reference the latest published recipe version."

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
