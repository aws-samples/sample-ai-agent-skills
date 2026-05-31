---
title: "A2 — Log Group Retention"
description: "Diagnose and fix log group retention policy issues"
status: active
severity: MEDIUM
triggers:
  - "log retention"
  - "logs never expire"
  - "log storage cost"
  - "set retention policy"
  - "logs accumulating"
owner: devops-agent
objective: "Identify log groups with missing or incorrect retention policies and fix them to control costs"
context: "CloudWatch Logs retention defaults to Never Expire. Logs stored indefinitely accumulate costs. Retention can be set from 1 day to 10 years. Changing retention does not delete already-expired logs immediately — deletion happens asynchronously within 72 hours."
---

## Phase 1 — Triage

MUST:
- List log groups and their retention: `aws logs describe-log-groups --query 'logGroups[].{name:logGroupName,retention:retentionInDays,storedBytes:storedBytes}'`
- Identify log groups with no retention (retentionInDays is null = Never Expire)
- Check stored bytes to estimate cost impact

SHOULD:
- Sort log groups by storedBytes to find the largest cost contributors
- Check if any compliance requirements mandate specific retention periods
- Review CloudTrail for recent put-retention-policy calls

MAY:
- Calculate monthly storage cost: storedBytes * $0.03 per GB (varies by region)
- Check if log groups are actively receiving new events

## Phase 2 — Remediate

MUST:
- Set retention policy on log groups without one: `aws logs put-retention-policy --log-group-name <name> --retention-in-days <days>`
- Valid retention values: 1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1096, 1827, 2192, 2557, 2922, 3288, 3653

SHOULD:
- Apply retention policies via infrastructure-as-code (CloudFormation, Terraform)
- Set account-level default retention for new log groups using account policy

MAY:
- Export old logs to S3 before setting retention (for compliance)
- Use `aws logs create-export-task` to archive logs to S3

## Common Issues

- symptoms: "CloudWatch Logs bill unexpectedly high"
  diagnosis: "Multiple log groups with Never Expire retention accumulating data."
  resolution: "Audit all log groups and set appropriate retention policies."

- symptoms: "Retention set but old logs still visible"
  diagnosis: "Log deletion after retention change is asynchronous."
  resolution: "Wait up to 72 hours for expired log events to be deleted."

- symptoms: "Cannot set retention on log group"
  diagnosis: "IAM policy missing logs:PutRetentionPolicy permission."
  resolution: "Add logs:PutRetentionPolicy to the IAM policy."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Set retention policy | YELLOW | Triggers asynchronous deletion of expired log events |
| Export logs to S3 before retention | GREEN | Read-only export |
| Set account-level default retention | YELLOW | Affects all new log groups |

## Escalation Conditions

- Retention changes on compliance-critical log groups — escalate to compliance team
- Large-scale retention policy changes across many log groups — escalate to operations team
- Log groups with no retention accumulating significant costs — escalate to cost management team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws logs describe-log-groups` | MEDIUM | Exposes log group names, retention, and storage size |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** set retention shorter than compliance requirements

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Set retention policy | Remove retention (revert to Never Expire): `aws logs delete-retention-policy --log-group-name <name>` |
| Set account-level default | Remove account policy |
| Note: expired data already deleted cannot be recovered | N/A |

## Output Format

```yaml
root_cause: "log_retention — <specific_cause>"
evidence:
  - type: log_group_retention
    content: "<log groups and their retention settings>"
  - type: storage_cost
    content: "<estimated monthly cost>"
severity: MEDIUM
mitigation:
  immediate: "Set retention policies on identified log groups"
  long_term: "Enforce retention via account policy or IaC"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
