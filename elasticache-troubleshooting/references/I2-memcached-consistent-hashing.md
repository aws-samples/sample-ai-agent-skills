---
title: "I2 — Memcached Consistent Hashing Issues"
description: "Diagnose data distribution and consistent hashing problems in Memcached clusters"
status: active
severity: MEDIUM
triggers:
  - "consistent hashing"
  - "key distribution"
  - "Memcached scaling"
  - "cache miss after scaling"
  - "uneven distribution"
  - "hot node"
owner: devops-agent
objective: "Resolve data distribution issues and minimize cache disruption during node changes"
context: "Memcached distributes data across nodes using client-side hashing. Standard modulo hashing causes massive redistribution when nodes are added/removed. Consistent hashing minimizes redistribution — only ~1/N keys are remapped when a node changes. The hashing algorithm is implemented in the client library, not in Memcached itself. Different client libraries may use different hashing algorithms."
---

## Phase 1 — Triage

MUST:
- Check node count and status: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --show-cache-node-info`
- Check cache hit rate per node: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name GetHits --dimensions Name=CacheClusterId,Value=<cluster-id>,Name=CacheNodeId,Value=<node-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check evictions per node: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name Evictions --dimensions Name=CacheClusterId,Value=<cluster-id>,Name=CacheNodeId,Value=<node-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check CurrItems per node: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name CurrItems --dimensions Name=CacheClusterId,Value=<cluster-id>,Name=CacheNodeId,Value=<node-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify the client hashing algorithm (consistent hashing vs modulo)

SHOULD:
- Check if nodes were recently added or removed
- Compare item count and memory usage across nodes for balance
- Verify all clients use the same hashing algorithm and node list order
- Check CPU utilization per node for hot spots

MAY:
- Test key distribution with sample keys
- Review client library documentation for hashing configuration
- Check if virtual nodes (vnodes) are configured for better distribution

## Phase 2 — Remediate

MUST:
- Use consistent hashing in the client library (not modulo hashing)
- Ensure all application instances use the same client configuration and node list order
- Accept that ~1/N keys will be remapped when adding/removing nodes

SHOULD:
- Use the ElastiCache Cluster Client with auto-discovery for automatic node list management
- Configure virtual nodes (vnodes) for more even distribution (typically 100-200 per node)
- Implement cache warming after node changes to repopulate lost keys

MAY:
- Pre-warm the cache before removing nodes
- Use a gradual scaling approach (add one node at a time)
- Monitor cache hit ratio closely after node changes

## Common Issues

- symptoms: "Massive cache miss spike after adding a node"
  diagnosis: "Using modulo hashing — adding a node remaps most keys."
  resolution: "Switch to consistent hashing. With consistent hashing, only ~1/N keys are remapped."

- symptoms: "One node has significantly more items than others"
  diagnosis: "Uneven key distribution due to insufficient virtual nodes or poor hash function."
  resolution: "Increase virtual node count in the client. Verify consistent hashing is configured."

- symptoms: "Different application instances route the same key to different nodes"
  diagnosis: "Inconsistent node list order or different hashing algorithms across clients."
  resolution: "Use auto-discovery to ensure all clients have the same node list. Standardize client library."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Switch to consistent hashing | GREEN | Application-level client configuration change |
| Use ElastiCache Cluster Client with auto-discovery | GREEN | Application-level improvement |
| Configure virtual nodes (vnodes) | GREEN | Client-level configuration; improves distribution |
| Implement cache warming after node changes | GREEN | Application-level improvement |

## Escalation Conditions

- Massive cache miss spike after adding/removing a node (modulo hashing)
- Uneven data distribution causing hot nodes in production
- Different application instances routing same key to different nodes
- Node addition/removal causing backend database overload from cache misses
- Client library inconsistency across application fleet

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-cache-clusters` (node info) | MEDIUM | Exposes cluster configuration |
| `get-metric-statistics` (per-node metrics) | LOW | Operational metrics only |
| Client configuration review | LOW | Configuration only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix hashing issues
- NEVER suggest disabling AUTH (not applicable to Memcached standard, but do not remove security controls)
- NEVER suggest disabling encryption in transit to simplify client configuration
- NEVER suggest reducing node count during peak traffic
- NEVER mix different hashing algorithms across application instances

## Phase 3 — Rollback

If consistent hashing changes cause issues:
1. Revert client hashing configuration to previous algorithm (note: will cause key redistribution)
2. If ElastiCache Cluster Client was deployed and causes issues, revert to previous client library
3. If virtual node count was changed, revert to previous configuration
4. If cache warming was implemented and causes excessive backend load, reduce warming rate
5. Monitor cache hit rate and per-node distribution after rollback

## Output Format

```yaml
root_cause: "consistent_hashing — <specific_cause>"
evidence:
  - type: node_distribution
    content: "<item count and memory per node>"
  - type: client_config
    content: "<hashing algorithm and node list>"
  - type: cache_hit_rate
    content: "<per-node hit rates>"
severity: MEDIUM
mitigation:
  immediate: "Switch to consistent hashing and standardize client configuration"
  long_term: "Use auto-discovery, configure virtual nodes, implement cache warming"
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
