---
title: "F1 — Cache Configuration"
description: "Diagnose AppSync caching configuration and operation issues"
status: active
severity: MEDIUM
triggers:
  - "cache not working"
  - "caching configuration"
  - "cache instance"
  - "cache miss"
  - "enable caching"
owner: devops-agent
objective: "Identify and resolve AppSync caching configuration issues"
context: "AppSync caching requires a provisioned cache instance (t2.small to r5.12xlarge). Caching can be configured at the API level (all resolvers) or per-resolver. Cache TTL ranges from 1 to 3600 seconds. Per-resolver caching overrides API-level settings. The cache key is based on the resolver context (arguments, identity, source)."
---

## Phase 1 — Triage

MUST:
- Check if caching is enabled: `aws appsync get-graphql-api --api-id <api-id>`
- Check the API cache configuration: `aws appsync get-api-cache --api-id <api-id>`
- Verify the cache instance type and status
- Check if per-resolver caching is configured: `aws appsync get-resolver --api-id <api-id> --type-name <type> --field-name <field>` (check cachingConfig)
- Check CloudWatch cache metrics: CacheHitCount, CacheMissCount

SHOULD:
- Verify the cache TTL is appropriate for the data freshness requirements
- Check if the cache key includes the right context fields (arguments, identity)
- Verify the cache instance size is adequate for the data volume

MAY:
- Check cache memory utilization metrics
- Verify encryption settings (at-rest and in-transit)

## Phase 2 — Remediate

MUST:
- Create a cache instance if not provisioned: `aws appsync create-api-cache --api-id <api-id> --type <instance-type> --api-caching-behavior FULL_REQUEST_CACHING --ttl <seconds>`
- Ensure the cache instance status is AVAILABLE
- Configure appropriate TTL for the use case

SHOULD:
- Use per-resolver caching for resolvers with different freshness requirements
- Configure cache keys to include only relevant context (avoid over-specific keys that reduce hit rate)
- Size the cache instance based on expected data volume

MAY:
- Enable encryption at rest and in transit for sensitive data
- Use FULL_REQUEST_CACHING for API-level caching or PER_RESOLVER_CACHING for granular control

## Common Issues

- symptoms: "Caching enabled but all requests are cache misses"
  diagnosis: "Cache key includes identity or source fields that vary per request, making every request unique."
  resolution: "Review cache key configuration. Remove unnecessary context from the cache key."

- symptoms: "CreateApiCache returns error"
  diagnosis: "Invalid instance type or the API already has a cache configured."
  resolution: "Use a valid instance type (t2.small, t2.medium, r4.large, etc.). Delete existing cache first if reconfiguring."

- symptoms: "Cache returns stale data"
  diagnosis: "TTL is too long for the data update frequency."
  resolution: "Reduce TTL or use cache invalidation after mutations."

## Output Format

```yaml
root_cause: "cache_configuration — <specific_cause>"
evidence:
  - type: cache_config
    content: "<cache instance type, TTL, behavior>"
  - type: cache_metrics
    content: "<CacheHitCount and CacheMissCount>"
severity: MEDIUM
mitigation:
  immediate: "Fix cache configuration or instance provisioning"
  long_term: "Tune cache keys and TTL based on access patterns"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Create cache instance | YELLOW | New infrastructure provisioning |
| Ensure cache instance AVAILABLE | GREEN | Status verification |
| Configure appropriate TTL | GREEN | Cache tuning |
| Use per-resolver caching | GREEN | Granular cache configuration |
| Configure cache keys | YELLOW | Changes cache behavior |
| Enable encryption | GREEN | Security improvement |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **Cache configuration**: Instance type, TTL, behavior — INTERNAL
- **Cache metrics**: Hit/miss counts — INTERNAL
- **Cached data**: Query results stored in cache — CONFIDENTIAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER cache responses containing user-specific sensitive data without proper cache keys
- NEVER use large cache instances without cost justification

## Phase 3 — Rollback

1. Delete the cache instance if it was newly created: `aws appsync delete-api-cache --api-id <api-id>`
2. Revert per-resolver caching configuration
3. Revert cache key configuration
4. Verify API functions correctly without caching
5. Monitor latency metrics after cache removal

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
