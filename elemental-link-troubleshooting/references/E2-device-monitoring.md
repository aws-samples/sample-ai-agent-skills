---
title: "E2 — Device Monitoring"
description: "Diagnose Elemental Link monitoring and alerting gaps"
status: active
severity: LOW
triggers: ["no metrics", "monitoring not working", "alerts missing", "CloudWatch empty", "no device data"]
owner: devops-agent
objective: "Establish and troubleshoot monitoring for Elemental Link devices"
context: "Elemental Link publishes metrics to CloudWatch under the AWS/MediaLive namespace. Metrics include frame rate, network throughput, and connection state. Custom alarms can be configured."
---

## Phase 1 — Triage

MUST:
- Check if metrics are being published: `aws cloudwatch list-metrics --namespace AWS/MediaLive --dimensions Name=InputDeviceId,Value=<device-id>`
- Verify device is connected and streaming: `aws medialive describe-input-device --input-device-id <device-id> --query 'ConnectionState'`
- Check existing alarms: `aws cloudwatch describe-alarms --alarm-name-prefix elemental-link`

SHOULD:
- Verify the correct namespace and dimension names are being used in queries
- Check if the device has been active recently (metrics only publish during active streaming)

MAY:
- Review CloudWatch dashboard configurations for Elemental Link widgets
- Check IAM permissions for CloudWatch access

## Phase 2 — Remediate

MUST:
- Verify the device is connected and actively streaming (metrics require active streaming)
- Use correct namespace (AWS/MediaLive) and dimension (InputDeviceId) for queries
- Create basic connectivity alarm: `aws cloudwatch put-metric-alarm --alarm-name link-<device-id>-connectivity --namespace AWS/MediaLive --metric-name NetworkIn --dimensions Name=InputDeviceId,Value=<device-id> --comparison-operator LessThanThreshold --threshold 1 --evaluation-periods 3 --period 60 --alarm-actions <sns-topic-arn>`

SHOULD:
- Create frame rate monitoring alarm for quality tracking
- Set up a CloudWatch dashboard for fleet-wide device monitoring
- Configure SNS notifications for alarm state changes

MAY:
- Implement composite alarms combining connectivity and quality metrics
- Create automated remediation using CloudWatch alarm actions
- Export metrics to third-party monitoring tools

## Safety Ratings

```
safety_ratings:
  - "cloudwatch list-metrics: GREEN — read-only metrics enumeration"
  - "describe-input-device: GREEN — read-only device status"
  - "describe-alarms: GREEN — read-only alarm listing"
  - "put-metric-alarm: YELLOW — creates monitoring resource, may trigger notifications"
```

## Escalation Conditions

- No metrics published despite confirmed active streaming
- CloudWatch metrics show data gaps during known streaming periods
- Alarm notifications not being delivered despite correct SNS configuration
- Need for custom metrics not available in the standard AWS/MediaLive namespace

## Data Sensitivity

- `aws cloudwatch list-metrics`: LOW — returns metric names and dimensions
- `aws cloudwatch describe-alarms`: LOW — returns alarm configurations
- `aws cloudwatch put-metric-alarm`: LOW — creates monitoring resources
- SNS topic ARN: MEDIUM — reveals notification infrastructure

## Prohibited Actions

- NEVER create alarms with overly sensitive thresholds that cause alert fatigue
- NEVER delete existing production alarms without understanding their purpose
- NEVER publish custom metrics using the AWS/MediaLive namespace (reserved for AWS)
- NEVER configure alarm actions that modify device settings automatically without approval

## Phase 3 — Rollback

1. Delete alarms created in error: `aws cloudwatch delete-alarms --alarm-names <alarm-name>`
2. Remove dashboard widgets if they display incorrect data
3. Revert SNS subscription changes if notifications were misconfigured
4. Restore previous alarm thresholds if they were modified
5. Document the monitoring configuration for the operations team

## Common Issues

- symptoms: "CloudWatch shows no metrics for the device"
  diagnosis: "Device is not actively streaming — metrics only publish during active encoding."
  resolution: "Start a MediaLive channel using the device input. Metrics appear within minutes of streaming."

## Output Format

```yaml
root_cause: "device_monitoring — <specific_cause>"
evidence:
  - type: monitoring_gap
    content: "<details>"
severity: LOW
mitigation:
  immediate: "Fix monitoring configuration"
  long_term: "Implement comprehensive fleet monitoring with dashboards and alarms"
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
