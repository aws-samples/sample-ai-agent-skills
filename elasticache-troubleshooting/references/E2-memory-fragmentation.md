---
title: "E2 — ElastiCache Memory Fragmentation"
description: "Diagnose high memory fragmentation ratio causing inefficient memory usage"
status: active
severity: MEDIUM
triggers:
  - "memory fragmentation"
  - "mem_fragmentation_ratio"
  - "memory inefficiency"
  - "RSS high"
  - "fragmentation"
owner: devops-agent
objective: "Identify and resolve memory fragmentation to improve memory efficiency"
context: "Memory fragmentation occurs when the allocator (jemalloc) cannot efficiently reuse freed memory blocks. The fragmentation ratio (mem_fragmentation_ratio = used_memory_rss / used_memory) indicates efficiency. A ratio of 1.0-1.5 is normal. Above 1.5 indicates fragmentation. Below 1.0 indicates swapping (critical). Fragmentation increases with frequent key creation/deletion of varying sizes."
---

## Phase 1 — Triage

MUST:
- Check fragmentation ratio: `redis-cli -h <endpoint> -p 6379 INFO memory | grep mem_fragmentation_ratio`
- Check memory details: `redis-cli -h <endpoint> -p 6379 INFO memory`
- Check used_memory vs used_memory_rss: `redis-cli -h <endpoint> -p 6379 INFO memory | grep -E 'used_memory:|used_memory_rss:'`
- Check DatabaseMemoryUsagePercentage: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name DatabaseMemoryUsagePercentage --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check SwapUsage: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name SwapUsage --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

SHOULD:
- Check if active defragmentation is enabled: `redis-cli -h <endpoint> -p 6379 CONFIG GET activedefrag`
- Check allocator stats: `redis-cli -h <endpoint> -p 6379 MEMORY STATS`
- Identify key patterns causing fragmentation (frequent create/delete of varying sizes)
- Check if the fragmentation ratio has been increasing over time

MAY:
- Check jemalloc stats: `redis-cli -h <endpoint> -p 6379 MEMORY MALLOC-STATS`
- Review key size distribution patterns
- Check if background save is contributing to RSS growth

## Phase 2 — Remediate

MUST:
- If SwapUsage > 0, scale up immediately — swapping causes severe performance degradation
- Enable active defragmentation (Redis 4.0+) in the parameter group: set activedefrag to yes
- If fragmentation ratio > 2.0, consider restarting the node (failover for primary)

SHOULD:
- Configure active defrag thresholds in parameter group: active-defrag-threshold-lower (10), active-defrag-threshold-upper (100)
- Use consistent key sizes where possible to reduce fragmentation
- Monitor mem_fragmentation_ratio over time

MAY:
- Schedule periodic restarts during maintenance windows for persistent fragmentation
- Consider using OBJECT ENCODING to optimize data structure encoding
- Evaluate if key access patterns can be optimized to reduce fragmentation

## Common Issues

- symptoms: "mem_fragmentation_ratio above 2.0"
  diagnosis: "Heavy key churn with varying sizes causing allocator fragmentation."
  resolution: "Enable active defragmentation. If severe, restart the node during maintenance."

- symptoms: "mem_fragmentation_ratio below 1.0"
  diagnosis: "Redis is swapping to disk — used_memory exceeds available physical memory."
  resolution: "CRITICAL: Scale up node type immediately. Swapping causes orders-of-magnitude latency increase."

- symptoms: "Fragmentation increasing steadily over weeks"
  diagnosis: "Gradual fragmentation from normal key lifecycle. Active defrag not enabled."
  resolution: "Enable active defragmentation in the parameter group."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable active defragmentation | GREEN | Background process; minimal performance impact |
| Configure active defrag thresholds | GREEN | Parameter tuning; controls defrag aggressiveness |
| Scale up node type (if swapping) | RED | CRITICAL if swapping; requires immediate action; causes failover |
| Restart node (for severe fragmentation) | YELLOW | Causes failover; clears fragmentation but brief downtime |

## Escalation Conditions

- mem_fragmentation_ratio below 1.0 (CRITICAL: node is swapping)
- SwapUsage > 0 on any node (severe performance degradation)
- mem_fragmentation_ratio above 3.0 (severe fragmentation)
- Fragmentation causing OOM despite available physical memory
- Active defragmentation unable to reduce fragmentation ratio

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `INFO memory` | LOW | Memory statistics only |
| `MEMORY STATS` | LOW | Allocator statistics only |
| `MEMORY MALLOC-STATS` | LOW | jemalloc statistics only |
| `get-metric-statistics` (SwapUsage) | LOW | Operational metrics only |
| `CONFIG GET activedefrag` | LOW | Configuration only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix fragmentation
- NEVER suggest disabling AUTH to reduce memory overhead
- NEVER suggest disabling encryption in transit to free memory
- NEVER suggest reducing node count during peak traffic
- NEVER ignore SwapUsage > 0 — this is a critical condition requiring immediate action

## Phase 3 — Rollback

If fragmentation remediation causes issues:
1. If active defragmentation was enabled and causes CPU impact, disable it: set activedefrag to no in parameter group
2. If active defrag thresholds were changed, revert to previous values
3. If node type was scaled up, scale back if fragmentation is resolved (note: causes failover)
4. If a node restart was performed, monitor fragmentation ratio to ensure it doesn't rapidly return
5. Monitor mem_fragmentation_ratio and SwapUsage after rollback

## Output Format

```yaml
root_cause: "memory_fragmentation — <specific_cause>"
evidence:
  - type: fragmentation_ratio
    content: "<mem_fragmentation_ratio value>"
  - type: memory_info
    content: "<used_memory vs used_memory_rss>"
  - type: swap_usage
    content: "<SwapUsage metric>"
severity: MEDIUM
mitigation:
  immediate: "Enable active defragmentation or scale up if swapping"
  long_term: "Monitor fragmentation ratio, optimize key patterns"
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
