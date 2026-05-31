---
title: "B1 — Missing Metrics"
description: "Diagnose why expected CloudWatch metrics are not appearing"
status: active
severity: HIGH
triggers:
  - "metrics missing"
  - "no metric data"
  - "metric not found"
  - "empty graph"
  - "metric namespace"
owner: devops-agent
objective: "Identify why expected metrics are not visible in CloudWatch"
context: "Missing metrics are caused by incorrect namespace/dimension combinations, metric not yet published, detailed monitoring not enabled, wrong region, or the resource not actively emitting data. AWS services publish metrics at different intervals — EC2 basic monitoring is 5 minutes, detailed is 1 minute."
---

## Phase 1 — Triage

MUST:
- Verify the metric exists: `aws cloudwatch list-metrics --namespace <namespace> --metric-name <metric-name>`
- Check dimensions are correct: `aws cloudwatch list-metrics --namespace <namespace> --metric-name <metric-name> --dimensions Name=<key>,Value=<val>`
- Query for data: `aws cloudwatch get-metric-statistics --namespace <ns> --metric-name <metric> --dimensions Name=<key>,Value=<val> --start-time <iso> --end-time <iso> --period 300 --statistics Average`
- Verify the region matches the resource region

SHOULD:
- Check if detailed monitoring is enabled (EC2): `aws ec2 describe-instances --instance-ids <id> --query 'Reservations[].Instances[].Monitoring'`
- Verify the resource is running and actively emitting data
- Check metric retention: 1-min data retained 15 days, 5-min for 63 days, 1-hour for 455 days

MAY:
- Check if the metric was recently created (may take a few minutes to appear)
- Verify custom namespace spelling (case-sensitive)

## Phase 2 — Remediate

MUST:
- Fix namespace and dimension names (case-sensitive exact match required)
- Enable detailed monitoring if 1-minute data is needed
- Ensure the resource is in the expected region

SHOULD:
- Use list-metrics to discover available dimensions for the metric
- Widen the time range to find when data was last published

MAY:
- Set up a missing data alarm to detect future metric gaps

## Common Issues

- symptoms: "EC2 CPUUtilization shows no data at 1-minute period"
  diagnosis: "Basic monitoring is enabled (5-minute intervals). 1-minute data requires detailed monitoring."
  resolution: "Enable detailed monitoring: aws ec2 monitor-instances --instance-ids <id>"

- symptoms: "Custom metric not appearing in console"
  diagnosis: "Namespace or dimension name is misspelled (case-sensitive)."
  resolution: "Use list-metrics to find the exact namespace and dimensions."

- symptoms: "Metric existed before but now shows no data"
  diagnosis: "Data retention expired. 1-minute data is only retained for 15 days."
  resolution: "Query at a coarser period (5-min or 1-hour) for older data."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix namespace and dimension names | GREEN | Query correction, no state change |
| Enable detailed monitoring | YELLOW | Increases monitoring cost; changes metric resolution |
| Widen time range for queries | GREEN | Read-only query adjustment |

## Escalation Conditions

- Production monitoring dashboards showing no data — escalate to operations team
- Missing metrics affecting alarm evaluation — escalate to monitoring team
- Detailed monitoring enablement across fleet — escalate to cost management team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudwatch list-metrics` | LOW | Metric names and dimensions only |
| `aws cloudwatch get-metric-statistics` | LOW | Numeric metric data |
| `aws ec2 describe-instances` (monitoring) | MEDIUM | Instance configuration |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** disable detailed monitoring on instances that depend on 1-minute metrics for alarms

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled detailed monitoring | Disable: `aws ec2 unmonitor-instances --instance-ids <id>` |
| No state-changing actions for query fixes | No rollback needed |

## Output Format

```yaml
root_cause: "missing_metrics — <specific_cause>"
evidence:
  - type: metric_query
    content: "<list-metrics or get-metric-statistics output>"
  - type: resource_config
    content: "<monitoring configuration>"
severity: HIGH
mitigation:
  immediate: "Fix namespace, dimensions, or enable detailed monitoring"
  long_term: "Document metric naming conventions and set up missing data alarms"
```

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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
