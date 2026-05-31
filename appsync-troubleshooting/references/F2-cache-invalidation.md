---
title: "F2 — Cache Invalidation"
description: "Diagnose AppSync cache invalidation and stale data issues"
status: active
severity: MEDIUM
triggers:
  - "stale data"
  - "cache invalidation"
  - "flush cache"
  - "old data returned"
  - "cache not updating"
owner: devops-agent
objective: "Identify and resolve cache invalidation issues causing stale data"
context: "AppSync cache entries expire based on TTL. Manual invalidation can be done via the FlushApiCache API or by using cache invalidation in mutation resolvers. Common issues include stale data after mutations, inability to invalidate specific entries, and confusion between API-level flush and per-resolver invalidation."
---

## Phase 1 — Triage

MUST:
- Check the current cache configuration and TTL: `aws appsync get-api-cache --api-id <api-id>`
- Verify if mutations are configured to invalidate cache entries
- Check when the cache was last flushed: review CloudTrail for FlushApiCache events
- Determine if the stale data issue is TTL-related or invalidation-related

SHOULD:
- Check if mutation resolvers include cache invalidation logic
- Verify the cache key structure to understand what gets invalidated
- Check if per-resolver caching TTLs differ from API-level TTL

MAY:
- Check CloudWatch CacheHitCount trends to understand cache behavior
- Review the mutation resolver response mapping for invalidation calls

## Phase 2 — Remediate

MUST:
- For immediate stale data: flush the entire cache: `aws appsync flush-api-cache --api-id <api-id>`
- For ongoing issues: reduce TTL to match data freshness requirements
- Add cache invalidation logic to mutation resolvers

SHOULD:
- Use per-resolver caching with appropriate TTLs for different data types
- Implement cache-aside pattern: invalidate on write, populate on read
- Use shorter TTLs for frequently updated data

MAY:
- Implement selective cache invalidation using custom cache keys
- Use event-driven cache invalidation via DynamoDB Streams or EventBridge

## Common Issues

- symptoms: "Data updated via mutation but queries return old data"
  diagnosis: "Cache TTL hasn't expired and mutation doesn't invalidate the cache."
  resolution: "Add cache invalidation in the mutation resolver or reduce TTL."

- symptoms: "FlushApiCache doesn't seem to work"
  diagnosis: "Cache flush is asynchronous. New cache entries may be created before flush completes."
  resolution: "Wait for flush to complete. Consider reducing TTL instead of frequent flushes."

- symptoms: "Some queries return fresh data, others return stale"
  diagnosis: "Per-resolver caching with different TTLs. Some resolvers have longer TTLs."
  resolution: "Review per-resolver caching configuration and align TTLs."

## Output Format

```yaml
root_cause: "cache_invalidation — <specific_cause>"
evidence:
  - type: cache_config
    content: "<TTL and caching behavior>"
  - type: mutation_config
    content: "<mutation resolver cache invalidation logic>"
severity: MEDIUM
mitigation:
  immediate: "Flush cache or reduce TTL"
  long_term: "Implement mutation-triggered cache invalidation"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Flush entire cache | YELLOW | Clears all cached data |
| Reduce TTL | GREEN | Cache tuning |
| Add cache invalidation to mutation resolvers | YELLOW | Resolver code change |
| Use per-resolver caching with TTLs | GREEN | Cache configuration |
| Implement cache-aside pattern | YELLOW | Resolver code change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **Cache TTL**: Data freshness configuration — INTERNAL
- **Mutation config**: Cache invalidation logic — INTERNAL
- **Cached data**: Stale vs fresh data state — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER flush cache during peak traffic without understanding performance impact
- NEVER set TTL to 0 (effectively disables caching)

## Phase 3 — Rollback

1. Revert TTL changes to previous values
2. Revert mutation resolver cache invalidation logic
3. Flush cache to clear any stale data: `aws appsync flush-api-cache --api-id <api-id>`
4. Verify data freshness after rollback
5. Monitor cache hit/miss ratios after changes

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
  - "NEVER suggest removing authentication from APIs"
  - "NEVER suggest API_KEY auth for production"
  - "NEVER suggest disabling logging"
