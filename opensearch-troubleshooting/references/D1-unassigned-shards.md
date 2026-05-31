---
title: "D1 — OpenSearch Unassigned Shards"
description: "Diagnose and resolve unassigned shards causing cluster health degradation"
status: active
severity: HIGH
triggers:
  - "unassigned shard"
  - "shard not allocated"
  - "UNASSIGNED"
  - "allocation failed"
  - "shard allocation"
owner: devops-agent
objective: "Identify why shards are unassigned and restore full shard allocation"
context: "Unassigned shards indicate that OpenSearch cannot place a shard on any node. Unassigned primaries cause RED health; unassigned replicas cause YELLOW. Common reasons include insufficient disk space, allocation filtering, node count less than replica count + 1, zone awareness constraints, or shard allocation being disabled. The _cluster/allocation/explain API provides the specific reason."
---

## Phase 1 — Triage

MUST:
- List unassigned shards: `curl -XGET "https://<endpoint>/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason,unassigned.at,node&s=state"`
- Get allocation explanation: `curl -XGET "https://<endpoint>/_cluster/allocation/explain?pretty"`
- Check disk allocation: `curl -XGET "https://<endpoint>/_cat/allocation?v"`
- Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`
- Check node count and roles: `curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,node.role,disk.used_percent"`

SHOULD:
- Check allocation settings: `curl -XGET "https://<endpoint>/_cluster/settings?include_defaults=true&flat_settings=true&filter_path=*.cluster.routing.allocation.*"`
- Check index-level allocation filtering: `curl -XGET "https://<endpoint>/<index>/_settings?pretty&filter_path=*.settings.index.routing.allocation"`
- Check zone awareness: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.ClusterConfig.{ZoneAwareness:ZoneAwarenessEnabled,AZCount:ZoneAwarenessConfig.AvailabilityZoneCount}'`

MAY:
- Check recovery progress: `curl -XGET "https://<endpoint>/_cat/recovery?v&active_only=true"`
- Check pending tasks: `curl -XGET "https://<endpoint>/_cat/pending_tasks?v"`

## Phase 2 — Remediate

MUST:
- If disk full: free space by deleting indices or increasing volume size
- If allocation disabled: re-enable: `curl -XPUT "https://<endpoint>/_cluster/settings" -H 'Content-Type: application/json' -d '{"persistent":{"cluster.routing.allocation.enable":"all"}}'`
- If not enough nodes for replicas: reduce replica count or add nodes
- Retry failed allocations: `curl -XPOST "https://<endpoint>/_cluster/reroute?retry_failed=true" -H 'Content-Type: application/json' -d '{"commands":[]}'`

SHOULD:
- Remove restrictive allocation filters if no longer needed
- Ensure zone awareness configuration matches AZ count
- Monitor shard allocation after remediation

MAY:
- Manually allocate a shard as last resort (risk of data loss for stale shards): `curl -XPOST "https://<endpoint>/_cluster/reroute" -H 'Content-Type: application/json' -d '{"commands":[{"allocate_stale_primary":{"index":"<index>","shard":0,"node":"<node>","accept_data_loss":true}}]}'`

## Common Issues

- symptoms: "UNASSIGNED reason: ALLOCATION_FAILED"
  diagnosis: "Previous allocation attempt failed (e.g., disk full, corrupt shard)."
  resolution: "Fix underlying issue. Retry with _cluster/reroute?retry_failed=true."

- symptoms: "UNASSIGNED reason: NODE_LEFT"
  diagnosis: "The node holding the shard left the cluster."
  resolution: "Wait for node to rejoin or for replica promotion. Check node health."

- symptoms: "UNASSIGNED reason: INDEX_CREATED but no allocation"
  diagnosis: "New index created but no eligible nodes (disk full, filtering, zone awareness)."
  resolution: "Check allocation explain API for specific reason. Fix constraints."

## Output Format

```yaml
root_cause: "unassigned_shards — <specific_cause>"
evidence:
  - type: unassigned_list
    content: "<_cat/shards showing unassigned>"
  - type: allocation_explain
    content: "<_cluster/allocation/explain output>"
  - type: disk_allocation
    content: "<_cat/allocation output>"
severity: HIGH
mitigation:
  immediate: "Resolve allocation constraint and retry shard assignment"
  long_term: "Monitor disk usage, right-size cluster, set up allocation alarms"
```


## Safety Ratings
```
safety_ratings:
  - "List unassigned shards and allocation explain: GREEN — read-only diagnostics"
  - "Re-enable allocation: YELLOW — triggers shard movement across nodes"
  - "Retry failed allocations: YELLOW — attempts to place previously failed shards"
  - "Reduce replica count: YELLOW — reduces data redundancy"
  - "Allocate stale primary: RED — accepts data loss for the shard"
```

## Escalation Conditions
- Domain serves production search
- Unassigned primary shards causing RED cluster (data unavailable)
- Fix requires blue/green deployment for node or storage changes
- Stale primary allocation needed (data loss risk)
- Allocation constraints preventing shard placement

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Shard allocation details: cluster topology"
    - "Index names: data structure information"
    - "Allocation explain output: infrastructure details"
  handling: "Do not expose shard allocation details or cluster topology externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER use allocate_stale_primary without explicit confirmation of acceptable data loss
- NEVER disable shard allocation permanently

## Phase 3 — Rollback
- If allocation was re-enabled: can be disabled again if causing issues (not recommended)
- If replica count was reduced: increase replicas back to original value
- If stale primary was allocated: CANNOT be undone — data loss is permanent
- If allocation filters were removed: re-add original filter settings

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling fine-grained access control"
  - "NEVER suggest public access domains"
  - "NEVER suggest disabling encryption at rest"
