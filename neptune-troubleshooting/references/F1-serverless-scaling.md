---
title: "F1 — Neptune Serverless Scaling"
description: "Diagnose Neptune Serverless scaling issues"
status: active
severity: HIGH
triggers:
  - "serverless scaling"
  - "ncu scaling"
  - "auto scaling neptune"
  - "serverless slow"
owner: devops-agent
objective: "Identify and resolve Neptune Serverless scaling issues"
context: "Neptune Serverless scales in Neptune Capacity Units (NCUs). Each NCU provides ~2 GiB memory. Scaling is automatic between min/max NCU settings but not instantaneous."
---

## Phase 1 — Triage

MUST:
- Check serverless configuration: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].ServerlessV2ScalingConfiguration'`
- Check current NCU usage: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name ServerlessDatabaseCapacity --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name CPUUtilization --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify min/max NCU settings are appropriate

SHOULD:
- Check scaling events in cluster events: `aws neptune describe-events --source-type db-cluster --duration 1440`
- Monitor NCU consumption pattern over time
- Check for query timeouts during scaling events
- Verify engine version supports serverless

MAY:
- Compare NCU usage with query patterns
- Test scaling behavior with load testing
- Review cost implications of NCU settings

## Phase 2 — Remediate

MUST:
- Adjust min NCU to handle baseline workload without scaling
- Set max NCU high enough for peak workload
- Verify scaling resolves performance issues

SHOULD:
- Set min NCU to avoid cold start latency for consistent workloads
- Monitor ServerlessDatabaseCapacity metric
- Set up CloudWatch alarms for NCU thresholds

MAY:
- Consider provisioned instances for predictable workloads
- Implement query optimization to reduce NCU consumption

## Common Issues

- symptoms: "Slow queries during traffic spikes"
  diagnosis: "Min NCU too low, causing scaling delay during sudden load increase."
  resolution: "Increase min NCU to handle expected baseline traffic."

- symptoms: "NCU at maximum but still slow"
  diagnosis: "Max NCU insufficient for workload or query optimization needed."
  resolution: "Increase max NCU or optimize queries to reduce resource consumption."

- symptoms: "High cost with serverless"
  diagnosis: "Min NCU set too high or workload is consistently high."
  resolution: "Lower min NCU for variable workloads or switch to provisioned for steady workloads."

## Output Format

```yaml
root_cause: "serverless_scaling — <specific_cause>"
evidence:
  - type: serverless_config
    content: "<min/max NCU settings>"
  - type: ncu_metrics
    content: "<ServerlessDatabaseCapacity over time>"
severity: HIGH
mitigation:
  immediate: "Adjust NCU settings"
  long_term: "Monitor and optimize NCU consumption patterns"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — scaling operations do not expose data
- Service Disruption Risk: MEDIUM — scaling issues can cause performance degradation
- Compliance Impact: LOW — scaling is an operational concern

## Escalation Conditions
- Serverless scaling fails to respond to increased workload
- Scaling causes connection drops or query failures
- Minimum NCU setting is insufficient for baseline workload
- Maximum NCU limit reached and queries are being throttled
- Unexpected cost increases due to scaling behavior

## Data Sensitivity
- NCU capacity metrics — INTERNAL
- Scaling event logs — INTERNAL
- Cost and utilization data — INTERNAL
- Capacity configuration — INTERNAL
- Never expose internal scaling thresholds

## Prohibited Actions
- Do NOT set minimum NCUs to zero for production workloads
- Do NOT set maximum NCUs without understanding cost implications
- Do NOT change scaling configuration during active performance incidents
- Do NOT ignore scaling alerts and capacity warnings
- Do NOT bypass serverless scaling limits without AWS Support guidance
- Do NOT use serverless for workloads with strict latency requirements without testing

## Phase 3 — Rollback
- Revert NCU minimum and maximum settings to previous values
- Monitor scaling behavior after configuration changes
- Restore previous capacity configuration if changes caused issues
- Verify query performance after scaling configuration rollback
- Document scaling behavior patterns for capacity planning
- Update cost alerts based on scaling configuration changes

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
