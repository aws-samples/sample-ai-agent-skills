---
title: "D1 — Replication Instance Sizing"
description: "Diagnose replication instance sizing issues affecting migration performance"
status: active
severity: MEDIUM
triggers:
  - "instance too small"
  - "CPU high on replication instance"
  - "memory pressure"
  - "slow migration"
  - "instance sizing"
owner: devops-agent
objective: "Right-size the DMS replication instance for the migration workload"
context: "Undersized replication instances cause slow migrations, task failures, and CDC lag. T-class instances have burstable CPU and are unsuitable for sustained workloads. R-class instances provide consistent performance. Instance class determines CPU, memory, and network throughput limits."
---

## Phase 1 — Triage

MUST:
- Check current instance class: `aws dms describe-replication-instances --filters Name=replication-instance-id,Values=<instance-id> --query 'ReplicationInstances[*].{Class:ReplicationInstanceClass,MultiAZ:MultiAZ,Storage:AllocatedStorage,Status:ReplicationInstanceStatus}'`
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CPUUtilization --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Check freeable memory: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name FreeableMemory --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Minimum`
- Check swap usage: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name SwapUsage --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Check network throughput: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name NetworkTransmitThroughput --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Count active tasks on the instance: `aws dms describe-replication-tasks --filters Name=replication-instance-arn,Values=<instance-arn> --query 'ReplicationTasks[?Status==`running`] | length(@)'`

MAY:
- Check if T-class CPU credits are depleted (for burstable instances)
- Compare current utilization against instance class limits

## Phase 2 — Remediate

MUST:
- Scale up if CPU > 80% sustained or memory < 20% of total
- Switch from T-class to R-class for sustained workloads
- Modify instance: `aws dms modify-replication-instance --replication-instance-arn <arn> --replication-instance-class dms.r5.xlarge --apply-immediately`

SHOULD:
- Distribute tasks across multiple instances if one is overloaded
- Use R5 or R6i class for production migrations
- Monitor metrics after scaling to verify improvement

MAY:
- Consider C5 class for CPU-intensive transformations
- Right-size down after migration completes to save costs

## Common Issues

- symptoms: "Migration slow but no errors"
  diagnosis: "Instance undersized — CPU or memory constrained."
  resolution: "Scale up to a larger instance class. Use R-class for sustained workloads."

- symptoms: "CPU spikes to 100% periodically"
  diagnosis: "T-class instance exhausting CPU burst credits."
  resolution: "Switch to R-class instance for consistent CPU performance."

- symptoms: "Multiple tasks competing for resources"
  diagnosis: "Too many tasks on a single replication instance."
  resolution: "Distribute tasks across multiple replication instances."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Scale up replication instance class | YELLOW | Instance modification — brief downtime during apply |
| Switch from T-class to R-class | YELLOW | Instance modification — brief downtime |
| Distribute tasks across multiple instances | YELLOW | Task restructuring — verify data consistency |
| Right-size down after migration | GREEN | Cost optimization — non-destructive |
| Monitor metrics after scaling | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Instance modification causes downtime for all tasks on the instance
- Multiple tasks share the instance and scaling affects all of them

## Data Sensitivity

- **Classification: HIGH**
- Instance metrics reveal replication workload characteristics
- CPU/memory utilization exposes data processing intensity
- Network throughput reveals data transfer volumes between source and target
- Task count and distribution reveal migration architecture

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest scaling down an instance while tasks are actively replicating
- **NEVER** suggest using T-class instances for production CDC workloads

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Scaled up replication instance | Scale down after migration completes to reduce cost |
| Switched to R-class instance | Revert to previous class if cost is a concern (after migration) |
| Distributed tasks across instances | Consolidate tasks if distribution causes management overhead |

## Output Format

```yaml
root_cause: "instance_sizing — <specific_cause>"
evidence:
  - type: instance_class
    content: "<current instance class and specs>"
  - type: cpu_utilization
    content: "<CPU metrics>"
  - type: memory
    content: "<memory and swap metrics>"
severity: MEDIUM
mitigation:
  immediate: "Scale up the replication instance"
  long_term: "Size instances based on workload requirements from the start"
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
  - "NEVER suggest disabling SSL for replication endpoints"
  - "NEVER suggest public replication instances"
  - "NEVER suggest deleting replication tasks without data verification"
