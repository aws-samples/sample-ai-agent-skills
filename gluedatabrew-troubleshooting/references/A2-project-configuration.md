---
title: "A2 — Project Configuration"
description: "Diagnose project settings and session configuration issues"
status: active
severity: MEDIUM
triggers:
  - "project configuration error"
  - "sample size issue"
  - "project session timeout"
owner: devops-agent
objective: "Resolve project configuration and session problems"
context: "Project configuration includes sample size, session timeout, and dataset association. Interactive sessions have idle timeouts. Sample size affects preview data in the session."
---

## Phase 1 — Triage

MUST:
- Describe project: `aws databrew describe-project --name <project-name>`
- Check sample configuration (type and size)
- Verify session role permissions
- Check CloudWatch for session logs

SHOULD:
- Verify dataset format matches project expectations
- Check for concurrent session limits

## Phase 2 — Remediate

MUST:
- Update project sample: `aws databrew update-project --name <name> --sample '{"size":500,"type":"FIRST_N"}'  --role-arn <role-arn>`
- Restart session if timed out

## Common Issues

- symptoms: "Session times out quickly"
  diagnosis: "Idle timeout reached. Default is 30 minutes."
  resolution: "Interact with the session to prevent timeout. Sessions auto-terminate after idle period."

- symptoms: "Sample data doesn't represent full dataset"
  diagnosis: "Sample size too small or using FIRST_N instead of RANDOM."
  resolution: "Increase sample size or use RANDOM sampling type."

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
