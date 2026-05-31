---
title: "C2 — Backup Schedule Issues"
description: "Diagnose why backup plans are not running on the expected schedule"
status: active
severity: MEDIUM
triggers:
  - "backup not running"
  - "schedule not triggering"
  - "missed backup"
  - "cron expression"
  - "backup frequency"
owner: devops-agent
objective: "Identify why backup plan schedules are not triggering as expected"
context: "Backup plan rules use cron expressions (in UTC) to define schedules. The start window defines when a job can begin, and the backup window defines the acceptable start period. Misunderstanding UTC conversion, cron syntax, or start windows causes most schedule issues."
---

## Phase 1 — Triage

MUST:
- Get the backup plan schedule: `aws backup get-backup-plan --backup-plan-id <plan-id>`
- Check the ScheduleExpression (cron format) and StartWindowMinutes for each rule
- List recent backup jobs to see actual execution times: `aws backup list-backup-jobs --by-backup-plan-id <plan-id> --max-results 20`
- Verify the cron expression is in UTC (all AWS Backup schedules are UTC)
- Check if the plan is active and has valid selections

SHOULD:
- Verify the cron expression syntax is correct for AWS (6-field cron: minute hour day month day-of-week year)
- Check if the start window is too narrow (job must start within this window)
- Verify the backup selection has matching resources
- Check CloudTrail for any plan modification events

MAY:
- Compare expected vs actual backup times over the past week
- Check if AWS Backup service had any outages during missed windows
- Verify no SCP is blocking backup operations

## Phase 2 — Remediate

MUST:
- Fix the cron expression to match the desired schedule (remember UTC)
- Ensure the start window is wide enough for the job to begin
- Verify the backup selection targets the correct resources

SHOULD:
- Test the schedule by running an on-demand backup to verify configuration
- Set up SNS notifications for backup job completion/failure
- Document the schedule in local time with UTC conversion

MAY:
- Use AWS Backup's built-in frequency options (every 12 hours, daily, weekly, monthly) instead of custom cron
- Create CloudWatch alarms for missed backup windows

## Common Issues

- symptoms: "Backup runs at unexpected time"
  diagnosis: "The cron expression is in UTC but the user expected local time."
  resolution: "Convert the desired local time to UTC for the cron expression."

- symptoms: "Backup plan exists but no jobs are created"
  diagnosis: "The backup selection has no matching resources (tags don't match or ARNs are invalid)."
  resolution: "Verify the selection matches actual resources. Check tag values are exact matches."

- symptoms: "Backup job created but immediately cancelled"
  diagnosis: "The start window expired before the job could begin (too many concurrent jobs)."
  resolution: "Increase StartWindowMinutes or stagger schedules to reduce concurrency."

- symptoms: "Weekly backup runs every day"
  diagnosis: "Cron expression uses * for day-of-month instead of ? when specifying day-of-week."
  resolution: "Use ? for day-of-month when specifying day-of-week: cron(0 5 ? * 1 *) for weekly Sunday."

## Output Format

```yaml
root_cause: "schedule_issue — <specific_cause>"
evidence:
  - type: schedule
    content: "<cron expression and start window>"
  - type: recent_jobs
    content: "<actual backup job execution times>"
  - type: selection
    content: "<resource selection and matching resources>"
severity: MEDIUM
mitigation:
  immediate: "Fix the schedule expression and verify with on-demand backup"
  long_term: "Set up monitoring for missed backup windows"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix cron expression | YELLOW | Changes when backups run — verify UTC conversion |
| Ensure start window is wide enough | YELLOW | Changes backup timing flexibility |
| Verify backup selection targets correct resources | GREEN | Verification only |
| Test schedule with on-demand backup | GREEN | Testing — non-destructive |
| Set up SNS notifications | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- Schedule changes may create gaps in backup coverage
- Start window changes affect when backup jobs can begin

## Data Sensitivity

- Backup schedules reveal operational patterns and maintenance windows
- Schedule configuration reveals data protection frequency requirements
- Resource selection reveals which resources are under backup protection

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER remove backup schedules without ensuring alternative coverage
- NEVER set start windows so narrow that jobs cannot begin

## Phase 3 — Rollback

- If cron expression was changed incorrectly: restore original schedule expression
- If start window was modified: restore original start window
- If backup selection was changed: restore original resource targeting
- Document all schedule changes for compliance audit trail

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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
