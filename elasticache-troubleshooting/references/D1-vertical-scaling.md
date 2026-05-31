---
title: "D1 — ElastiCache Vertical Scaling Issues"
description: "Diagnose issues with scaling up or down ElastiCache node types"
status: active
severity: MEDIUM
triggers:
  - "scale up"
  - "scale down"
  - "node type change"
  - "modify cache cluster"
  - "vertical scaling"
  - "instance type"
owner: devops-agent
objective: "Successfully complete vertical scaling operations and resolve scaling failures"
context: "Vertical scaling changes the node type (e.g., cache.r6g.large to cache.r6g.xlarge). For Redis with replication, ElastiCache performs a rolling upgrade — replicas are scaled first, then failover occurs, then the old primary is scaled. This minimizes downtime but causes a brief failover. For Memcached, nodes are replaced one at a time (data on replaced nodes is lost). Scaling requires sufficient capacity in the target AZ."
---

## Phase 1 — Triage

MUST:
- Check current cluster configuration: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].{NodeType:CacheNodeType,Engine:Engine,Status:CacheClusterStatus}'`
- Check replication group status: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id>`
- Check pending modifications: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].PendingModifiedValues'`
- Check recent events for scaling errors: `aws elasticache describe-events --source-type cache-cluster --duration 1440`
- Verify target node type is available: `aws elasticache list-allowed-node-type-modifications --replication-group-id <repl-group-id>`

SHOULD:
- Check current memory and CPU utilization to validate scaling direction
- Verify the target node type supports the current engine version
- Check if a maintenance window is configured: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].PreferredMaintenanceWindow'`
- Ensure sufficient data fits in the target node type (for scale-down)

MAY:
- Check service quotas for the target node type
- Review CloudTrail for modify API calls
- Check if reserved nodes exist for the target node type

## Phase 2 — Remediate

MUST:
- Use list-allowed-node-type-modifications to verify valid target node types
- For Redis with replication, expect a brief failover during scaling
- For Memcached, plan for data loss on each node as it is replaced
- Apply scaling during low-traffic periods if using apply-immediately

SHOULD:
- Scale during the maintenance window to minimize impact: `aws elasticache modify-replication-group --replication-group-id <repl-group-id> --cache-node-type <new-type> --no-apply-immediately`
- Monitor the scaling operation via describe-events
- Notify application teams about the brief failover (Redis) or data loss (Memcached)

MAY:
- Consider cluster mode resharding (horizontal scaling) as an alternative to vertical scaling
- Purchase reserved nodes for the target node type to reduce costs
- Test scaling in a non-production environment first

## Common Issues

- symptoms: "Modify operation stuck in 'modifying' status"
  diagnosis: "Scaling operation is in progress. Large datasets take longer to migrate."
  resolution: "Wait for completion. Monitor events. Scaling can take 10-30+ minutes depending on data size."

- symptoms: "InsufficientCacheClusterCapacity error"
  diagnosis: "Target node type is not available in the requested AZ."
  resolution: "Try a different node type or AZ. Check AWS service health for capacity issues."

- symptoms: "Data loss after scaling Memcached cluster"
  diagnosis: "Expected behavior — Memcached nodes are replaced, losing all cached data."
  resolution: "This is by design. Ensure the application can repopulate the cache. Consider Redis if data persistence is needed."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Scale up during maintenance window | GREEN | Planned operation; minimal impact with proper scheduling |
| Scale up with apply-immediately | YELLOW | Causes immediate failover for Redis with replication |
| Scale down node type | YELLOW | Risk of data not fitting in smaller node; causes failover |
| Notify application teams | GREEN | Communication only; no operational impact |

## Escalation Conditions

- InsufficientCacheClusterCapacity error (target node type unavailable)
- Scaling operation stuck in 'modifying' status for more than 1 hour
- Scaling required during active production incident
- Data loss after Memcached scaling (expected but needs communication)
- Scaling to a node type that doesn't support current engine version

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-cache-clusters` (node type, status) | MEDIUM | Exposes cluster configuration |
| `list-allowed-node-type-modifications` | LOW | Available node types only |
| `describe-events` | LOW | Operational events only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) before scaling
- NEVER suggest disabling AUTH during scaling operations
- NEVER suggest disabling encryption in transit during scaling
- NEVER suggest reducing node count during peak traffic
- NEVER scale down a Redis node type without verifying data fits in the smaller node's memory

## Phase 3 — Rollback

If vertical scaling causes issues:
1. Scale back to previous node type: `aws elasticache modify-replication-group --replication-group-id <repl-group-id> --cache-node-type <previous-type> --apply-immediately` (note: causes another failover for Redis)
2. If scaling is stuck in 'modifying', wait for completion or contact AWS Support
3. For Memcached, data loss during scaling is permanent — application must repopulate cache
4. If the new node type causes performance issues, scale to a different type
5. Monitor cluster status and performance metrics after rollback

## Output Format

```yaml
root_cause: "vertical_scaling — <specific_cause>"
evidence:
  - type: cluster_config
    content: "<current node type and status>"
  - type: allowed_modifications
    content: "<valid target node types>"
  - type: events
    content: "<scaling events>"
severity: MEDIUM
mitigation:
  immediate: "Resolve scaling failure or wait for completion"
  long_term: "Plan scaling operations during maintenance windows with proper notification"
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
