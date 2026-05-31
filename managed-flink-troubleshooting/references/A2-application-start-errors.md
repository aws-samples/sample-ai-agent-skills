---
title: "A2 — Application Start Errors"
description: "Diagnose Managed Flink application start and initialization failures"
status: active
severity: HIGH
triggers: ["application won't start", "start failed", "application stuck starting", "initialization error"]
owner: devops-agent
objective: "Resolve application start issues including code errors, resource allocation, and snapshot restoration"
context: "Application start involves code download, resource allocation, Flink job submission, and optional snapshot restoration."
---

## Phase 1 — Triage

MUST:
- Check application status: `aws kinesisanalyticsv2 describe-application --application-name <app-name> --query 'ApplicationDetail.ApplicationStatus'`
- Review CloudWatch Logs: `aws logs filter-log-events --log-group-name /aws/kinesis-analytics/<app-name> --start-time <epoch-ms> --limit 50`
- Check for start errors in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=StartApplication --max-results 10`

SHOULD:
- Verify application code is valid (no compilation errors)
- Check if starting from a snapshot and snapshot is compatible

MAY:
- Check CloudWatch metrics for resource allocation: `aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name uptime --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 60 --statistics Maximum`

## Phase 2 — Remediate

MUST:
- Fix code compilation or initialization errors
- Ensure IAM role has permissions for source/sink access
- If snapshot restoration fails, try starting without snapshot

SHOULD:
- Enable CloudWatch logging if not configured
- Verify source streams/topics exist and are accessible

MAY:
- Start with reduced parallelism to isolate resource issues
- Test code locally with Flink mini-cluster before deploying

## Safety Ratings

```
safety_ratings:
  - "describe-application: GREEN — read-only application details"
  - "logs filter-log-events: GREEN — read-only log review"
  - "StartApplication: YELLOW — starts application and allocates KPUs"
  - "StopApplication: YELLOW — stops application processing"
```

## Escalation Conditions

- Application transitions to RUNNING then immediately fails
- Start fails with internal service errors not related to user code
- Snapshot restoration fails and snapshot cannot be recreated
- Application requires more KPUs than the service quota allows

## Data Sensitivity

- `aws kinesisanalyticsv2 describe-application`: MEDIUM — reveals application configuration
- `aws logs filter-log-events`: HIGH — may contain application data and error details
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER start production applications without verifying source/sink connectivity
- NEVER discard snapshots before confirming the application starts successfully
- NEVER start applications with maximum parallelism for initial testing
- NEVER ignore start failures — they indicate configuration or code issues

## Phase 3 — Rollback

1. If application was started with wrong configuration, stop it: `aws kinesisanalyticsv2 stop-application --application-name <name>`
2. If configuration was updated before start, revert using update-application
3. Restore from a known-good snapshot if available
4. Verify source streams are not affected by the failed start
5. Document the start failure and root cause

## Common Issues

- symptoms: "Application status transitions from STARTING to READY repeatedly"
  diagnosis: "Flink job fails during initialization. Code error or missing dependency."
  resolution: "Check CloudWatch Logs for Java exceptions. Verify all dependencies are included in the JAR. Check main class configuration."

## Output Format

```yaml
root_cause: "application_start — <specific_cause>"
evidence:
  - type: start_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix start issue"
  long_term: "Add pre-start validation"
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
