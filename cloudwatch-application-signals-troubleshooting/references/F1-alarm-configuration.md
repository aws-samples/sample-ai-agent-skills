---
title: "F1 — Alarm Configuration"
description: "Diagnose CloudWatch Application Signals alarm configuration issues"
status: active
severity: MEDIUM
triggers: ["alarm not firing", "SLO alarm", "error budget alarm", "Application Signals alert"]
owner: devops-agent
objective: "Resolve alarm configuration issues for Application Signals and SLOs"
context: "SLOs do not automatically create alarms. Alarms must be configured separately using CloudWatch Alarms."
---

## Phase 1 — Triage

MUST:
- List alarms for Application Signals metrics: `aws cloudwatch describe-alarms --alarm-name-prefix 'ApplicationSignals' --query 'MetricAlarms[*].{Name:AlarmName,State:StateValue,Metric:MetricName}'`
- Verify alarm metric namespace is AWS/ApplicationSignals
- Check alarm threshold and evaluation period configuration

SHOULD:
- Verify alarm actions (SNS topic, Lambda, etc.) are configured
- Check if the alarm has sufficient data points for evaluation
- Review alarm history: `aws cloudwatch describe-alarm-history --alarm-name <alarm-name>`

MAY:
- Check SNS topic subscription for alarm notifications
- Review composite alarm configuration for SLO-based alerting

## Phase 2 — Remediate

MUST:
- Create alarms for SLO metrics if missing
- Fix alarm threshold or evaluation period if misconfigured
- Verify alarm actions point to valid SNS topics or targets

SHOULD:
- Create error budget consumption rate alarms (burn rate alerts)
- Use composite alarms for multi-signal alerting

MAY:
- Implement alarm suppression during maintenance windows
- Create dashboards combining SLO status and alarm state

## Common Issues

- symptoms: "SLO is breached but no alarm fires"
  diagnosis: "No CloudWatch alarm was created for the SLO. SLOs do not auto-create alarms."
  resolution: "Create a CloudWatch alarm using the SLO metric. Configure appropriate threshold and actions."

- symptoms: "Alarm is in INSUFFICIENT_DATA state"
  diagnosis: "Application Signals metrics are not flowing or alarm metric configuration is wrong."
  resolution: "Verify metrics exist in AWS/ApplicationSignals namespace. Check alarm metric name and dimensions match."

## Safety Ratings

```
safety_ratings:
  - "describe-alarms: GREEN — read-only alarm listing"
  - "describe-alarm-history: GREEN — read-only alarm history"
  - "put-metric-alarm: YELLOW — creates or modifies alarm"
  - "delete-alarms: RED — removes monitoring alerts"
```

## Escalation Conditions

- Critical SLO breaches are not generating alerts
- Alarm actions fail to deliver notifications
- Alarm configuration requires cross-account SNS access
- Complex composite alarm logic needed for error budget policies

## Data Sensitivity

- `aws cloudwatch describe-alarms`: LOW — returns alarm configuration
- `aws cloudwatch describe-alarm-history`: LOW — returns alarm state changes
- SNS topic configuration: MEDIUM — may reveal notification endpoints
- Alarm thresholds: LOW — contain performance targets

## Prohibited Actions

- NEVER delete alarms for production SLOs without replacement
- NEVER set alarm evaluation periods shorter than metric collection intervals
- NEVER configure alarms that auto-remediate without human approval for critical services
- NEVER ignore INSUFFICIENT_DATA alarm states — they indicate monitoring gaps

## Phase 3 — Rollback

1. If alarm was misconfigured, update it: `aws cloudwatch put-metric-alarm --alarm-name <name> --metric-name <metric> --namespace AWS/ApplicationSignals --threshold <original-threshold> --comparison-operator <operator> --evaluation-periods <periods> --period <period> --statistic <stat>`
2. If alarm was deleted, recreate it with the original configuration
3. Verify alarm state transitions correctly after fix
4. Test alarm by temporarily lowering threshold to trigger
5. Document alarm configuration and notification targets

## Output Format

```yaml
root_cause: "alarm_configuration — <specific_cause>"
evidence:
  - type: alarm_error
    content: "<configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix alarm configuration"
  long_term: "Implement alarm-as-code via CloudFormation"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
