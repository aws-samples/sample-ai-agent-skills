---
title: "E2 — Monitoring Integration"
description: "Diagnose monitoring and observability gaps in M2 environments"
status: active
severity: MEDIUM
triggers: ["no metrics from M2", "logging not working", "monitoring gap", "alerting not configured", "CloudWatch not showing M2 data"]
owner: devops-agent
objective: "Configure and troubleshoot monitoring for Mainframe Modernization environments"
context: "M2 publishes metrics to CloudWatch and logs to CloudWatch Logs. Application-level logging depends on the engine type. Batch job monitoring requires specific metric configuration."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch metrics availability: `aws cloudwatch list-metrics --namespace AWS/M2 --dimensions Name=EnvironmentId,Value=<env-id>`
- Verify log groups exist: `aws logs describe-log-groups --log-group-name-prefix /aws/m2/`
- Check for recent log entries: `aws logs filter-log-events --log-group-name /aws/m2/<environment-name> --start-time <epoch-ms> --limit 10`

SHOULD:
- Verify CloudWatch alarms are configured for critical metrics
- Check if application-level logging is enabled in the application definition
- Review log retention settings

MAY:
- Set up CloudWatch dashboards for M2 environments
- Configure SNS notifications for alarm state changes

## Phase 2 — Remediate

MUST:
- Ensure M2 environment is publishing metrics (environment must be Available)
- Verify log group permissions allow M2 service to write logs
- Configure basic alarms for CPU, memory, and application errors

SHOULD:
- Set appropriate log retention periods
- Configure batch job execution monitoring

MAY:
- Implement custom metrics for application-specific KPIs
- Create CloudWatch Insights queries for common troubleshooting patterns

## Common Issues

- symptoms: "No CloudWatch metrics for M2 environment"
  diagnosis: "Environment is not in Available status or metrics namespace is incorrect."
  resolution: "Verify environment status: aws m2 get-environment. Use namespace AWS/M2 with dimension EnvironmentId."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch list-metrics: GREEN — read-only metric enumeration"
  - "logs describe-log-groups: GREEN — read-only log group check"
  - "cloudwatch put-metric-alarm: YELLOW — creates alarms that may trigger notifications"
  - "logs delete-log-group: RED — permanently destroys log data"
```

## Escalation Conditions

- Complete loss of monitoring visibility for production M2 environments
- Log delivery failures causing audit compliance gaps
- Alarm fatigue from misconfigured thresholds
- Monitoring gaps during critical batch processing windows

## Data Sensitivity

- CloudWatch metrics: LOW — aggregate performance data
- Application logs: MEDIUM — may contain business data references
- Log group configuration: LOW — reveals logging setup
- Alarm configurations: LOW — reveals monitoring thresholds

## Prohibited Actions

- NEVER delete log groups without archiving data first
- NEVER disable logging on production M2 environments
- NEVER set log retention below compliance requirements
- NEVER configure alarms with thresholds that cause excessive false positives

## Phase 3 — Rollback

1. If alarms were deleted, recreate from documented configurations
2. If log groups were modified, restore previous retention settings
3. If custom metrics were removed, re-enable the publishing mechanism
4. Verify all monitoring is restored and functioning
5. Document monitoring configuration for disaster recovery

## Output Format

```yaml
root_cause: "monitoring_integration — <specific_cause>"
evidence:
  - type: monitoring_status
    content: "<metric availability and log group status>"
severity: MEDIUM
mitigation:
  immediate: "Restore monitoring visibility"
  long_term: "Implement comprehensive monitoring with alerting"
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
