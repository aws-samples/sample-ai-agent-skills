---
title: "A2 — Task Stuck"
description: "Diagnose why a DMS replication task is stuck or not progressing"
status: active
severity: HIGH
triggers:
  - "task stuck"
  - "task not progressing"
  - "task running but no data"
  - "full load stuck"
  - "CDC not applying"
owner: devops-agent
objective: "Identify why a DMS task appears stuck and restore progress"
context: "Tasks can appear stuck due to large table loads, LOB processing, resource exhaustion on the replication instance, source throttling, target bottlenecks, or lock contention. The task may show 'running' status but table statistics show no progress."
---

## Phase 1 — Triage

MUST:
- Check task status and progress: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].{Status:Status,Progress:ReplicationTaskStats}'`
- Check table statistics for progress: `aws dms describe-table-statistics --replication-task-arn <task-arn> --query 'TableStatistics[*].{Schema:SchemaName,Table:TableName,State:TableState,Inserts:Inserts,Updates:Updates,Deletes:Deletes,FullLoadRows:FullLoadRows}'`
- Check replication instance CPU and memory: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CPUUtilization --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Check free storage: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name FreeStorageSpace --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 60 --statistics Average`

SHOULD:
- Check CDC latency metrics: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencySource --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Look for tables in LOADING state that haven't progressed
- Check if LOB columns are causing slow processing

MAY:
- Review task settings for ParallelLoadThreads and MaxFullLoadSubTasks
- Check source database for long-running transactions or locks

## Phase 2 — Remediate

MUST:
- If resource-constrained, scale up the replication instance class
- If storage full, increase allocated storage
- If a single large table is blocking, consider table-level parallelism

SHOULD:
- Increase ParallelLoadThreads for full load: modify task settings
- Enable BatchApplyEnabled for CDC to improve apply throughput
- Switch LOB mode from full to limited or inline if LOBs are causing delays

MAY:
- Split large tables using partition-based parallel load
- Stop and restart the task if it appears genuinely hung

## Common Issues

- symptoms: "Task running but FullLoadRows not increasing"
  diagnosis: "Large table with LOB columns being processed in full LOB mode."
  resolution: "Switch to limited or inline LOB mode. Check LOB column sizes."

- symptoms: "CDC latency increasing but task shows running"
  diagnosis: "Target apply cannot keep up with source changes."
  resolution: "Enable BatchApplyEnabled. Scale up instance. Check target for bottlenecks."

- symptoms: "Task stuck at 0% for a long time"
  diagnosis: "Initial table inventory or assessment phase taking long."
  resolution: "Wait for inventory to complete. Check CloudWatch Logs for progress details."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Scale up replication instance class | YELLOW | Instance modification — brief downtime |
| Increase allocated storage | GREEN | Storage increase — no downtime |
| Increase ParallelLoadThreads | GREEN | Task setting change — applied on restart |
| Switch LOB mode to limited/inline | YELLOW | May truncate LOB data — verify data requirements |
| Stop and restart stuck task | YELLOW | Interrupts processing — verify checkpoint state |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Task has been stuck for extended period causing growing data sync gap
- LOB mode change may cause data truncation on critical columns

## Data Sensitivity

- **Classification: HIGH**
- Table statistics reveal database schema, row counts, and data volumes
- CDC latency metrics expose replication lag and data freshness
- LOB column details reveal data types and sizes in source database
- Instance metrics expose resource utilization patterns

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest switching to limited LOB mode without confirming acceptable data truncation
- **NEVER** suggest killing source database transactions to unblock DMS

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Scaled up replication instance | Scale down after issue is resolved |
| Increased allocated storage | Storage cannot be decreased — monitor usage |
| Changed LOB mode | Revert LOB mode and reload affected tables if data was truncated |
| Stopped and restarted task | Resume processing from checkpoint if restart caused issues |

## Output Format

```yaml
root_cause: "task_stuck — <specific_cause>"
evidence:
  - type: task_progress
    content: "<task statistics and progress>"
  - type: table_statistics
    content: "<per-table load status>"
  - type: instance_metrics
    content: "<CPU, memory, storage metrics>"
severity: HIGH
mitigation:
  immediate: "Address the specific bottleneck causing the stall"
  long_term: "Tune task settings and instance sizing for the workload"
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
