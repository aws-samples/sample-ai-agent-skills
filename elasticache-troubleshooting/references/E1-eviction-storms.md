---
title: "E1 — ElastiCache Eviction Storms"
description: "Diagnose sudden spikes in eviction rates causing cache hit ratio degradation"
status: active
severity: HIGH
triggers:
  - "eviction storm"
  - "mass evictions"
  - "cache miss spike"
  - "eviction rate high"
  - "cache hit ratio drop"
owner: devops-agent
objective: "Identify the cause of eviction storms and stabilize cache hit ratio"
context: "An eviction storm occurs when the eviction rate spikes dramatically, causing a cascade of cache misses that overload the backend database. This can happen when memory suddenly fills (large key ingestion, TTL expiry cliff), when maxmemory-reserved is insufficient, or when the eviction policy is misconfigured. The resulting cache miss storm can cause a thundering herd effect on the origin database."
---

## Phase 1 — Triage

MUST:
- Check eviction rate: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name Evictions --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check cache hit ratio: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name CacheHitRate --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Check memory usage: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name DatabaseMemoryUsagePercentage --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check eviction policy: `aws elasticache describe-cache-parameters --cache-parameter-group-name <param-group> --query "Parameters[?ParameterName=='maxmemory-policy']"`
- Check memory info: `redis-cli -h <endpoint> -p 6379 INFO memory`

SHOULD:
- Check for TTL expiry cliffs (many keys expiring at the same time): `redis-cli -h <endpoint> -p 6379 INFO keyspace`
- Identify large keys consuming disproportionate memory: `redis-cli -h <endpoint> -p 6379 --bigkeys`
- Check if a bulk data load recently occurred
- Check maxmemory-reserved: `aws elasticache describe-cache-parameters --cache-parameter-group-name <param-group> --query "Parameters[?ParameterName=='reserved-memory-percent']"`

MAY:
- Check for pub/sub output buffer growth: `redis-cli -h <endpoint> -p 6379 INFO clients`
- Analyze key TTL distribution to identify expiry patterns
- Check if replication buffer is consuming reserved memory

## Phase 2 — Remediate

MUST:
- Scale up node type or add shards to increase available memory
- Ensure eviction policy matches workload (allkeys-lru for cache-only)
- Ensure maxmemory-reserved is at least 25%

SHOULD:
- Spread TTL values with jitter to avoid expiry cliffs (TTL + random(0, 300))
- Implement cache warming for critical keys
- Set up CloudWatch alarms on Evictions metric

MAY:
- Implement circuit breaker to protect backend database during eviction storms
- Use client-side caching as a secondary cache layer
- Consider ElastiCache Serverless for automatic memory scaling

## Common Issues

- symptoms: "Evictions spike from 0 to thousands per minute"
  diagnosis: "TTL expiry cliff — many keys set with the same TTL expire simultaneously, filling memory with new writes."
  resolution: "Add jitter to TTL values. Implement cache warming. Scale up memory."

- symptoms: "Evictions constant at high rate, cache hit ratio below 50%"
  diagnosis: "Dataset significantly exceeds available memory. Continuous eviction-refill cycle."
  resolution: "Scale up node type or add shards. Review data retention policy. Remove unnecessary keys."

- symptoms: "Eviction storm after bulk data load"
  diagnosis: "Bulk ingestion filled memory, triggering mass eviction of existing cached data."
  resolution: "Rate-limit bulk loads. Pre-size the cluster for expected data volume. Use separate cluster for bulk data."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Scale up node type for more memory | YELLOW | Requires failover for Redis with replication; brief downtime |
| Adjust eviction policy | YELLOW | Changes eviction behavior; may evict different keys |
| Set maxmemory-reserved to 25% | YELLOW | Reduces available data memory; may trigger evictions |
| Spread TTL values with jitter | GREEN | Application-level improvement; prevents expiry cliffs |
| Implement cache warming | GREEN | Application-level improvement; no infrastructure risk |
| Implement circuit breaker | GREEN | Application-level resilience; protects backend |

## Escalation Conditions

- Eviction storm causing thundering herd on backend database
- Cache hit ratio below 30% due to mass evictions
- Eviction storm triggered by bulk data load in production
- TTL expiry cliff affecting >50% of cached keys simultaneously
- Backend database unable to handle the cache miss storm

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-metric-statistics` (Evictions, CacheHitRate) | LOW | Operational metrics only |
| `INFO memory` | LOW | Memory statistics only |
| `--bigkeys` | MEDIUM | Exposes key names and sizes |
| `describe-cache-parameters` | LOW | Parameter configuration only |
| `INFO clients` | LOW | Connection statistics only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) during an eviction storm (worsens the problem)
- NEVER suggest disabling AUTH to reduce memory overhead
- NEVER suggest disabling encryption in transit to free memory
- NEVER suggest reducing node count during peak traffic or eviction storms
- NEVER perform bulk data loads without rate limiting on a production cache

## Phase 3 — Rollback

If eviction storm remediation causes issues:
1. If node type was scaled up and causes issues, scale back (note: causes failover)
2. Revert eviction policy to previous setting in parameter group
3. If TTL jitter was added and causes unexpected expiration patterns, revert TTL logic
4. If cache warming was implemented and causes excessive load, reduce warming rate
5. If circuit breaker was added and is too aggressive, adjust thresholds or disable temporarily
6. Monitor Evictions and CacheHitRate after rollback

## Output Format

```yaml
root_cause: "eviction_storm — <specific_cause>"
evidence:
  - type: evictions
    content: "<eviction rate over time>"
  - type: memory_usage
    content: "<DatabaseMemoryUsagePercentage>"
  - type: cache_hit_ratio
    content: "<CacheHitRate metrics>"
severity: HIGH
mitigation:
  immediate: "Scale up memory or optimize eviction policy"
  long_term: "Implement TTL jitter, cache warming, and memory monitoring"
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
