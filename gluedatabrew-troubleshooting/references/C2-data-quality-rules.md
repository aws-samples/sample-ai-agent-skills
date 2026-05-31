---
title: "C2 — Data Quality Rules"
description: "Diagnose data quality ruleset evaluation failures"
status: active
severity: MEDIUM
triggers:
  - "data quality rule failed"
  - "ruleset error"
  - "quality check not passing"
owner: devops-agent
objective: "Resolve data quality ruleset configuration and evaluation issues"
context: "Data quality rulesets define validation rules (null checks, range checks, uniqueness, patterns). Rulesets are attached to profile jobs. Rules evaluate against profiled data and report pass/fail results."
---

## Phase 1 — Triage

MUST:
- List rulesets: `aws databrew list-rulesets`
- Describe ruleset: `aws databrew describe-ruleset --name <ruleset-name>`
- Check profile job configuration for ruleset attachment
- Review profile job results for rule evaluation outcomes

SHOULD:
- Verify rule expressions reference valid column names
- Check threshold values are appropriate for the data

## Phase 2 — Remediate

MUST:
- Fix rule expressions or thresholds
- Update ruleset: `aws databrew update-ruleset --name <name> --rules <rules-json>`
- Re-run profile job to re-evaluate rules

## Common Issues

- symptoms: "Rule references column that doesn't exist"
  diagnosis: "Column name in rule doesn't match dataset column names."
  resolution: "Update rule to reference correct column name (case-sensitive)."

- symptoms: "All rules fail unexpectedly"
  diagnosis: "Ruleset attached to wrong dataset or data format changed."
  resolution: "Verify ruleset is attached to the correct profile job and dataset."

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
