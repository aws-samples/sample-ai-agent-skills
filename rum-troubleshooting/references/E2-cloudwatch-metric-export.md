---
title: "E2 — CloudWatch Metric Export"
description: "Diagnose CloudWatch metric export configuration and delivery issues"
status: active
severity: MEDIUM
triggers: ["RUM metrics not in CloudWatch", "metric export failed", "no RUM metrics", "metric definition error"]
owner: devops-agent
objective: "Resolve CloudWatch metric export configuration for RUM data"
context: "RUM data is not automatically exported as CloudWatch metrics. Export requires configuring a metrics destination and defining metric definitions."
---

## Phase 1 — Triage

MUST:
- Check metric destinations: `aws rum list-rum-metrics-destinations --app-monitor-name <name>`
- Check metric definitions: `aws rum batch-get-rum-metric-definitions --app-monitor-name <name> --destination CloudWatch`
- Verify the app monitor is ACTIVE and collecting data

SHOULD:
- Check if CloudWatch metrics exist: `aws cloudwatch list-metrics --namespace AWS/RUM --dimensions Name=AppMonitorName,Value=<name>`
- Verify IAM permissions for cloudwatch:PutMetricData
- Check if the metric definitions match the expected RUM event types

MAY:
- Review CloudTrail for metric export-related errors
- Check CloudWatch metric data for the expected time range

## Phase 2 — Remediate

MUST:
- Configure metric destination if not set: `aws rum put-rum-metrics-destination --app-monitor-name <name> --destination CloudWatch`
- Create metric definitions: `aws rum batch-create-rum-metric-definitions --app-monitor-name <name> --destination CloudWatch --metric-definitions '[{"Name":"<metric-name>","EventPattern":"<pattern>"}]'`
- Verify IAM permissions for metric export

SHOULD:
- Define metrics for key performance indicators (page load time, errors, Web Vitals)
- Set up CloudWatch alarms on exported metrics
- Verify metrics appear in CloudWatch after configuration

MAY:
- Create CloudWatch dashboards using exported RUM metrics
- Configure metric math for derived metrics

## Common Issues

- symptoms: "Metric destination configured but no metrics appearing in CloudWatch"
  diagnosis: "Metric definitions may not be created, or the event pattern doesn't match collected data."
  resolution: "Create metric definitions: `aws rum batch-create-rum-metric-definitions --app-monitor-name <name> --destination CloudWatch --metric-definitions '[{\"Name\":\"PageLoadTime\",\"EventPattern\":\"{\\\"event_type\\\":[\\\"com.amazon.rum.performance_navigation_event\\\"]}\"}]'`"

## Safety Ratings

```
safety_ratings:
  - "list-rum-metrics-destinations: GREEN — read-only destination listing"
  - "batch-get-rum-metric-definitions: GREEN — read-only definition listing"
  - "put-rum-metrics-destination: YELLOW — configures metric export"
  - "batch-create-rum-metric-definitions: YELLOW — creates metric definitions"
```

## Escalation Conditions

- Metric export configured but metrics consistently not appearing in CloudWatch
- Metric definitions accepted but producing incorrect values
- Metric export causing unexpected CloudWatch costs
- Custom metric definitions needed for business-specific KPIs

## Data Sensitivity

- `aws rum list-rum-metrics-destinations`: LOW — returns destination configuration
- `aws rum batch-get-rum-metric-definitions`: LOW — returns metric definition metadata
- CloudWatch metrics: LOW — contains aggregated performance data

## Prohibited Actions

- NEVER create excessive metric definitions that inflate CloudWatch costs
- NEVER delete metric definitions that feed production dashboards or alarms
- NEVER modify metric event patterns without understanding the impact on existing alarms
- NEVER assume metrics are exported automatically without explicit configuration

## Phase 3 — Rollback

1. If metric definitions were created in error, delete them: `aws rum batch-delete-rum-metric-definitions --app-monitor-name <name> --destination CloudWatch --metric-definition-ids <ids>`
2. If the metric destination was configured in error, remove it: `aws rum delete-rum-metrics-destination --app-monitor-name <name> --destination CloudWatch`
3. Verify CloudWatch alarms are not affected by metric definition changes
4. Document all metric export configuration changes
5. Confirm existing dashboards still function correctly

## Output Format

```yaml
root_cause: "cloudwatch_metric_export — <specific_cause>"
evidence:
  - type: export_error
    content: "<metric destination and definition details>"
severity: MEDIUM
mitigation:
  immediate: "Fix metric export configuration"
  long_term: "Automate metric definition management via IaC"
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
