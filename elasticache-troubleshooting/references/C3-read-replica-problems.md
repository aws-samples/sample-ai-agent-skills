---
title: "C3 — ElastiCache Read Replica Problems"
description: "Diagnose issues with Redis read replicas including sync failures, stale data, and replica unavailability"
status: active
severity: HIGH
triggers:
  - "read replica"
  - "replica sync"
  - "stale data"
  - "replica unavailable"
  - "reader endpoint"
  - "replica not syncing"
owner: devops-agent
objective: "Resolve read replica issues and ensure replicas serve consistent, up-to-date data"
context: "Redis read replicas receive asynchronous replication from the primary. The reader endpoint load-balances across all replicas. Replicas can fall behind (replication lag), fail to sync (full resync required), or become unavailable. Each shard supports up to 5 replicas. Replicas serve read traffic and provide failover targets."
---

## Phase 1 — Triage

MUST:
- Check replica status: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].NodeGroups[*].NodeGroupMembers'`
- Check replication info from primary: `redis-cli -h <primary-endpoint> -p 6379 INFO replication`
- Check replica replication status: `redis-cli -h <replica-endpoint> -p 6379 INFO replication`
- Check ReplicationLag: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name ReplicationLag --dimensions Name=CacheClusterId,Value=<replica-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check recent events for sync issues: `aws elasticache describe-events --source-identifier <replica-id> --source-type cache-cluster --duration 1440`

SHOULD:
- Check replica memory usage: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name DatabaseMemoryUsagePercentage --dimensions Name=CacheClusterId,Value=<replica-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check replica EngineCPU: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name EngineCPUUtilization --dimensions Name=CacheClusterId,Value=<replica-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Verify reader endpoint resolves to healthy replicas: `nslookup <reader-endpoint>`
- Check if replica is undergoing maintenance

MAY:
- Check replication backlog on primary: `redis-cli -h <primary-endpoint> -p 6379 INFO replication | grep repl_backlog`
- Verify replica node type matches primary (recommended for consistent performance)
- Check if slow commands on replica are causing issues

## Phase 2 — Remediate

MUST:
- If replica is stuck in sync, check for memory issues (full resync requires 2x memory)
- Ensure replica node type has sufficient memory and CPU
- If replica is permanently broken, remove and re-add it

SHOULD:
- Use the same node type for replicas as the primary
- Increase repl-backlog-size to prevent unnecessary full resyncs
- Monitor ReplicationLag with CloudWatch alarms

MAY:
- Add additional replicas for read scaling
- Place replicas in different AZs for availability
- Consider cluster mode enabled for horizontal read scaling

## Common Issues

- symptoms: "Replica shows master_link_status:down"
  diagnosis: "Replica lost connection to primary and cannot resync."
  resolution: "Check network connectivity. If repl-backlog is exhausted, a full resync will occur automatically."

- symptoms: "Reader endpoint returns stale data"
  diagnosis: "Replication lag is high. Reads from replicas return older data."
  resolution: "Reduce replication lag (see C1). For strong consistency, read from the primary endpoint."

- symptoms: "Replica stuck in 'syncing' state"
  diagnosis: "Full resync is in progress. Large datasets take time. Insufficient memory can cause repeated failures."
  resolution: "Wait for sync to complete. Ensure replica has sufficient memory (2x dataset for fork)."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Ensure replica node type matches primary | YELLOW | Requires node replacement if different |
| Increase repl-backlog-size | GREEN | Parameter change; prevents full resyncs |
| Monitor ReplicationLag with alarms | GREEN | Monitoring only; no operational impact |
| Remove and re-add broken replica | YELLOW | Causes full resync; temporary loss of that replica |
| Add additional replicas | GREEN | Adds read capacity; no impact on existing nodes |

## Escalation Conditions

- Replica stuck in 'syncing' state for more than 1 hour
- master_link_status:down on a production replica
- Reader endpoint returning stale data impacting customer experience
- Multiple replicas failing simultaneously
- Replica sync failures causing primary memory pressure

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `INFO replication` (primary and replica) | LOW | Replication statistics only |
| `describe-replication-groups` (node members) | MEDIUM | Exposes cluster architecture |
| `get-metric-statistics` (ReplicationLag) | LOW | Operational metrics only |
| `nslookup` (reader endpoint) | LOW | DNS resolution only |
| `describe-events` | LOW | Operational events only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix replica sync issues
- NEVER suggest disabling AUTH on replicas
- NEVER suggest disabling encryption in transit on replicas
- NEVER suggest reducing node count during peak traffic
- NEVER force a failover to a lagging replica (causes data loss)

## Phase 3 — Rollback

If replica remediation causes issues:
1. If a replica was removed and re-added, wait for full sync to complete before routing read traffic
2. If replica node type was changed, revert to previous type if the new type causes issues
3. If repl-backlog-size was changed, revert in parameter group
4. If additional replicas were added and cause primary CPU pressure, remove the extra replicas
5. Monitor ReplicationLag and replica status after rollback

## Output Format

```yaml
root_cause: "read_replica — <specific_cause>"
evidence:
  - type: replica_status
    content: "<replica node status and role>"
  - type: replication_info
    content: "<INFO replication from primary and replica>"
  - type: replication_lag
    content: "<ReplicationLag metric>"
severity: HIGH
mitigation:
  immediate: "Fix replica sync or replace broken replica"
  long_term: "Implement replication monitoring and right-size replica nodes"
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
