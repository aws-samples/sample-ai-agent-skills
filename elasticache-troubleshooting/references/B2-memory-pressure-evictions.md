---
title: "B2 — ElastiCache Memory Pressure and Evictions"
description: "Diagnose memory pressure, high eviction rates, and OOM conditions on ElastiCache nodes"
status: active
severity: HIGH
triggers:
  - "evictions"
  - "memory pressure"
  - "OOM"
  - "out of memory"
  - "memory full"
  - "DatabaseMemoryUsagePercentage"
owner: devops-agent
objective: "Identify the cause of memory pressure and reduce eviction rates to acceptable levels"
context: "ElastiCache nodes have a fixed amount of memory determined by the node type. When used_memory approaches maxmemory, Redis begins evicting keys based on the eviction policy. High eviction rates degrade cache hit ratio. OOM errors occur when the eviction policy is noeviction or volatile-* with no TTL keys. maxmemory-reserved must be accounted for — it reduces available data memory."
---

## Phase 1 — Triage

MUST:
- Check memory usage: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name DatabaseMemoryUsagePercentage --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Check eviction rate: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name Evictions --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check memory details: `redis-cli -h <endpoint> -p 6379 INFO memory`
- Check eviction policy: `aws elasticache describe-cache-parameters --cache-parameter-group-name <param-group> --query "Parameters[?ParameterName=='maxmemory-policy']"`
- Check maxmemory-reserved: `aws elasticache describe-cache-parameters --cache-parameter-group-name <param-group> --query "Parameters[?ParameterName=='reserved-memory-percent']"`

SHOULD:
- Identify large keys: `redis-cli -h <endpoint> -p 6379 --bigkeys`
- Check key count and TTL distribution: `redis-cli -h <endpoint> -p 6379 INFO keyspace`
- Check memory fragmentation ratio: `redis-cli -h <endpoint> -p 6379 INFO memory | grep mem_fragmentation_ratio`
- Check BytesUsedForCache metric: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name BytesUsedForCache --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

MAY:
- Analyze memory usage by data type: `redis-cli -h <endpoint> -p 6379 MEMORY USAGE <key>`
- Check for expired keys not yet reclaimed: `redis-cli -h <endpoint> -p 6379 INFO stats | grep expired`
- Review pub/sub output buffer memory usage

## Phase 2 — Remediate

MUST:
- Set appropriate eviction policy: allkeys-lru for cache-only, volatile-lru for mixed workloads
- Ensure maxmemory-reserved is at least 25% of maxmemory
- Scale up node type or add shards if memory is consistently above 80%

SHOULD:
- Set TTL on all cache keys to enable volatile-* eviction policies
- Optimize data structures (use hashes for small objects, compress large values)
- Remove unnecessary keys or reduce key sizes

MAY:
- Enable active defragmentation (Redis 4.0+): `activedefrag yes` in parameter group
- Consider cluster mode enabled to distribute data across shards
- Implement client-side TTL management

## Common Issues

- symptoms: "OOM errors but eviction policy is volatile-lru"
  diagnosis: "No keys have TTL set. volatile-* policies only evict keys with expiration."
  resolution: "Set TTL on cache keys or switch to allkeys-lru eviction policy."

- symptoms: "High evictions with DatabaseMemoryUsagePercentage at 100%"
  diagnosis: "Dataset exceeds available memory. Evictions are constant."
  resolution: "Scale up node type, add shards, or reduce dataset size. Set appropriate TTLs."

- symptoms: "OOM during replication sync"
  diagnosis: "maxmemory-reserved is too low. Replication buffer exceeds reserved memory."
  resolution: "Increase reserved-memory-percent to 25% or higher in the parameter group."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Set eviction policy (allkeys-lru) | YELLOW | Changes eviction behavior; may evict different keys than before |
| Set maxmemory-reserved to 25% | YELLOW | Reduces available data memory; may trigger evictions |
| Scale up node type | YELLOW | Requires failover for Redis with replication; brief downtime |
| Set TTL on cache keys | GREEN | Application-level change; enables controlled expiration |
| Optimize data structures | GREEN | Application-level improvement; reduces memory usage |
| Enable active defragmentation | GREEN | Background process; minimal performance impact |

## Escalation Conditions

- OOM errors causing write failures in production
- Eviction rate exceeding 1000/second causing cache hit ratio below 50%
- DatabaseMemoryUsagePercentage at 100% with noeviction policy
- SwapUsage > 0 (critical: causes severe performance degradation)
- Memory pressure causing replication sync failures

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `INFO memory` | LOW | Memory statistics only |
| `--bigkeys` | MEDIUM | Exposes key names and sizes |
| `describe-cache-parameters` | LOW | Parameter configuration only |
| `get-metric-statistics` (memory, evictions) | LOW | Operational metrics only |
| `MEMORY USAGE <key>` | MEDIUM | Exposes specific key memory consumption |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to free memory in production
- NEVER suggest disabling AUTH to reduce memory overhead
- NEVER suggest disabling encryption in transit to free memory
- NEVER suggest reducing node count during peak traffic or memory pressure
- NEVER set maxmemory-reserved below 15% (risks OOM during replication)

## Phase 3 — Rollback

If memory remediation changes cause issues:
1. Revert eviction policy to previous setting in parameter group and reboot if required
2. Revert maxmemory-reserved to previous value in parameter group
3. If node type was scaled up and causes issues, scale back (note: ensure data fits in smaller node)
4. If active defragmentation was enabled and causes CPU impact, disable it in parameter group
5. Monitor DatabaseMemoryUsagePercentage and Evictions after rollback

## Output Format

```yaml
root_cause: "memory_pressure — <specific_cause>"
evidence:
  - type: memory_usage
    content: "<DatabaseMemoryUsagePercentage and INFO memory>"
  - type: evictions
    content: "<eviction rate and policy>"
  - type: big_keys
    content: "<large keys identified>"
severity: HIGH
mitigation:
  immediate: "Adjust eviction policy or scale up node type"
  long_term: "Implement TTL management, optimize data structures, right-size the cluster"
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
