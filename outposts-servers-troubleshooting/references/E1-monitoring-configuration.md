---
title: "E1 — Monitoring Configuration"
description: "Diagnose monitoring and observability gaps on Outposts Servers"
status: active
severity: MEDIUM
triggers: ["no metrics from outpost", "CloudWatch not showing outpost data", "monitoring gap", "alerting not working for outpost"]
owner: devops-agent
objective: "Configure and troubleshoot monitoring for Outposts Servers"
context: "CloudWatch metrics for Outposts require an active service link. During disconnection, metrics are not published. Local monitoring alternatives should be considered for edge scenarios."
---

## Phase 1 — Triage

MUST:
- Check ConnectedStatus metric: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- List CloudWatch alarms for Outpost: `aws cloudwatch describe-alarms --alarm-name-prefix "Outpost" --query 'MetricAlarms[*].{Name:AlarmName,State:StateValue,Metric:MetricName}'`
- Check EC2 instance metrics: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Verify CloudWatch agent is installed on Outpost instances
- Check if custom metrics are being published
- Review alarm configurations for appropriate thresholds

MAY:
- Set up CloudWatch dashboards for Outpost monitoring
- Configure SNS notifications for alarm state changes

## Phase 2 — Remediate

MUST:
- Ensure service link is active for CloudWatch metric delivery
- Configure ConnectedStatus alarm for service link monitoring
- Set up basic EC2 instance monitoring (CPU, disk, network)

SHOULD:
- Install CloudWatch agent on instances for detailed OS-level metrics
- Configure alarms for capacity thresholds (CPU, memory, storage)

MAY:
- Implement local monitoring solutions for disconnected scenarios
- Create CloudWatch dashboards aggregating all Outpost metrics

## Common Issues

- symptoms: "No CloudWatch metrics appearing for Outpost instances"
  diagnosis: "Service link is down or CloudWatch agent not installed."
  resolution: "Verify service link connectivity. Install CloudWatch agent on instances. Check IAM role has cloudwatch:PutMetricData permission."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metric retrieval"
  - "cloudwatch describe-alarms: GREEN — read-only alarm enumeration"
  - "cloudwatch put-metric-alarm: YELLOW — creates alarms that may trigger notifications"
  - "CloudWatch agent installation: YELLOW — installs software on instances"
```

## Escalation Conditions

- Complete loss of monitoring visibility for production Outpost workloads
- Service link disconnection causing extended monitoring gaps
- CloudWatch agent failures across multiple instances
- Alarm fatigue from misconfigured thresholds

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — returns performance metrics
- `aws cloudwatch describe-alarms`: LOW — returns alarm configurations
- CloudWatch agent logs: MEDIUM — may contain application-level details
- Custom metrics: MEDIUM — may reveal business-specific operational data

## Prohibited Actions

- NEVER disable monitoring on production Outpost instances
- NEVER delete CloudWatch alarms without documenting the reason and creating replacements
- NEVER assume metrics are available during service link disconnection
- NEVER configure alarms with overly sensitive thresholds that cause alert fatigue

## Phase 3 — Rollback

1. If alarms were deleted, recreate them from documented configurations
2. If CloudWatch agent was misconfigured, restore the previous agent configuration file
3. If custom metrics were removed, re-enable the publishing mechanism
4. Verify all monitoring is restored and alarms are in the correct state
5. Document monitoring configuration for disaster recovery

## Output Format

```yaml
root_cause: "monitoring_configuration — <specific_cause>"
evidence:
  - type: monitoring_status
    content: "<metric availability and alarm states>"
severity: MEDIUM
mitigation:
  immediate: "Restore monitoring visibility"
  long_term: "Implement comprehensive monitoring with disconnected fallback"
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
