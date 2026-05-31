---
title: "C1 — ElastiCache Replication Lag"
description: "Diagnose replication lag between Redis primary and replica nodes"
status: active
severity: HIGH
triggers:
  - "replication lag"
  - "replica behind"
  - "sync delay"
  - "ReplicationLag"
  - "stale reads"
owner: devops-agent
objective: "Identify the cause of replication lag and restore near-zero lag between primary and replicas"
context: "Redis replication is asynchronous — replicas may lag behind the primary. ElastiCache reports ReplicationLag in CloudWatch (seconds). Normal lag is under 1 second. High lag causes stale reads from replicas. Causes include high write volume on primary, network issues, replica CPU saturation, large key operations, or background save on the replica. During initial sync (full resync), lag can be significant."
---

## Phase 1 — Triage

MUST:
- Check replication lag metric: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name ReplicationLag --dimensions Name=CacheClusterId,Value=<replica-cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum,Average`
- Check replication info: `redis-cli -h <primary-endpoint> -p 6379 INFO replication`
- Check replica status: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].NodeGroups[*].NodeGroupMembers'`
- Check primary write volume: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name SetTypeCmds --dimensions Name=CacheClusterId,Value=<primary-cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check replica EngineCPU: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name EngineCPUUtilization --dimensions Name=CacheClusterId,Value=<replica-cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`

SHOULD:
- Check replication buffer: `redis-cli -h <primary-endpoint> -p 6379 INFO replication | grep repl_backlog`
- Check if full resync is occurring: `redis-cli -h <primary-endpoint> -p 6379 INFO stats | grep sync`
- Check network throughput: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name NetworkBytesOut --dimensions Name=CacheClusterId,Value=<primary-cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check for recent events: `aws elasticache describe-events --source-type cache-cluster --duration 60`

MAY:
- Check if replica is in a different AZ (cross-AZ replication adds latency)
- Verify replication backlog size is sufficient: `redis-cli -h <primary-endpoint> -p 6379 CONFIG GET repl-backlog-size`
- Check if Lua scripts or large transactions are causing replication bursts

## Phase 2 — Remediate

MUST:
- Scale up replica node type if replica EngineCPU is saturated
- Reduce write volume on primary if possible (batch writes, reduce write frequency)
- Ensure replication backlog size is large enough to avoid full resyncs

SHOULD:
- Place replicas in the same AZ as the primary for lowest latency
- Increase repl-backlog-size if partial resyncs are failing (causing full resyncs)
- Monitor ReplicationLag with CloudWatch alarms (alert at >1 second)

MAY:
- Consider cluster mode enabled to distribute writes across shards
- Reduce the number of replicas if replication fan-out is causing primary CPU pressure
- Optimize large write operations that generate large replication streams

## Common Issues

- symptoms: "ReplicationLag steadily increasing over time"
  diagnosis: "Write volume exceeds replica's ability to apply changes. Replica CPU is saturated."
  resolution: "Scale up replica node type. Reduce write volume. Add shards to distribute writes."

- symptoms: "ReplicationLag spikes periodically then recovers"
  diagnosis: "Background save on primary or replica causes temporary lag spike."
  resolution: "Schedule backups during low-traffic periods. Ensure sufficient memory for fork()."

- symptoms: "Full resync occurring repeatedly"
  diagnosis: "Replication backlog is too small. When replica disconnects briefly, it cannot do partial resync."
  resolution: "Increase repl-backlog-size to accommodate the write volume during disconnection windows."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Scale up replica node type | YELLOW | Requires node replacement; brief unavailability of that replica |
| Increase repl-backlog-size | GREEN | Parameter change; prevents unnecessary full resyncs |
| Place replicas in same AZ | YELLOW | Requires replica recreation in different AZ |
| Reduce write volume | GREEN | Application-level optimization |
| Add shards (cluster mode) | YELLOW | Resharding operation; MOVED/ASK redirections during migration |

## Escalation Conditions

- ReplicationLag exceeding 5 seconds on a production cluster
- Full resync occurring repeatedly (repl-backlog exhausted)
- Replication lag causing stale reads impacting customer experience
- Primary CPU saturated by replication fan-out to multiple replicas
- Replication lag increasing steadily with no sign of recovery

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `INFO replication` | LOW | Replication statistics only |
| `get-metric-statistics` (ReplicationLag) | LOW | Operational metrics only |
| `describe-replication-groups` | MEDIUM | Exposes cluster architecture and node roles |
| `describe-events` | LOW | Operational events only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to reduce replication lag
- NEVER suggest disabling AUTH to reduce replication overhead
- NEVER suggest disabling encryption in transit to reduce replication latency
- NEVER suggest reducing node count during peak traffic or high replication lag
- NEVER force a failover to fix replication lag (may cause data loss)

## Phase 3 — Rollback

If replication lag remediation causes issues:
1. If replica node type was scaled up and causes issues, scale back to previous type
2. If repl-backlog-size was increased, revert in parameter group (note: may require reboot)
3. If write volume was reduced and impacts application functionality, revert to previous write patterns
4. If shards were added, resharding can be reversed but is a lengthy operation
5. Monitor ReplicationLag to verify it stabilizes after rollback

## Output Format

```yaml
root_cause: "replication_lag — <specific_cause>"
evidence:
  - type: replication_lag
    content: "<ReplicationLag metric values>"
  - type: replication_info
    content: "<INFO replication output>"
  - type: write_volume
    content: "<SetTypeCmds metrics>"
severity: HIGH
mitigation:
  immediate: "Scale up replicas or reduce write volume"
  long_term: "Right-size replication backlog, implement monitoring, consider sharding"
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
