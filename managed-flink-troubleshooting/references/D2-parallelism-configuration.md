---
title: "D2 — Parallelism Configuration"
description: "Diagnose Managed Flink parallelism and KPU configuration issues"
status: active
severity: MEDIUM
triggers: ["parallelism error", "KPU configuration", "ParallelismPerKPU", "resource allocation"]
owner: devops-agent
objective: "Resolve parallelism configuration issues including Parallelism, ParallelismPerKPU, and resource allocation"
context: "Total parallelism = Parallelism × ParallelismPerKPU. Each KPU provides 1 vCPU and 4 GB memory. Incorrect configuration wastes resources or causes bottlenecks."
---

## Phase 1 — Triage

MUST:
- Check parallelism configuration: `aws kinesisanalyticsv2 describe-application --application-name <app-name> --query 'ApplicationDetail.ApplicationConfigurationDescription.FlinkApplicationConfigurationDescription.ParallelismConfigurationDescription'`
- Calculate total parallelism: Parallelism × ParallelismPerKPU
- Compare total parallelism with source partition count

SHOULD:
- Check CPU and memory utilization per KPU
- Verify parallelism matches the number of source shards/partitions

MAY:
- Review operator-level parallelism in Flink job graph
- Check if any operators are bottlenecked

## Phase 2 — Remediate

MUST:
- Set Parallelism to match source partition count (e.g., Kinesis shard count)
- Set ParallelismPerKPU based on memory requirements (1 = 4 GB per task, 2 = 2 GB per task)
- Ensure total KPU count is within service quota

SHOULD:
- Use ConfigurationType CUSTOM for explicit control
- Monitor resource utilization after changes

MAY:
- Set operator-level parallelism in Flink code for fine-grained control
- Use slot sharing groups for resource isolation

## Safety Ratings

```
safety_ratings:
  - "describe-application: GREEN — read-only application details"
  - "update-application (parallelism): YELLOW — modifies resource allocation, triggers restart"
  - "create-application-snapshot: YELLOW — captures state before changes"
  - "start-application: YELLOW — starts application with new configuration"
```

## Escalation Conditions

- Parallelism changes do not improve throughput
- Memory per task is insufficient for state size
- Source partition count changes frequently requiring parallelism adjustments
- KPU cost exceeds budget at required parallelism

## Data Sensitivity

- `aws kinesisanalyticsv2 describe-application`: MEDIUM — reveals resource configuration
- Parallelism settings: LOW — operational configuration
- KPU utilization metrics: LOW — aggregated resource metrics

## Prohibited Actions

- NEVER set ParallelismPerKPU higher than 8 without understanding memory implications
- NEVER change parallelism without creating a snapshot first
- NEVER set parallelism higher than source partition count (wastes KPUs)
- NEVER use DEFAULT ConfigurationType for production applications

## Phase 3 — Rollback

1. Create snapshot before changes: `aws kinesisanalyticsv2 create-application-snapshot --application-name <name> --snapshot-name pre-parallelism-change`
2. If parallelism change caused issues, revert to previous values
3. Restart application from the pre-change snapshot
4. Verify application processes data correctly after rollback
5. Document optimal parallelism configuration

## Common Issues

- symptoms: "Application uses more KPUs than expected, high cost"
  diagnosis: "ParallelismPerKPU is set to 1 with high Parallelism, resulting in many KPUs."
  resolution: "Increase ParallelismPerKPU to reduce KPU count. Total parallelism = Parallelism × ParallelismPerKPU."

## Output Format

```yaml
root_cause: "parallelism_configuration — <specific_cause>"
evidence:
  - type: parallelism_detail
    content: "<config details>"
severity: MEDIUM
mitigation:
  immediate: "Fix parallelism config"
  long_term: "Document resource planning"
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
