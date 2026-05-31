---
title: "B1 — ElastiCache High CPU Utilization"
description: "Diagnose high CPU and EngineCPU utilization on ElastiCache Redis or Memcached nodes"
status: active
severity: HIGH
triggers:
  - "high CPU"
  - "CPU utilization"
  - "EngineCPU"
  - "slow response"
  - "CPU spike"
owner: devops-agent
objective: "Identify the cause of high CPU utilization and restore normal performance"
context: "ElastiCache exposes two CPU metrics: CPUUtilization (total host CPU including OS, management) and EngineCPUUtilization (Redis/Memcached engine only). For Redis, EngineCPU is the critical metric because Redis is single-threaded for command processing. EngineCPU above 65% indicates the node is becoming a bottleneck. For Memcached, CPUUtilization is more relevant as Memcached is multi-threaded."
---

## Phase 1 — Triage

MUST:
- Check EngineCPU (Redis) or CPUUtilization (Memcached): `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name EngineCPUUtilization --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Average,Maximum`
- Check CPUUtilization: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name CPUUtilization --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Average,Maximum`
- Check slow log for expensive commands: `redis-cli -h <endpoint> -p 6379 SLOWLOG GET 50`
- Check command stats: `redis-cli -h <endpoint> -p 6379 INFO commandstats`
- Check operations per second: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name CacheHits --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`

SHOULD:
- Identify hot keys: `redis-cli -h <endpoint> -p 6379 --hotkeys` (requires maxmemory-policy with LFU)
- Check if background save is running: `redis-cli -h <endpoint> -p 6379 INFO persistence`
- Check Lua script execution: `redis-cli -h <endpoint> -p 6379 INFO stats | grep lua`
- Review if TLS is enabled (adds ~25% CPU overhead)

MAY:
- Check for large key operations: `redis-cli -h <endpoint> -p 6379 --bigkeys`
- Monitor real-time command throughput: `redis-cli -h <endpoint> -p 6379 INFO stats`
- Check if cluster mode resharding is in progress

## Phase 2 — Remediate

MUST:
- Optimize or eliminate slow commands (KEYS → SCAN, large HGETALL → HSCAN, large SMEMBERS → SSCAN)
- Scale up to a larger node type if EngineCPU consistently exceeds 65%
- Offload read traffic to read replicas using the reader endpoint

SHOULD:
- Use cluster mode enabled to distribute writes across multiple shards
- Implement client-side caching to reduce command volume
- Optimize Lua scripts to reduce execution time
- Schedule backups during off-peak hours to avoid fork() CPU impact

MAY:
- Consider ElastiCache Serverless for automatic scaling
- Use pipeline/batch operations to reduce round trips
- Evaluate if TLS overhead is acceptable or if encryption at rest alone suffices

## Common Issues

- symptoms: "EngineCPU at 90%+ with high SLOWLOG entries"
  diagnosis: "O(N) commands like KEYS, SORT, or large collection operations consuming CPU."
  resolution: "Replace with SCAN-family commands. Break large collections into smaller ones."

- symptoms: "CPU spikes during backup window"
  diagnosis: "Redis fork() for background save consumes CPU temporarily."
  resolution: "Schedule backups during low-traffic periods. Ensure sufficient memory headroom."

- symptoms: "Steady high EngineCPU with normal command patterns"
  diagnosis: "Node type is undersized for the workload volume."
  resolution: "Scale up to a larger node type or add shards (cluster mode enabled)."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Optimize slow commands (KEYS → SCAN) | GREEN | Application-level improvement; immediate CPU relief |
| Scale up node type | YELLOW | Requires failover for Redis with replication; brief downtime |
| Offload reads to replicas | GREEN | Application-level routing; no infrastructure risk |
| Use cluster mode enabled | YELLOW | Requires new cluster creation and data migration |
| Schedule backups during off-peak | GREEN | Configuration change; reduces peak CPU impact |
| Implement client-side caching | GREEN | Application-level optimization |

## Escalation Conditions

- EngineCPU consistently above 80% on a production cluster
- Slow commands blocking the event loop for >5 seconds
- CPU spike causing cascading timeouts across dependent services
- Node type already at maximum available size
- CPU issue caused by background save (BGSAVE) during peak hours

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `SLOWLOG GET` | MEDIUM | Exposes command patterns and key names |
| `INFO commandstats` | LOW | Command frequency statistics only |
| `--hotkeys` | MEDIUM | Exposes frequently accessed key names |
| `--bigkeys` | MEDIUM | Exposes key names and sizes |
| `get-metric-statistics` (EngineCPU) | LOW | Operational metrics only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to reduce CPU usage
- NEVER suggest disabling AUTH to reduce CPU overhead
- NEVER suggest disabling encryption in transit without understanding compliance requirements
- NEVER suggest reducing node count during peak traffic
- NEVER use KEYS command in production (O(N) scan causes CPU spikes)

## Phase 3 — Rollback

If CPU remediation changes cause issues:
1. If node type was scaled up and causes issues, scale back to previous type (note: causes another failover)
2. If read traffic was routed to replicas and causes stale reads, revert to primary-only reads
3. If backup schedule was changed, revert to previous backup window
4. Revert any application-level command optimizations if they cause functional issues
5. Monitor EngineCPU to verify it returns to acceptable levels after rollback

## Output Format

```yaml
root_cause: "high_cpu — <specific_cause>"
evidence:
  - type: engine_cpu
    content: "<EngineCPU metrics>"
  - type: slow_log
    content: "<top slow commands>"
  - type: command_stats
    content: "<command frequency breakdown>"
severity: HIGH
mitigation:
  immediate: "Optimize slow commands or scale up node type"
  long_term: "Implement read replicas, cluster mode, and command optimization"
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
