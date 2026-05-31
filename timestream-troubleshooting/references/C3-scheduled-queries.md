---
title: "C3 — Scheduled Query Issues"
description: "Diagnose Timestream scheduled query failures"
status: active
severity: HIGH
triggers:
  - "scheduled query failed"
  - "scheduled query not running"
  - "materialized view error"
  - "scheduled query configuration"
owner: devops-agent
objective: "Identify and resolve scheduled query execution failures"
context: "Scheduled queries run SQL on a schedule and write results to a destination table. They require an IAM role, SNS topic for notifications, and S3 bucket for error reports. Without SNS, failures are silent."
---

## Phase 1 — Triage

MUST:
- List scheduled queries: `aws timestream-query list-scheduled-queries`
- Check scheduled query status: `aws timestream-query describe-scheduled-query --scheduled-query-arn <arn>`
- Verify the IAM role has required permissions
- Check SNS topic for failure notifications

SHOULD:
- Check the S3 error bucket for error reports
- Verify the source table exists and has data
- Verify the destination table exists with correct schema
- Check CloudTrail for scheduled query events

MAY:
- Manually run the query to test: `aws timestream-query query --query-string '<scheduled-query-sql>'`
- Check CloudWatch for scheduled query metrics

## Phase 2 — Remediate

MUST:
- Fix the IAM role permissions (read source, write destination, publish SNS, write S3)
- Ensure source and destination tables exist
- Configure SNS notifications for failure alerting

SHOULD:
- Test the query manually before scheduling
- Verify the schedule expression is correct (cron or rate)
- Ensure the destination table schema matches the query output

MAY:
- Set up CloudWatch alarms for scheduled query failures
- Create a monitoring dashboard for scheduled queries

## Common Issues

- symptoms: "Scheduled query shows DISABLED state"
  diagnosis: "Query was disabled due to repeated failures."
  resolution: "Fix the underlying issue, then re-enable the scheduled query."

- symptoms: "No notifications about scheduled query failures"
  diagnosis: "SNS topic not configured or IAM role cannot publish to SNS."
  resolution: "Configure SNS topic and ensure the IAM role has sns:Publish permission."

- symptoms: "Scheduled query runs but destination table empty"
  diagnosis: "Query returns no results, or destination table schema mismatch."
  resolution: "Test the query manually. Verify destination table schema matches query output."

## Output Format

```yaml
root_cause: "scheduled_query_issue — <specific_cause>"
evidence:
  - type: query_status
    content: "<scheduled query state>"
  - type: iam_role
    content: "<IAM role permissions>"
  - type: error_reports
    content: "<S3 error reports>"
severity: HIGH
mitigation:
  immediate: "Fix the scheduled query configuration"
  long_term: "Monitor scheduled queries with SNS and CloudWatch"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Time-series data may contain IoT sensor readings and telemetry
- Query results may reveal operational patterns and metrics
- Scheduled query outputs may aggregate sensitive measurements
- Classification: CONFIDENTIAL - operational telemetry and measurement data

## Prohibited Actions

- NEVER delete Timestream tables containing production time-series data
- NEVER reduce memory store retention below the current data window
- NEVER modify magnetic store retention without data lifecycle review
- NEVER cancel scheduled queries that feed production dashboards

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
