---
title: "A3 — Missed Schedule Invocations"
description: "Diagnose why scheduled invocations are not executing"
status: active
severity: HIGH
triggers:
  - "missed invocation"
  - "schedule not firing"
  - "schedule not running"
  - "no invocation"
  - "schedule skipped"
owner: devops-agent
objective: "Identify why scheduled invocations are being missed and restore reliable execution"
context: "Missed invocations can be caused by disabled schedules, IAM role issues, target failures, flexible time window confusion, or one-time schedule auto-deletion. CloudWatch metrics and CloudTrail provide diagnostic data."
---

## Phase 1 — Triage

MUST:
- Check schedule state: `aws scheduler get-schedule --name <schedule-name>`
- Verify State is ENABLED (not DISABLED)
- Check IAM execution role permissions: `aws iam get-role --role-name <role-name>`
- Check CloudWatch metrics for invocations: `aws cloudwatch get-metric-statistics --namespace AWS/Scheduler --metric-name InvocationCount --dimensions Name=ScheduleName,Value=<schedule-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check for failed invocations: `aws cloudwatch get-metric-statistics --namespace AWS/Scheduler --metric-name TargetErrorCount --dimensions Name=ScheduleName,Value=<schedule-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify the target is healthy and accessible
- Check if the schedule was a one-time (at) expression that already executed and was deleted
- Review flexible time window configuration

MAY:
- Check DLQ for failed invocation events
- Review CloudTrail for schedule modifications
- Verify no SCP or IAM boundary is blocking the execution role

## Phase 2 — Remediate

MUST:
- Enable the schedule if disabled
- Fix IAM execution role permissions for the target
- Verify target is healthy and accessible

SHOULD:
- Configure DLQ to capture failed invocations
- Set up CloudWatch alarms for missed invocations
- Configure retry policy for transient failures

MAY:
- Implement schedule health monitoring
- Use schedule groups for organized management

## Common Issues

- symptoms: "Schedule exists but no invocations recorded"
  diagnosis: "Schedule State is DISABLED or the start date is in the future."
  resolution: "Enable the schedule. Verify StartDate is in the past or not set."

- symptoms: "One-time schedule disappeared"
  diagnosis: "One-time (at) schedules are auto-deleted after execution by default."
  resolution: "Set ActionAfterCompletion=NONE to keep the schedule. Check CloudTrail for deletion."

- symptoms: "Invocations happen but target reports no events"
  diagnosis: "IAM role lacks permission to invoke the target — invocation fails silently without DLQ."
  resolution: "Add target permissions to the role. Configure DLQ to capture failures."

## Output Format

```yaml
root_cause: "missed_invocation — <specific_cause>"
evidence:
  - type: schedule
    content: "<schedule state and configuration>"
  - type: metrics
    content: "<invocation count and error metrics>"
  - type: iam_role
    content: "<execution role permissions>"
severity: HIGH
mitigation:
  immediate: "Enable schedule and fix permissions"
  long_term: "Configure DLQ, retry policy, and monitoring"
```


## Safety Ratings
```
safety_ratings:
  - "Check schedule state: GREEN — read-only API call"
  - "Check CloudWatch invocation metrics: GREEN — read-only monitoring query"
  - "Verify IAM execution role: GREEN — read-only IAM inspection"
  - "Enable disabled schedule: YELLOW — resumes invocations, may trigger immediate execution"
  - "Fix IAM execution role permissions: YELLOW — changes access scope"
  - "Configure DLQ and retry policy: GREEN — adds error handling without changing invocation behavior"
```

## Escalation Conditions
- Schedule drives production workflow automation
- Missed invocations causing data processing gaps or SLA violations
- One-time schedule auto-deleted before execution was confirmed
- Target service unhealthy causing cascading failures
- IAM role changes needed across multiple schedules

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Invocation metrics: business process execution patterns"
    - "DLQ messages: failed invocation payloads with business data"
    - "Target error details: downstream service state information"
  handling: "DLQ messages may contain sensitive business data — handle with appropriate access controls."
```

## Prohibited Actions
- NEVER delete a one-time schedule before confirming it has executed successfully
- NEVER assume a schedule is working without checking CloudWatch invocation metrics
- NEVER modify the execution role without understanding impact on all schedules using it
- NEVER purge DLQ messages before analyzing failure patterns

## Phase 3 — Rollback
- If schedule was enabled: disable the schedule if invocations are causing issues
- If IAM role permissions were changed: revert to previous policy version
- If retry policy was modified: restore previous retry configuration
- If DLQ was configured: remove DLQ configuration if not needed (failed events will be lost)
- If ActionAfterCompletion was changed: update back to previous setting

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
