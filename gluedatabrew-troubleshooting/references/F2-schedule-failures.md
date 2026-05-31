---
title: "F2 — Schedule Failures"
description: "Diagnose scheduled job run failures and missed triggers"
status: active
severity: MEDIUM
triggers:
  - "scheduled job didn't run"
  - "schedule not triggering"
  - "missed scheduled run"
owner: devops-agent
objective: "Resolve scheduled job execution failures"
context: "Scheduled jobs may fail to trigger or fail during execution. Check schedule status, job configuration, and IAM permissions. Disabled schedules don't trigger."
---

## Phase 1 — Triage

MUST:
- Check schedule status: `aws databrew describe-schedule --name <schedule-name>`
- List recent job runs: `aws databrew list-job-runs --name <job-name> --max-results 10`
- Check if schedule is enabled (not disabled)
- Verify job still exists and is properly configured

SHOULD:
- Check CloudTrail for schedule-related events
- Verify no concurrent job run limit was hit
- Check for service quota limits on concurrent jobs

## Phase 2 — Remediate

MUST:
- Enable schedule if disabled
- Fix job configuration if job runs fail after trigger
- Check and increase concurrent job quota if needed

## Common Issues

- symptoms: "Schedule exists but job never runs"
  diagnosis: "Schedule is in disabled state."
  resolution: "Enable the schedule or recreate it in enabled state."

- symptoms: "Scheduled run starts but fails immediately"
  diagnosis: "Job configuration changed since schedule was created (e.g., dataset deleted)."
  resolution: "Verify job configuration — dataset, role, and output are all valid."

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
