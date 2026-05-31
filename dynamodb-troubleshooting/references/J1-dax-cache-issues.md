---
title: "J1 — DAX Cache Issues"
description: "Diagnose DAX cache invalidation, TTL settings, and item cache vs query cache behavior"
status: active
severity: MEDIUM
triggers:
  - "DAX cache"
  - "stale data DAX"
  - "cache invalidation"
  - "DAX TTL"
  - "item cache vs query cache"
owner: devops-agent
objective: "Identify and resolve DAX caching issues"
context: "DAX has two caches: item cache (for GetItem/BatchGetItem) and query cache (for Query/Scan). Each has a separate TTL (default 5 minutes). Item cache is keyed by table name and primary key. Query cache is keyed by the exact query parameters. Writes through DAX update the item cache but do NOT invalidate the query cache. Direct DynamoDB writes bypass DAX entirely. DAX only supports eventually consistent reads."
---

## Phase 1 — Triage

MUST:
- Check DAX cluster status: `aws dax describe-clusters --cluster-names <cluster-name>`
- Check DAX cache hit/miss metrics: `aws cloudwatch get-metric-statistics --namespace AWS/DAX --metric-name ItemCacheHits --dimensions Name=ClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check query cache metrics: `aws cloudwatch get-metric-statistics --namespace AWS/DAX --metric-name QueryCacheMisses --dimensions Name=ClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check DAX cluster parameters for TTL settings: `aws dax describe-parameter-groups`

SHOULD:
- Check if writes go through DAX or directly to DynamoDB (direct writes bypass item cache)
- Verify item cache TTL and query cache TTL settings
- Check DAX node CPU and memory utilization: `aws cloudwatch get-metric-statistics --namespace AWS/DAX --metric-name CPUUtilization --dimensions Name=ClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Check for cache evictions: `aws cloudwatch get-metric-statistics --namespace AWS/DAX --metric-name ItemCacheEvictions --dimensions Name=ClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`

MAY:
- Check DAX cluster node count and instance type
- Review if the working set fits in DAX memory
- Check network connectivity between application and DAX cluster

## Phase 2 — Remediate

MUST:
- If stale data is the issue, reduce cache TTL or route writes through DAX
- If cache misses are high, verify the working set fits in DAX memory
- Ensure all writes go through DAX if you need the item cache to stay current

SHOULD:
- Set appropriate TTLs: lower for frequently changing data, higher for static data
- Route all writes through DAX to keep the item cache consistent
- Understand that query cache is NOT invalidated by writes — set query cache TTL accordingly

MAY:
- Scale DAX cluster (add nodes or use larger instance types) for larger working sets
- Implement cache-aside pattern for data that changes frequently
- Use separate DAX clusters for different access patterns

## Common Issues

- symptoms: "DAX returns stale data after writes"
  diagnosis: "Writes go directly to DynamoDB, bypassing DAX. The item cache still has the old value until TTL expires."
  resolution: "Route all writes through DAX. Or reduce item cache TTL. Or accept eventual consistency."

- symptoms: "Query results are stale even after writing through DAX"
  diagnosis: "DAX query cache is separate from item cache. Writes update the item cache but NOT the query cache."
  resolution: "Reduce query cache TTL. Or invalidate by re-running the query after writes. Query cache TTL should be set based on staleness tolerance."

- symptoms: "DAX cache hit rate is very low"
  diagnosis: "Working set is larger than DAX memory, causing frequent evictions. Or access patterns are not cache-friendly (unique queries)."
  resolution: "Scale DAX cluster with more memory. Review if the access pattern benefits from caching (repeated reads of same data)."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Reduce cache TTL | GREEN | Configuration change; may increase DynamoDB load |
| Route writes through DAX | GREEN | Application-level routing change |
| Scale DAX cluster (add nodes) | YELLOW | Infrastructure change; brief rebalancing period |
| Change DAX instance type | YELLOW | Requires cluster modification; brief unavailability |

## Escalation Conditions

- DAX returning stale data causing application logic errors in production
- DAX cache hit rate below 50% (ineffective caching)
- DAX cluster memory exhausted causing high eviction rates
- DAX node failure impacting application availability
- DAX consistency issues affecting data-sensitive operations

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-clusters` (DAX) | MEDIUM | Exposes cluster configuration and endpoints |
| `get-metric-statistics` (cache metrics) | LOW | Operational metrics only |
| `describe-parameter-groups` (DAX) | LOW | Parameter configuration only |

## Prohibited Actions

- NEVER suggest deleting a table to fix DAX cache issues
- NEVER suggest disabling encryption on the DAX cluster
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER use DAX for strongly consistent reads (bypasses cache, adds latency)

## Phase 3 — Rollback

If DAX changes cause issues:
1. If cache TTL was reduced and increases DynamoDB load, increase TTL back to previous value
2. If writes were routed through DAX and cause issues, revert to direct DynamoDB writes
3. If DAX cluster was scaled and causes instability, revert to previous node count/type
4. If DAX is causing widespread issues, bypass DAX entirely by switching application to use DynamoDB client directly
5. Monitor DynamoDB consumed capacity after bypassing DAX (expect increased load)

## Output Format

```yaml
root_cause: "dax_cache — <specific_cause>"
evidence:
  - type: cache_metrics
    content: "<hit rate, miss rate, evictions>"
  - type: ttl_config
    content: "<item cache TTL and query cache TTL>"
  - type: write_pattern
    content: "<writes through DAX vs direct to DynamoDB>"
severity: MEDIUM
dax_cache_analysis:
  item_cache_hit_rate: "<percentage>"
  query_cache_hit_rate: "<percentage>"
  item_cache_ttl_ms: "<ttl>"
  query_cache_ttl_ms: "<ttl>"
  writes_through_dax: "yes | no"
  eviction_rate: "<rate>"
mitigation:
  immediate: "Adjust cache TTL and route writes through DAX"
  long_term: "Right-size DAX cluster and optimize access patterns"
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
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting a table to fix issues"
  - "NEVER run full table scans on production tables"
