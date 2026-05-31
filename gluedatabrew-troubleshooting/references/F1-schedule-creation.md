---
title: "F1 — Schedule Creation"
description: "Diagnose schedule creation and cron expression issues"
status: active
severity: MEDIUM
triggers:
  - "schedule creation failed"
  - "cron expression error"
  - "invalid schedule"
owner: devops-agent
objective: "Resolve schedule creation and cron expression problems"
context: "DataBrew schedules use AWS cron expressions (6 fields). Schedules trigger job runs automatically. A schedule can be attached to one or more jobs."
---

## Phase 1 — Triage

MUST:
- List schedules: `aws databrew list-schedules`
- Describe schedule: `aws databrew describe-schedule --name <schedule-name>`
- Verify cron expression syntax (6 fields: min hour dom month dow year)
- Check schedule-to-job associations

SHOULD:
- Verify day-of-week and day-of-month aren't both specified (use ? for one)
- Check timezone considerations (schedules run in UTC)

## Phase 2 — Remediate

MUST:
- Create schedule: `aws databrew create-schedule --name <name> --cron-expression "cron(0 8 ? * MON-FRI *)" --job-names <job-name>`
- Verify schedule: `aws databrew describe-schedule --name <name>`

## Common Issues

- symptoms: "Invalid cron expression"
  diagnosis: "Using 5-field Linux cron instead of 6-field AWS cron."
  resolution: "Use 6 fields: cron(min hour dom month dow year). Add year field."

- symptoms: "Schedule runs at wrong time"
  diagnosis: "Cron expressions use UTC, not local timezone."
  resolution: "Convert desired local time to UTC for the cron expression."

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
