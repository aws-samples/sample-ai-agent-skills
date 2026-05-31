---
title: "E2 — Outpost Monitoring Issues"
description: "Diagnose monitoring gaps and metric collection problems on AWS Outposts"
status: active
severity: MEDIUM
triggers:
  - "outpost monitoring"
  - "missing metrics"
  - "CloudWatch outpost"
  - "outpost alarms"
owner: devops-agent
objective: "Ensure comprehensive monitoring coverage for Outpost infrastructure and workloads"
context: "Outpost metrics are sent to CloudWatch in the parent Region via the service link. If the service link is down, metrics are buffered locally and sent when connectivity is restored. Some metrics require explicit enablement."
---

## Phase 1 — Triage

MUST:
- Check available Outpost metrics: `aws cloudwatch list-metrics --namespace AWS/Outposts`
- Verify ConnectedStatus metric: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check EC2 instance metrics: `aws cloudwatch list-metrics --namespace AWS/EC2 --dimensions Name=InstanceId,Value=<instance-id>`
- Verify CloudWatch agent status on instances if detailed metrics needed

SHOULD:
- Check for metric gaps during service link outages
- Verify alarm configurations: `aws cloudwatch describe-alarms --alarm-name-prefix "Outpost"`
- Review dashboard configurations for Outpost resources

MAY:
- Check CloudWatch Logs for application-level monitoring
- Review Container Insights if running ECS on Outposts

## Phase 2 — Remediate

MUST:
- If metrics missing: verify service link connectivity (metrics require it for delivery)
- If alarms not firing: check alarm configuration and SNS topic permissions
- Ensure CloudWatch agent is installed for detailed OS-level metrics

SHOULD:
- Create CloudWatch dashboards for Outpost capacity and connectivity
- Set up alarms for ConnectedStatus, InstanceFamilyCapacityAvailability
- Configure SNS notifications for critical Outpost events

MAY:
- Implement custom metrics for application-specific monitoring
- Set up AWS Health event notifications via EventBridge

## Common Issues

- symptoms: "No Outpost metrics in CloudWatch"
  diagnosis: "Service link is down — metrics cannot be delivered to the Region."
  resolution: "Restore service link. Metrics will be backfilled when connectivity returns."

- symptoms: "Capacity alarms not triggering"
  diagnosis: "Alarm threshold or metric dimension misconfigured."
  resolution: "Verify alarm uses correct OutpostId dimension and appropriate threshold."

## Output Format

```yaml
root_cause: "monitoring — <specific_cause>"
evidence:
  - type: available_metrics
    content: "<CloudWatch metrics list>"
  - type: alarm_config
    content: "<alarm configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix metric collection or alarm configuration"
  long_term: "Implement comprehensive Outpost monitoring dashboard"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — monitoring data is operational metrics
- Service Disruption Risk: LOW — monitoring issues do not affect workloads
- Compliance Impact: MEDIUM — monitoring gaps affect operational visibility and audit

## Escalation Conditions
- All monitoring data stops being collected
- CloudWatch metrics delivery from Outpost fails
- Monitoring gaps during critical operational periods
- Alert configurations stop functioning
- Monitoring agent failures on Outpost instances

## Data Sensitivity
- Performance metrics — INTERNAL
- Health check data — INTERNAL
- Alert configurations — INTERNAL
- Monitoring agent configurations — INTERNAL
- Never expose monitoring data that reveals infrastructure vulnerabilities

## Prohibited Actions
- Do NOT disable monitoring on production Outposts
- Do NOT ignore monitoring gaps
- Do NOT grant broad access to monitoring data
- Do NOT delete monitoring configurations without backup
- Do NOT bypass monitoring alert thresholds
- Do NOT expose monitoring dashboards to unauthorized users

## Phase 3 — Rollback
- Restore previous monitoring configurations
- Re-enable disabled monitoring agents
- Restore alert configurations
- Verify monitoring data collection after changes
- Restore CloudWatch metric delivery
- Document monitoring issues and configuration changes

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
