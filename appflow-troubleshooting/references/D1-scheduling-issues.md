---
title: "D1 — Scheduling Issues"
description: "Diagnose Amazon AppFlow flow scheduling and timing issues"
status: active
severity: MEDIUM
triggers: ["flow not running on schedule", "schedule missed", "cron expression error"]
owner: devops-agent
objective: "Resolve flow scheduling issues including cron expressions, timezone, and missed executions"
context: "Scheduled flows use EventBridge cron syntax (6 fields) in UTC. Minimum interval varies by connector type."
---

## Phase 1 — Triage

MUST:
- Describe flow trigger configuration: `aws appflow describe-flow --flow-name <flow-name>`
- Check execution history for schedule gaps: `aws appflow describe-flow-execution-records --flow-name <flow-name> --max-results 20`
- Verify flow status is ACTIVE (suspended flows do not execute on schedule)

SHOULD:
- Validate cron expression uses EventBridge 6-field syntax
- Confirm all times are in UTC

MAY:
- Check EventBridge rules if flow uses event-driven trigger: `aws events list-rules --name-prefix appflow`

## Phase 2 — Remediate

MUST:
- Fix cron expression to use EventBridge syntax (6 fields: min hour day month dow year)
- Ensure flow status is ACTIVE
- Convert local times to UTC for schedule configuration

SHOULD:
- Set up CloudWatch alarms for missed executions
- Document the expected schedule in UTC

MAY:
- Use EventBridge rules for complex scheduling patterns

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow details"
  - "describe-flow-execution-records: GREEN — read-only execution history"
  - "events list-rules: GREEN — read-only EventBridge rules"
  - "update-flow (schedule change): YELLOW — modifies execution schedule"
```

## Escalation Conditions

- Scheduled executions are silently missed without error logs
- Schedule changes require coordination with downstream data consumers
- Complex scheduling requirements exceed AppFlow's cron capabilities
- Time-sensitive data pipelines are affected by scheduling failures

## Data Sensitivity

- `aws appflow describe-flow`: MEDIUM — reveals scheduling configuration
- `aws appflow describe-flow-execution-records`: LOW — reveals execution timestamps
- `aws events list-rules`: LOW — reveals EventBridge rule configuration

## Prohibited Actions

- NEVER change production flow schedules without notifying downstream consumers
- NEVER assume local timezone — all AppFlow schedules are UTC
- NEVER use Unix 5-field cron syntax — AppFlow uses EventBridge 6-field syntax
- NEVER set schedules more frequent than the connector's minimum interval

## Phase 3 — Rollback

1. If schedule was changed, revert to previous cron expression using update-flow
2. If flow was suspended during troubleshooting, reactivate it
3. Run a manual execution to fill any data gaps from missed schedules
4. Verify the next scheduled execution fires at the expected time
5. Document the correct schedule and timezone conversion

## Common Issues

- symptoms: "Flow runs at unexpected times"
  diagnosis: "Schedule configured in local time but AppFlow uses UTC."
  resolution: "Convert schedule to UTC. For US Eastern, add 4-5 hours depending on DST."

## Output Format

```yaml
root_cause: "scheduling — <specific_cause>"
evidence:
  - type: schedule_detail
    content: "<schedule config>"
severity: MEDIUM
mitigation:
  immediate: "Fix schedule"
  long_term: "Document UTC schedules"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
