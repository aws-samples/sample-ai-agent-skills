---
title: "D2 — OpenSearch Shard Imbalance"
description: "Diagnose and resolve uneven shard distribution across cluster nodes"
status: active
severity: MEDIUM
triggers:
  - "shard imbalance"
  - "uneven shards"
  - "hot node"
  - "shard skew"
  - "unbalanced cluster"
owner: devops-agent
objective: "Identify shard distribution imbalance and rebalance shards across nodes"
context: "Shard imbalance occurs when some nodes hold significantly more shards or data than others, leading to hot spots. This causes uneven CPU, memory, and disk usage. Common causes include allocation awareness misconfiguration, index-level allocation filtering, or adding new nodes without rebalancing. OpenSearch automatically rebalances, but constraints can prevent it."
---

## Phase 1 — Triage

MUST:
- Check shard distribution per node: `curl -XGET "https://<endpoint>/_cat/allocation?v"`
- Check per-node resource usage: `curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,heap.percent,cpu,disk.used_percent,shards"`
- Check shard count per node: `curl -XGET "https://<endpoint>/_cat/shards?v&h=index,shard,prirep,node&s=node" | awk '{print $4}' | sort | uniq -c | sort -rn`
- Check rebalance settings: `curl -XGET "https://<endpoint>/_cluster/settings?include_defaults=true&flat_settings=true&filter_path=*.cluster.routing.rebalance.*"`
- Check allocation awareness: `curl -XGET "https://<endpoint>/_cluster/settings?include_defaults=true&flat_settings=true&filter_path=*.cluster.routing.allocation.awareness.*"`

SHOULD:
- Check for index-level allocation filters: `curl -XGET "https://<endpoint>/_all/_settings?pretty&filter_path=*.settings.index.routing.allocation"`
- Check zone awareness config: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.ClusterConfig.{ZoneAwareness:ZoneAwarenessEnabled,AZCount:ZoneAwarenessConfig.AvailabilityZoneCount}'`
- Check for recently added nodes

MAY:
- Check total shard weight per node: `curl -XGET "https://<endpoint>/_cat/shards?v&h=node,store&s=node"`
- Review cluster routing allocation settings

## Phase 2 — Remediate

MUST:
- If rebalancing disabled: re-enable: `curl -XPUT "https://<endpoint>/_cluster/settings" -H 'Content-Type: application/json' -d '{"persistent":{"cluster.routing.rebalance.enable":"all"}}'`
- If allocation filtering causing imbalance: remove or adjust filters
- If new nodes added: wait for automatic rebalancing or trigger manually

SHOULD:
- Ensure zone awareness is properly configured for multi-AZ
- Use index templates to avoid allocation filters on new indices
- Monitor shard distribution after changes

MAY:
- Manually move shards for immediate relief: `curl -XPOST "https://<endpoint>/_cluster/reroute" -H 'Content-Type: application/json' -d '{"commands":[{"move":{"index":"<index>","shard":0,"from_node":"<heavy-node>","to_node":"<light-node>"}}]}'`
- Adjust cluster.routing.allocation.balance.shard threshold

## Common Issues

- symptoms: "One node at 90% disk while others at 50%"
  diagnosis: "Shard imbalance causing uneven disk usage. Large shards concentrated on one node."
  resolution: "Check allocation filters. Enable rebalancing. Move large shards manually if needed."

- symptoms: "New nodes have no shards after scaling"
  diagnosis: "Rebalancing may be slow or disabled. Existing shards not moving to new nodes."
  resolution: "Verify rebalancing is enabled. Wait for automatic rebalancing or trigger reroute."

- symptoms: "CPU hot spot on specific nodes"
  diagnosis: "Hot indices concentrated on specific nodes due to allocation filtering."
  resolution: "Remove allocation filters. Distribute hot indices across all nodes."

## Output Format

```yaml
root_cause: "shard_imbalance — <specific_cause>"
evidence:
  - type: allocation
    content: "<_cat/allocation showing per-node distribution>"
  - type: node_resources
    content: "<per-node CPU, heap, disk usage>"
  - type: rebalance_settings
    content: "<cluster rebalance configuration>"
severity: MEDIUM
mitigation:
  immediate: "Enable rebalancing or manually move shards"
  long_term: "Configure proper allocation awareness, remove unnecessary filters"
```


## Safety Ratings
```
safety_ratings:
  - "Check shard distribution and node resources: GREEN — read-only diagnostics"
  - "Re-enable rebalancing: YELLOW — triggers shard movement"
  - "Remove allocation filters: YELLOW — changes shard placement rules"
  - "Manually move shards: YELLOW — moves data between nodes, impacts I/O"
```

## Escalation Conditions
- Domain serves production search
- Hot node causing performance degradation
- Fix requires blue/green deployment for node changes
- Allocation filters from previous operations causing persistent imbalance

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Shard distribution: cluster topology"
    - "Node resource usage: capacity data"
    - "Allocation settings: infrastructure configuration"
  handling: "Do not expose shard distribution or node resource details externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER move shards manually without understanding the impact on cluster I/O
- NEVER disable rebalancing permanently

## Phase 3 — Rollback
- If rebalancing was re-enabled: can be disabled if causing excessive I/O
- If allocation filters were removed: re-add original filters
- If shards were manually moved: move them back to original nodes
- If balance threshold was adjusted: restore default value

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
