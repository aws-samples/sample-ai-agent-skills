---
title: "C2 — Scheduling Issues"
description: "Diagnose SSM Distributor association scheduling problems"
status: active
severity: MEDIUM
triggers: ["schedule not working", "association not running on time", "cron expression error", "rate expression"]
owner: devops-agent
objective: "Resolve association scheduling and timing issues"
context: "Associations use cron or rate expressions for scheduling. ApplyOnlyAtCronInterval controls whether the association runs immediately on creation."
---

## Phase 1 — Triage

MUST:
- Check association schedule: `aws ssm describe-association --association-id <assoc-id> --query '{ScheduleExpression:ScheduleExpression,ApplyOnlyAtCronInterval:ApplyOnlyAtCronInterval,LastExecutionDate:Overview.LastExecutionDate}'`
- Verify last execution time matches expected schedule
- Check if ApplyOnlyAtCronInterval is set correctly

SHOULD:
- List recent executions to verify schedule pattern: `aws ssm describe-association-executions --association-id <assoc-id> --filters Key=CreatedTime,Value=<date>,Type=GREATER_THAN --query 'AssociationExecutions[*].{CreatedTime:CreatedTime,Status:Status}'`
- Verify the cron/rate expression syntax is valid

MAY:
- Compare UTC time with expected execution time (SSM uses UTC)
- Check for overlapping associations on the same targets

## Phase 2 — Remediate

MUST:
- Fix cron/rate expression syntax errors
- Set ApplyOnlyAtCronInterval appropriately (false for run-on-create behavior)
- Update schedule expression: `aws ssm update-association --association-id <assoc-id> --schedule-expression "cron(0 2 ? * SUN *)"`

SHOULD:
- Test schedule by triggering manual execution: `aws ssm start-associations-once --association-ids <assoc-id>`
- Stagger schedules across large fleets using MaxConcurrency

MAY:
- Use rate expressions for simple intervals: `rate(1 day)`
- Implement maintenance windows for coordinated deployments

## Common Issues

- symptoms: "Association does not run at the expected time"
  diagnosis: "Schedule expression uses local time but SSM uses UTC."
  resolution: "Convert schedule to UTC. SSM cron expressions are always evaluated in UTC."

- symptoms: "Association runs immediately on creation but should only run on schedule"
  diagnosis: "ApplyOnlyAtCronInterval is false (default)."
  resolution: "Set ApplyOnlyAtCronInterval to true to prevent immediate execution on creation."

## Safety Ratings

```
safety_ratings:
  - "describe-association: GREEN — read-only association details"
  - "describe-association-executions: GREEN — read-only execution history"
  - "update-association schedule: YELLOW — changes when packages are deployed"
  - "start-associations-once: YELLOW — triggers immediate execution"
```

## Escalation Conditions

- Schedule changes cause packages to deploy during production hours
- Overlapping schedules cause resource contention on target instances
- Schedule expression accepted but never triggers execution
- Fleet-wide schedule change needed with zero-downtime requirement

## Data Sensitivity

- `aws ssm describe-association`: LOW — returns association configuration
- `aws ssm describe-association-executions`: LOW — returns execution timestamps and status
- Schedule expressions: LOW — contain timing information only

## Prohibited Actions

- NEVER change association schedules in production without communicating the maintenance window
- NEVER set schedules that overlap with other critical operations on the same instances
- NEVER use rate expressions shorter than 30 minutes for package installations
- NEVER assume local time — SSM always uses UTC for schedule evaluation

## Phase 3 — Rollback

1. Revert schedule expression: `aws ssm update-association --association-id <assoc-id> --schedule-expression "<original-expression>"`
2. If immediate execution was triggered accidentally, verify package state on affected instances
3. If ApplyOnlyAtCronInterval was changed, revert it to the original value
4. Verify next scheduled execution time is correct after rollback
5. Document the schedule change and notify affected teams

## Output Format

```yaml
root_cause: "scheduling_issue — <specific_cause>"
evidence:
  - type: schedule_config
    content: "<schedule details>"
severity: MEDIUM
mitigation:
  immediate: "Fix schedule expression"
  long_term: "Document and standardize scheduling practices"
```

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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
