---
title: "G1 — Schedule Configuration Issues"
description: "Diagnose DataSync task scheduling problems"
status: active
severity: MEDIUM
triggers:
  - "schedule not working"
  - "task not running on schedule"
  - "cron expression error"
  - "scheduled task missed"
owner: devops-agent
objective: "Identify and resolve DataSync task scheduling configuration issues"
context: "DataSync tasks can be scheduled using cron expressions or rate expressions. Schedules are configured on the task and trigger automatic task executions. Common issues include incorrect cron syntax, timezone confusion (schedules use UTC), and overlapping executions."
---

## Phase 1 — Triage

MUST:
- Check task schedule configuration: `aws datasync describe-task --task-arn <task-arn> --query 'Schedule'`
- Verify the cron/rate expression is valid
- Check task execution history: `aws datasync list-task-executions --task-arn <task-arn> --query 'TaskExecutions[*].{Arn:TaskExecutionArn,Status:Status}'`
- Confirm schedule uses UTC timezone

SHOULD:
- Check if previous execution is still running (blocks scheduled execution)
- Verify task status is AVAILABLE (not RUNNING or UNAVAILABLE)
- Review CloudTrail for StartTaskExecution events

MAY:
- Check EventBridge for schedule-related events
- Verify no SCP or IAM policy blocks scheduled execution

## Phase 2 — Remediate

MUST:
- Use valid cron expression format: `cron(minutes hours day-of-month month day-of-week year)`
- Remember schedules are in UTC: adjust for your timezone
- Update schedule: `aws datasync update-task --task-arn <task-arn> --schedule ScheduleExpression="cron(0 2 * * ? *)"`

SHOULD:
- Allow sufficient time between scheduled runs to avoid overlap
- Use rate expressions for simple intervals: `rate(1 hour)`, `rate(6 hours)`

MAY:
- Use EventBridge rules for more complex scheduling logic
- Implement a Lambda function to manage task execution with custom logic

## Common Issues

- symptoms: "Scheduled task not running at expected time"
  diagnosis: "Schedule is in UTC, not local timezone."
  resolution: "Convert desired local time to UTC for the cron expression."

- symptoms: "Scheduled execution skipped"
  diagnosis: "Previous execution still running when schedule triggers."
  resolution: "Ensure tasks complete before next scheduled run. Increase interval if needed."

- symptoms: "Invalid schedule expression error"
  diagnosis: "Cron expression syntax is incorrect."
  resolution: "Use AWS cron format: cron(0 12 * * ? *) for daily at noon UTC."

## Output Format

```yaml
root_cause: "schedule_config — <specific_cause>"
evidence:
  - type: schedule_expression
    content: "<configured schedule>"
  - type: execution_history
    content: "<recent task executions and times>"
  - type: task_status
    content: "<current task status>"
severity: MEDIUM
mitigation:
  immediate: "Fix schedule expression and verify UTC timezone"
  long_term: "Monitor scheduled executions and set up alerts for missed runs"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Schedule configuration is operational metadata |
| Blast Radius | MEDIUM — Incorrect schedules can cause missed transfers or overlapping executions |
| Rollback Complexity | LOW — Schedule can be updated immediately |
| Compliance Scope | MEDIUM — Scheduled transfers may be required for compliance data delivery |

## Escalation Conditions

- Escalate to operations if missed scheduled transfers affect data delivery SLAs
- Escalate to compliance if scheduled transfers are required for regulatory data delivery

## Data Sensitivity

- **Schedule expressions** are operational metadata
- **Execution history** reveals transfer patterns

## PROHIBITED Actions

- Do NOT schedule transfers during peak business hours without coordination
- Do NOT set schedules that overlap with previous execution completion times
- Do NOT remove schedules for compliance-required transfers without approval

## Phase 3 — Rollback

- Restore previous schedule expression: `aws datasync update-task --task-arn <arn> --schedule ScheduleExpression="<previous-expression>"`
- Manually trigger a task execution if a scheduled run was missed
- Verify the restored schedule fires at the expected time

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
