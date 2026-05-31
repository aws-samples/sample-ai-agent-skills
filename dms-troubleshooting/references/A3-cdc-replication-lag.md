---
title: "A3 — CDC Replication Lag"
description: "Diagnose and reduce CDC replication lag in DMS tasks"
status: active
severity: HIGH
triggers:
  - "CDC lag"
  - "replication lag"
  - "CDC latency high"
  - "changes not applying"
  - "target behind source"
owner: devops-agent
objective: "Identify the cause of CDC replication lag and reduce it to acceptable levels"
context: "CDC lag occurs when the target falls behind the source. Causes include undersized replication instance, high source change rate, slow target apply, LOB processing, network latency, or large transactions. CDCLatencySource measures time to read from source; CDCLatencyTarget measures time to apply to target."
---

## Phase 1 — Triage

MUST:
- Check CDC latency source: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencySource --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check CDC latency target: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencyTarget --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check replication instance CPU: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CPUUtilization --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check CDC throughput: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCThroughputRowsSource --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check table statistics for tables with high update rates: `aws dms describe-table-statistics --replication-task-arn <task-arn> --query 'TableStatistics[*].{Table:TableName,Updates:Updates,Inserts:Inserts,Deletes:Deletes,DDLs:Ddls}'`
- Check free memory: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name FreeableMemory --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify BatchApplyEnabled in task settings

MAY:
- Check source database for long-running transactions holding log positions
- Review network throughput between replication instance and endpoints

## Phase 2 — Remediate

MUST:
- If CDCLatencySource is high: check source connectivity and supplemental logging config
- If CDCLatencyTarget is high: enable BatchApplyEnabled, check target performance
- If CPU > 80%: scale up replication instance class

SHOULD:
- Enable BatchApplyEnabled in task settings for better CDC apply performance
- Increase ParallelApplyThreads for multi-threaded apply
- Optimize target indexes — too many indexes slow down applies

MAY:
- Consider splitting high-volume tables into separate tasks
- Reduce source change rate during initial catch-up

## Common Issues

- symptoms: "CDCLatencySource increasing steadily"
  diagnosis: "Source read throughput cannot keep up with change rate."
  resolution: "Scale up instance. Check source supplemental logging. Verify network bandwidth."

- symptoms: "CDCLatencyTarget high but CDCLatencySource low"
  diagnosis: "Target apply is the bottleneck."
  resolution: "Enable BatchApplyEnabled. Remove unnecessary target indexes during migration. Scale target."

- symptoms: "Lag spikes during business hours"
  diagnosis: "Source change rate peaks during business hours exceed apply capacity."
  resolution: "Scale up instance. Enable batch apply. Consider scheduling heavy operations off-peak."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Scale up replication instance class | YELLOW | Instance modification — brief downtime |
| Enable BatchApplyEnabled | YELLOW | Changes apply behavior — may affect ordering |
| Increase ParallelApplyThreads | GREEN | Task setting — applied on restart |
| Optimize target indexes | YELLOW | Schema change on target — may affect target applications |
| Split high-volume tables into separate tasks | YELLOW | Task restructuring — verify data consistency |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- CDC lag exceeds acceptable threshold for production cutover
- Target database serves live traffic and index changes affect performance

## Data Sensitivity

- **Classification: HIGH**
- CDC latency metrics reveal data freshness and replication health
- Source/target throughput metrics expose data change rates and volumes
- Table-level statistics reveal which tables have highest change rates
- Instance utilization reveals infrastructure sizing for data replication

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest dropping target indexes on a database serving live traffic without coordination
- **NEVER** suggest reducing source change rate by modifying production application behavior

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Scaled up replication instance | Scale down after lag is resolved |
| Enabled BatchApplyEnabled | Disable batch apply if it causes ordering issues |
| Removed target indexes | Rebuild target indexes after migration completes |
| Split tables into separate tasks | Consolidate tasks if split causes management overhead |

## Output Format

```yaml
root_cause: "cdc_lag — <specific_cause>"
evidence:
  - type: cdc_latency_source
    content: "<source latency metrics>"
  - type: cdc_latency_target
    content: "<target latency metrics>"
  - type: instance_utilization
    content: "<CPU, memory metrics>"
severity: HIGH
mitigation:
  immediate: "Address the specific lag bottleneck"
  long_term: "Right-size instance, enable batch apply, optimize target schema"
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
