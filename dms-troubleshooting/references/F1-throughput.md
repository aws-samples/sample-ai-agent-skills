---
title: "F1 — Throughput"
description: "Diagnose and improve DMS migration throughput"
status: active
severity: MEDIUM
triggers:
  - "slow migration"
  - "low throughput"
  - "migration taking too long"
  - "full load slow"
owner: devops-agent
objective: "Identify throughput bottlenecks and optimize DMS migration speed"
context: "Migration throughput depends on replication instance size, source read speed, target write speed, network bandwidth, LOB handling, parallel load settings, and task configuration. Full load and CDC have different optimization strategies."
---

## Phase 1 — Triage

MUST:
- Check full load throughput: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name FullLoadThroughputRowsSource --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check instance CPU: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CPUUtilization --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check network throughput: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name NetworkTransmitThroughput --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check table statistics for load progress: `aws dms describe-table-statistics --replication-task-arn <task-arn>`

SHOULD:
- Review task settings for ParallelLoadThreads and MaxFullLoadSubTasks
- Check if LOB handling is slowing down the migration
- Verify source and target are not the bottleneck

MAY:
- Compare throughput across different times of day
- Check if source database is throttling reads

## Phase 2 — Remediate

MUST:
- Scale up replication instance if CPU > 80%
- Increase ParallelLoadThreads for full load (default is 8)
- Increase MaxFullLoadSubTasks for concurrent table loads (default is 8)

SHOULD:
- Switch LOB mode to limited or inline for better throughput
- Enable BatchApplyEnabled for CDC throughput
- Remove unnecessary target indexes during full load (rebuild after)

MAY:
- Use partition-based parallel load for large tables
- Schedule migration during off-peak hours for better source/target performance

## Common Issues

- symptoms: "Full load processing one table at a time"
  diagnosis: "MaxFullLoadSubTasks set to 1 or very low."
  resolution: "Increase MaxFullLoadSubTasks to 8-16 depending on instance size."

- symptoms: "Throughput drops when LOB tables are loading"
  diagnosis: "Full LOB mode reading each LOB individually."
  resolution: "Switch to limited or inline LOB mode. Set appropriate MaxLobSize."

- symptoms: "Network throughput at instance limit"
  diagnosis: "Instance class network bandwidth is the bottleneck."
  resolution: "Scale up to a larger instance class with higher network bandwidth."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Scale up replication instance | YELLOW | Instance modification — brief downtime |
| Increase ParallelLoadThreads | GREEN | Task setting — applied on restart |
| Switch LOB mode to limited/inline | YELLOW | May truncate LOB data — verify requirements |
| Remove target indexes during full load | YELLOW | Schema change on target — rebuild after |
| Use partition-based parallel load | GREEN | Task configuration — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Slow throughput delays migration cutover timeline
- Target index removal affects applications using the target database

## Data Sensitivity

- **Classification: HIGH**
- Throughput metrics reveal data volumes and migration progress
- Instance utilization exposes infrastructure sizing for data replication
- LOB handling details reveal data types and sizes in source database
- Network throughput reveals data transfer rates between environments

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest removing target indexes on a database serving live traffic
- **NEVER** suggest switching to limited LOB mode without confirming acceptable truncation

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Scaled up replication instance | Scale down after migration completes |
| Changed LOB mode | Revert LOB mode and reload affected tables if data was truncated |
| Removed target indexes | Rebuild target indexes after full load completes |
| Changed parallel load settings | Revert to previous settings if they cause issues |

## Output Format

```yaml
root_cause: "throughput — <specific_cause>"
evidence:
  - type: throughput_metrics
    content: "<rows/sec and bytes/sec metrics>"
  - type: instance_utilization
    content: "<CPU, memory, network metrics>"
severity: MEDIUM
mitigation:
  immediate: "Address the specific throughput bottleneck"
  long_term: "Tune task settings and instance sizing for optimal throughput"
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
