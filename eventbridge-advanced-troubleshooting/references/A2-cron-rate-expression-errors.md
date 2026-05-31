---
title: "A2 — Cron/Rate Expression Errors"
description: "Diagnose cron and rate expression syntax and behavior issues"
status: active
severity: MEDIUM
triggers:
  - "cron expression"
  - "rate expression"
  - "schedule expression"
  - "wrong time"
  - "expression syntax"
owner: devops-agent
objective: "Identify and resolve cron/rate expression errors in EventBridge Scheduler"
context: "Scheduler supports cron expressions (6 fields), rate expressions, and at expressions (one-time). Cron syntax differs slightly from standard Unix cron. Common issues include timezone confusion, day-of-week numbering, and expression validation."
---

## Phase 1 — Triage

MUST:
- Get the schedule expression: `aws scheduler get-schedule --name <schedule-name>`
- Verify the expression syntax matches Scheduler format
- Check the ScheduleExpressionTimezone setting
- Verify the schedule State is ENABLED

SHOULD:
- Compare expected vs actual execution times
- Check if the timezone is correctly specified (default is UTC)
- Verify cron field values are within valid ranges
- Check for day-of-month vs day-of-week conflicts (use ? for one)

MAY:
- Test the expression with a short-lived schedule
- Review Scheduler cron documentation for syntax differences
- Check for DST (Daylight Saving Time) effects on the timezone

## Phase 2 — Remediate

MUST:
- Fix the expression syntax to match Scheduler cron format
- Set the correct timezone (ScheduleExpressionTimezone)
- Use ? for either day-of-month or day-of-week (cannot specify both)

SHOULD:
- Document the intended schedule in human-readable format alongside the expression
- Test expressions before production deployment
- Consider timezone effects for global schedules

MAY:
- Use rate expressions for simple intervals (simpler than cron)
- Implement schedule monitoring to verify execution timing

## Common Issues

- symptoms: "Schedule fires at wrong time"
  diagnosis: "Timezone is not set or defaults to UTC instead of local timezone."
  resolution: "Set ScheduleExpressionTimezone to the desired timezone (e.g., America/New_York)."

- symptoms: "Cron expression validation error"
  diagnosis: "Scheduler cron uses 6 fields (minute hour day-of-month month day-of-week year). Day-of-week is 1-7 (SUN-SAT)."
  resolution: "Use 6-field format: cron(0 12 * * ? *). Use ? for day-of-month or day-of-week."

- symptoms: "Rate expression not triggering as expected"
  diagnosis: "Rate expression syntax is rate(value unit) where unit is minutes, hours, or days."
  resolution: "Use correct format: rate(5 minutes), rate(1 hour), rate(1 day). Singular for value=1."

## Output Format

```yaml
root_cause: "expression_error — <specific_cause>"
evidence:
  - type: schedule
    content: "<schedule expression and timezone>"
  - type: execution
    content: "<expected vs actual execution times>"
  - type: syntax
    content: "<expression syntax analysis>"
severity: MEDIUM
mitigation:
  immediate: "Fix expression syntax and timezone"
  long_term: "Document and test expressions before deployment"
```


## Safety Ratings
```
safety_ratings:
  - "Get schedule expression: GREEN — read-only API call"
  - "Verify expression syntax: GREEN — read-only analysis"
  - "Fix expression syntax: GREEN — corrects schedule timing"
  - "Set timezone: GREEN — adjusts schedule timing reference"
  - "Update schedule with new expression: YELLOW — changes when the schedule fires"
```

## Escalation Conditions
- Schedule drives production workflow automation
- Wrong expression causing missed or extra invocations in production
- Timezone change affecting business-critical scheduling
- DST transition causing unexpected schedule behavior

## Data Sensitivity
```
data_sensitivity:
  classification: LOW
  sensitive_fields:
    - "Schedule expressions: timing patterns for business processes"
    - "Timezone settings: operational region indicators"
  handling: "Schedule expressions are generally low sensitivity but may reveal business operation patterns."
```

## Prohibited Actions
- NEVER change a production schedule expression without testing in a non-production environment
- NEVER assume UTC when the business expects local timezone
- NEVER use rate expressions for time-of-day-sensitive operations (use cron instead)
- NEVER modify multiple production schedules simultaneously without a rollback plan

## Phase 3 — Rollback
- If cron expression was changed: update the schedule with the previous cron expression
- If timezone was changed: revert ScheduleExpressionTimezone to the previous value
- If rate expression was changed to cron: revert to the original rate expression
- If schedule was disabled during testing: re-enable the schedule immediately after verification

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
