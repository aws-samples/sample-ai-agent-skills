---
title: "F1 — CloudWatch Monitoring"
description: "Diagnose MediaConnect CloudWatch monitoring and alerting issues"
status: active
severity: MEDIUM
triggers: ["no metrics", "missing CloudWatch data", "alarm not triggering", "monitoring gap", "metrics delayed"]
owner: devops-agent
objective: "Ensure CloudWatch metrics and alarms are correctly configured for MediaConnect flows"
context: "MediaConnect publishes metrics to CloudWatch at 1-minute intervals. Key metrics include SourceBitRate, OutputBitRate, TransportStreamError, and packet loss percentages."
---

## Phase 1 — Triage

MUST:
- Verify metrics are being published: `aws cloudwatch list-metrics --namespace AWS/MediaConnect --dimensions Name=FlowARN,Value=<flow-arn>`
- Check a key metric: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name SourceBitRate --dimensions Name=FlowARN,Value=<flow-arn> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average`
- Verify flow is ACTIVE (metrics only publish for active flows): `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Status'`

SHOULD:
- Check existing alarms: `aws cloudwatch describe-alarms --alarm-name-prefix MediaConnect`
- Verify alarm actions (SNS topics) are configured and subscriptions confirmed
- Check if the correct dimensions are used (FlowARN, SourceName, OutputName)

MAY:
- Review CloudWatch dashboard for MediaConnect if one exists
- Check CloudWatch Logs for flow-level logging if enabled

## Phase 2 — Remediate

MUST:
- Ensure flow is ACTIVE — stopped flows do not publish metrics
- Create alarms for critical metrics: `aws cloudwatch put-metric-alarm --alarm-name MediaConnect-SourceBitRate-Low --namespace AWS/MediaConnect --metric-name SourceBitRate --dimensions Name=FlowARN,Value=<flow-arn> --statistic Average --period 300 --threshold 1000000 --comparison-operator LessThanThreshold --evaluation-periods 3 --alarm-actions <sns-topic-arn>`
- Verify SNS topic subscriptions are confirmed

SHOULD:
- Create alarms for: SourceBitRate (low), TransportStreamError (high), SourcePacketLossPercent (high), OutputBitRate (low)
- Set appropriate thresholds based on expected content bitrate
- Use composite alarms for complex conditions

MAY:
- Create a CloudWatch dashboard for MediaConnect flows
- Enable anomaly detection on SourceBitRate for dynamic thresholds
- Set up CloudWatch Logs Insights queries for flow logs

## Safety Ratings

```
safety_ratings:
  - "list-metrics: GREEN — read-only metric enumeration"
  - "get-metric-statistics: GREEN — read-only metric query"
  - "put-metric-alarm: YELLOW — creates alarm that may trigger notifications"
  - "describe-alarms: GREEN — read-only alarm inspection"
```

## Escalation Conditions

- No metrics published for an ACTIVE flow for more than 5 minutes
- CloudWatch metrics show data but alarms do not trigger at expected thresholds
- SNS notifications are not being delivered despite alarm state changes
- Metrics show unexpected values that don't match actual flow behavior

## Data Sensitivity

- `aws cloudwatch list-metrics`: LOW — returns metric names and dimensions
- `aws cloudwatch get-metric-statistics`: LOW — returns numeric metric values
- `aws cloudwatch put-metric-alarm`: LOW — creates monitoring configuration
- `aws cloudwatch describe-alarms`: LOW — returns alarm configuration

## Prohibited Actions

- NEVER delete existing production alarms without creating replacements first
- NEVER set alarm thresholds to zero — this causes constant alarm state
- NEVER use insufficient evaluation periods (use at least 3) to avoid false alarms
- NEVER ignore missing metrics — they may indicate a flow health issue

## Phase 3 — Rollback

1. If an alarm was misconfigured, update it: `aws cloudwatch put-metric-alarm` with corrected parameters
2. If an alarm was deleted, recreate it with the original configuration
3. If SNS topic was changed, update alarm actions: `aws cloudwatch put-metric-alarm --alarm-name <name> --alarm-actions <correct-sns-arn>`
4. Verify alarms return to OK state after fixing thresholds
5. Document the monitoring change and update the runbook

## Common Issues

- symptoms: "No CloudWatch metrics for a MediaConnect flow"
  diagnosis: "Flow is in STANDBY state or was recently created."
  resolution: "Start the flow. Metrics publish only for ACTIVE flows. Allow 1-2 minutes after starting for first data points."

- symptoms: "Alarm does not trigger when source bitrate drops"
  diagnosis: "Alarm threshold, period, or evaluation periods are misconfigured."
  resolution: "Verify threshold matches expected bitrate. Use at least 3 evaluation periods with 60-second period for responsive alerting."

- symptoms: "Alarm triggers but no SNS notification received"
  diagnosis: "SNS topic subscription is not confirmed or alarm actions are not set."
  resolution: "Verify SNS subscription is confirmed. Check alarm AlarmActions includes the SNS topic ARN."

## Output Format

```yaml
root_cause: "monitoring — <specific_cause>"
evidence:
  - type: metric_availability
    content: "<list-metrics output or missing metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix monitoring configuration"
  long_term: "Deploy comprehensive CloudWatch alarms via IaC for all flows"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
