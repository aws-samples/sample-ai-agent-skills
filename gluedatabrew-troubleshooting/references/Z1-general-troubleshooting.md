---
title: "Z1 — General Troubleshooting"
description: "Catch-all for DataBrew issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "DataBrew error"
  - "something wrong with DataBrew"
owner: devops-agent
objective: "Systematic approach to unclassified DataBrew issues"
context: "Use this runbook when the issue doesn't clearly match a specific category."
---

## Phase 1 — Triage

MUST:
- List projects: `aws databrew list-projects`
- List jobs: `aws databrew list-jobs`
- List datasets: `aws databrew list-datasets`
- Check recent job runs: `aws databrew list-job-runs --name <job-name> --max-results 5`
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=databrew.amazonaws.com --max-results 10`

## Phase 2 — Route to specific runbook

- Project issues → A1, A2
- Recipe job issues → B1, B2
- Profile job issues → C1, C2
- Dataset issues → D1, D2
- Output/S3 issues → E1, E2
- Schedule issues → F1, F2

## Common Issues

- symptoms: "AccessDeniedException on any API call"
  diagnosis: "IAM permissions missing for DataBrew APIs."
  resolution: "Add databrew:* permissions (or specific actions) to the caller's IAM policy."

- symptoms: "ResourceNotFoundException"
  diagnosis: "Project, job, dataset, or schedule name is incorrect or deleted."
  resolution: "Verify resource names with list commands."

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
