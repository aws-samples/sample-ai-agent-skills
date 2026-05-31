---
title: "G1 — ElastiCache Backup Failures"
description: "Diagnose Redis snapshot/backup failures and excessive backup duration"
status: active
severity: HIGH
triggers:
  - "backup failed"
  - "snapshot failed"
  - "backup timeout"
  - "backup slow"
  - "CreateSnapshot"
  - "BGSAVE"
owner: devops-agent
objective: "Resolve backup failures and ensure reliable snapshot creation"
context: "Redis backups use fork() to create a child process for background saving (BGSAVE). This temporarily requires up to 2x memory due to copy-on-write. Backups can fail if insufficient memory is available. Memcached does NOT support backups. ElastiCache stores snapshots in S3 (managed by AWS). Backup window and retention are configured per replication group. Manual snapshots do not expire automatically."
---

## Phase 1 — Triage

MUST:
- Check snapshot status: `aws elasticache describe-snapshots --replication-group-id <repl-group-id>`
- Check recent events for backup errors: `aws elasticache describe-events --source-type cache-cluster --duration 1440`
- Check memory usage during backup window: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name DatabaseMemoryUsagePercentage --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check SaveInProgress: `redis-cli -h <endpoint> -p 6379 INFO persistence`
- Verify engine is Redis (Memcached does not support backups): `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].Engine'`

SHOULD:
- Check backup configuration: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].{SnapshotWindow:SnapshotWindow,SnapshotRetention:SnapshotRetentionLimit}'`
- Check if backup is running on the primary or replica
- Check SwapUsage during backup: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name SwapUsage --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check reserved-memory-percent (needs headroom for fork)

MAY:
- Check snapshot quota: `aws elasticache describe-snapshots --query 'length(Snapshots)'`
- Review CloudTrail for CreateSnapshot API calls
- Check if the backup window overlaps with peak traffic

## Phase 2 — Remediate

MUST:
- Ensure sufficient free memory for fork() — at least 2x headroom during backup
- Schedule backup window during low-traffic periods
- Set reserved-memory-percent to at least 25% to accommodate backup overhead

SHOULD:
- Configure backups to run on a replica (reduces primary impact)
- Set appropriate snapshot retention limit
- Monitor SaveInProgress and backup duration

MAY:
- Create manual snapshots before major changes: `aws elasticache create-snapshot --replication-group-id <repl-group-id> --snapshot-name <name>`
- Export snapshots to S3 for long-term retention: `aws elasticache copy-snapshot --source-snapshot-name <name> --target-snapshot-name <name> --target-bucket <bucket>`
- Reduce dataset size to speed up backups

## Common Issues

- symptoms: "Backup fails with OOM during BGSAVE"
  diagnosis: "Insufficient memory for fork(). Copy-on-write requires up to 2x memory during heavy writes."
  resolution: "Scale up node type. Reduce memory usage. Schedule backups during low-write periods."

- symptoms: "Backup takes hours to complete"
  diagnosis: "Large dataset with high write volume causes extensive copy-on-write."
  resolution: "Schedule during low-traffic. Consider cluster mode to distribute data across smaller shards."

- symptoms: "Cannot create backup for Memcached cluster"
  diagnosis: "Memcached does not support backups or snapshots."
  resolution: "Memcached has no persistence. Use Redis if backups are required."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Ensure sufficient memory for fork() | GREEN | Capacity verification; no immediate change |
| Schedule backup during low-traffic | GREEN | Configuration change; reduces peak impact |
| Set reserved-memory-percent to 25%+ | YELLOW | Reduces available data memory; may trigger evictions |
| Configure replica-based backups | GREEN | Reduces primary impact; no data risk |
| Create manual snapshot | GREEN | Point-in-time backup; brief performance impact |
| Export snapshot to S3 | GREEN | Copy operation; no cluster impact |

## Escalation Conditions

- Backup failures on a production cluster with no recent successful backup
- OOM during BGSAVE causing backup to fail repeatedly
- Backup duration exceeding the backup window
- Snapshot quota reached preventing new backups
- Backup required before a critical maintenance operation

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-snapshots` | LOW | Snapshot metadata only |
| `describe-events` (backup events) | LOW | Operational events only |
| `INFO persistence` | LOW | Persistence statistics only |
| `get-metric-statistics` (memory during backup) | LOW | Operational metrics only |
| `describe-replication-groups` (backup config) | LOW | Backup configuration only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to make backups succeed
- NEVER suggest disabling AUTH to troubleshoot backup issues
- NEVER suggest disabling encryption in transit during backup operations
- NEVER suggest reducing node count during peak traffic
- NEVER delete all existing snapshots without confirming retention requirements

## Phase 3 — Rollback

If backup configuration changes cause issues:
1. If reserved-memory-percent was increased and causes evictions, revert to previous value
2. If backup window was changed, revert to previous schedule
3. If replica-based backups were configured and cause replica lag, revert to primary-based backups
4. Manual snapshots can be deleted if no longer needed: `aws elasticache delete-snapshot --snapshot-name <name>`
5. Monitor backup success and memory usage after rollback

## Output Format

```yaml
root_cause: "backup_failure — <specific_cause>"
evidence:
  - type: snapshot_status
    content: "<snapshot details>"
  - type: memory_usage
    content: "<memory during backup window>"
  - type: events
    content: "<backup-related events>"
severity: HIGH
mitigation:
  immediate: "Ensure sufficient memory and retry backup"
  long_term: "Schedule backups during low-traffic, configure replica-based backups"
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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest disabling AUTH"
  - "NEVER suggest public subnet placement"
