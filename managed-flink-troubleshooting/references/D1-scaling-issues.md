---
title: "D1 — Scaling Issues"
description: "Diagnose Managed Flink application scaling and auto-scaling issues"
status: active
severity: HIGH
triggers: ["scaling failed", "auto-scaling not working", "KPU limit", "cannot scale"]
owner: devops-agent
objective: "Resolve scaling issues including auto-scaling configuration, KPU limits, and scaling downtime"
context: "Scaling changes parallelism and KPU count. Requires application restart with snapshot. Auto-scaling adjusts based on CPU utilization."
---

## Phase 1 — Triage

MUST:
- Check current parallelism: `aws kinesisanalyticsv2 describe-application --application-name <app-name> --query 'ApplicationDetail.ApplicationConfigurationDescription.FlinkApplicationConfigurationDescription.ParallelismConfigurationDescription'`
- Check KPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name KPUs --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name cpuUtilization --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Verify auto-scaling is enabled in application configuration
- Check service quota for maximum KPUs

MAY:
- Review scaling history in CloudTrail
- Check if source partitions limit effective parallelism

## Phase 2 — Remediate

MUST:
- Enable auto-scaling if needed: set AutoScalingEnabled to true
- Adjust parallelism to match workload requirements
- Request KPU quota increase if at the limit

SHOULD:
- Match parallelism to source partition count for optimal throughput
- Monitor scaling events and their impact on processing

MAY:
- Pre-provision parallelism for known traffic patterns
- Configure scaling policies based on custom metrics

## Safety Ratings

```
safety_ratings:
  - "describe-application: GREEN — read-only application details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "update-application (parallelism): YELLOW — triggers application restart"
  - "service-quotas request-service-quota-increase: YELLOW — requests quota change"
```

## Escalation Conditions

- Auto-scaling causes frequent restarts that disrupt processing
- KPU quota increase request is denied
- Scaling downtime exceeds acceptable data processing delay
- Scaling does not improve throughput (bottleneck is elsewhere)

## Data Sensitivity

- `aws kinesisanalyticsv2 describe-application`: MEDIUM — reveals scaling configuration
- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics only
- Scaling events: LOW — operational metadata

## Prohibited Actions

- NEVER scale to maximum KPUs without understanding cost implications
- NEVER change parallelism during peak processing without a snapshot
- NEVER disable auto-scaling without manual monitoring in place
- NEVER assume more KPUs always means better performance

## Phase 3 — Rollback

1. If parallelism was changed incorrectly, revert to previous value
2. If auto-scaling was enabled/disabled, restore original setting
3. Verify application restarts successfully after scaling changes
4. Check for data gaps during scaling downtime
5. Document optimal parallelism settings for the workload

## Common Issues

- symptoms: "Application restarts during auto-scaling and loses processing progress"
  diagnosis: "Checkpointing is failing, so scaling restart cannot restore state."
  resolution: "Fix checkpoint failures first. Ensure checkpoints succeed before enabling auto-scaling."

## Output Format

```yaml
root_cause: "scaling — <specific_cause>"
evidence:
  - type: scaling_metric
    content: "<metric details>"
severity: HIGH
mitigation:
  immediate: "Fix scaling issue"
  long_term: "Optimize parallelism"
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
