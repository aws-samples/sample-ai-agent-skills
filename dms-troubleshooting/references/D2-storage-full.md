---
title: "D2 — Storage Full"
description: "Diagnose and resolve storage full issues on DMS replication instances"
status: active
severity: CRITICAL
triggers:
  - "storage full"
  - "no space left"
  - "FreeStorageSpace zero"
  - "task stopped storage"
owner: devops-agent
objective: "Restore storage capacity on the replication instance and prevent recurrence"
context: "DMS uses local storage for change caching, sorting, LOB data, and transaction logs. When storage fills, tasks stop. Large transactions, LOB columns, and high CDC volume consume storage rapidly. Storage can be increased but not decreased."
---

## Phase 1 — Triage

MUST:
- Check free storage: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name FreeStorageSpace --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 60 --statistics Average,Minimum`
- Check allocated storage: `aws dms describe-replication-instances --filters Name=replication-instance-id,Values=<instance-id> --query 'ReplicationInstances[*].{Storage:AllocatedStorage,Class:ReplicationInstanceClass}'`
- Check active tasks and their CDC status: `aws dms describe-replication-tasks --filters Name=replication-instance-arn,Values=<instance-arn> --query 'ReplicationTasks[*].{Id:ReplicationTaskIdentifier,Status:Status,Type:MigrationType}'`
- Check CDC latency (high latency = more cached changes): `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencyTarget --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Identify tasks with LOB columns consuming storage
- Check for long-running transactions on the source holding change cache
- Review storage consumption trend over time

MAY:
- Check if validation data is consuming storage
- Review task settings for change processing memory limits

## Phase 2 — Remediate

MUST:
- Increase allocated storage: `aws dms modify-replication-instance --replication-instance-arn <arn> --allocated-storage <new-size-gb> --apply-immediately`
- If task stopped, restart after increasing storage: `aws dms start-replication-task --replication-task-arn <task-arn> --start-replication-task-type resume-processing`

SHOULD:
- Switch LOB mode from full to limited/inline to reduce storage usage
- Reduce CDC lag to minimize cached changes
- Set up CloudWatch alarm on FreeStorageSpace

MAY:
- Distribute tasks across multiple instances to spread storage load
- Reduce the number of concurrent tasks on the instance

## Common Issues

- symptoms: "Task stops with storage full error"
  diagnosis: "Replication instance storage exhausted by cached changes or LOB data."
  resolution: "Increase allocated storage. Reduce CDC lag. Optimize LOB handling."

- symptoms: "Storage fills rapidly during full load"
  diagnosis: "Large LOB columns or many parallel table loads consuming storage."
  resolution: "Reduce MaxFullLoadSubTasks. Switch to limited LOB mode."

- symptoms: "Storage fills during CDC with large transactions"
  diagnosis: "Long-running source transactions cache all changes until commit."
  resolution: "Increase storage. Avoid large batch operations on source during CDC."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase allocated storage | GREEN | Storage increase — no downtime, cannot be decreased |
| Restart stopped task with resume processing | YELLOW | Resumes from checkpoint — verify data consistency |
| Switch LOB mode to limited/inline | YELLOW | May truncate LOB data — verify data requirements |
| Set up CloudWatch alarm on FreeStorageSpace | GREEN | Monitoring — non-destructive |
| Distribute tasks across multiple instances | YELLOW | Task restructuring — verify data consistency |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Storage exhaustion caused task failure and data sync gap
- Multiple tasks stopped due to shared instance storage

## Data Sensitivity

- **Classification: HIGH**
- Storage metrics reveal data replication volumes and caching patterns
- CDC latency data exposes replication health and data freshness
- LOB column details reveal data types and sizes in source database
- Task configuration reveals migration architecture and data flow

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest reducing allocated storage (not possible — storage only increases)
- **NEVER** suggest deleting replication instance storage to free space

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Increased allocated storage | Cannot be decreased — monitor usage going forward |
| Restarted task with resume processing | Stop task if resume causes data inconsistency |
| Changed LOB mode | Revert LOB mode and reload affected tables if data was truncated |

## Output Format

```yaml
root_cause: "storage_full — <specific_cause>"
evidence:
  - type: free_storage
    content: "<FreeStorageSpace metric>"
  - type: allocated_storage
    content: "<current allocated storage GB>"
  - type: active_tasks
    content: "<tasks and their CDC status>"
severity: CRITICAL
mitigation:
  immediate: "Increase allocated storage and resume tasks"
  long_term: "Set up storage monitoring alarms and right-size from the start"
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
