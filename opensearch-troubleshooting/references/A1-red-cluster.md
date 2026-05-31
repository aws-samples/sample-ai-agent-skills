---
title: "A1 — OpenSearch RED Cluster Health"
description: "Diagnose and resolve RED cluster health status caused by unassigned primary shards"
status: active
severity: CRITICAL
triggers:
  - "RED cluster"
  - "cluster health red"
  - "unassigned primary"
  - "index unavailable"
  - "ClusterStatus.red"
owner: devops-agent
objective: "Identify unassigned primary shards, determine root cause, and restore cluster to GREEN or YELLOW"
context: "RED cluster health means at least one primary shard is unassigned. The affected index is partially or fully unavailable, but other indices continue to function. Common causes include node failures, insufficient disk space, shard allocation filtering, or corrupted shards. Automated snapshots may fail while the cluster is RED."
---

## Phase 1 — Triage

MUST:
- Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`
- Identify unassigned shards: `curl -XGET "https://<endpoint>/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason&s=state"`
- Get allocation explanation: `curl -XGET "https://<endpoint>/_cluster/allocation/explain?pretty"`
- Check node status: `curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,heap.percent,disk.used_percent,cpu,node.role"`
- Check CloudWatch for RED duration: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name ClusterStatus.red --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`

SHOULD:
- Check disk space per node: `curl -XGET "https://<endpoint>/_cat/allocation?v"`
- Check domain events: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.{Processing:Processing,UpgradeProcessing:UpgradeProcessing}'`
- Check for recent configuration changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=es.amazonaws.com --max-results 20`
- Check index settings for allocation filtering: `curl -XGET "https://<endpoint>/<affected-index>/_settings?pretty"`

MAY:
- Check shard recovery progress: `curl -XGET "https://<endpoint>/_cat/recovery?v&active_only=true"`
- Check pending tasks: `curl -XGET "https://<endpoint>/_cat/pending_tasks?v"`
- Review cluster-level allocation settings: `curl -XGET "https://<endpoint>/_cluster/settings?include_defaults=true&flat_settings=true&filter_path=*.cluster.routing.allocation.*"`

## Phase 2 — Remediate

MUST:
- If disk full: delete old indices, increase volume size, or add nodes
- If node failure: wait for AWS to replace the node (managed service) or scale the cluster
- If allocation filtering blocks assignment: remove or adjust allocation filters: `curl -XPUT "https://<endpoint>/<index>/_settings" -H 'Content-Type: application/json' -d '{"index.routing.allocation.require._name": null}'`
- If corrupted shard: as last resort, delete the index and restore from snapshot

SHOULD:
- Enable automated snapshots if not already enabled
- Add replica shards for critical indices to prevent future RED from single-node failures
- Scale the cluster to provide more capacity

MAY:
- Reroute unassigned shards manually (use with caution): `curl -XPOST "https://<endpoint>/_cluster/reroute?retry_failed=true" -H 'Content-Type: application/json' -d '{"commands":[]}'`
- Adjust allocation awareness settings for multi-AZ deployments

## Common Issues

- symptoms: "RED cluster after node replacement"
  diagnosis: "Primary shards were on the failed node and replicas are being promoted."
  resolution: "Wait for shard recovery to complete. Monitor _cat/recovery. Cluster should return to YELLOW then GREEN."

- symptoms: "RED cluster with disk watermark breach"
  diagnosis: "Flood stage watermark (95%) reached, indices set to read-only, new primaries cannot be allocated."
  resolution: "Delete old indices or increase storage. Remove read-only block after freeing space."

- symptoms: "RED cluster after index creation with too many shards"
  diagnosis: "Not enough nodes to allocate all primary shards."
  resolution: "Reduce primary shard count or add data nodes. Delete and recreate the index with fewer shards."

## Output Format

```yaml
root_cause: "red_cluster — <specific_cause>"
evidence:
  - type: cluster_health
    content: "<_cluster/health output>"
  - type: unassigned_shards
    content: "<_cat/shards showing unassigned primaries>"
  - type: allocation_explain
    content: "<_cluster/allocation/explain output>"
severity: CRITICAL
mitigation:
  immediate: "Resolve unassigned primary shards to restore index availability"
  long_term: "Ensure adequate capacity, replicas, and multi-AZ deployment"
```


## Safety Ratings
```
safety_ratings:
  - "Check cluster health and shards: GREEN — read-only API calls"
  - "Get allocation explanation: GREEN — read-only diagnostic"
  - "Delete old indices to free space: RED — permanently removes index data"
  - "Remove allocation filters: YELLOW — changes shard placement rules"
  - "Reroute unassigned shards: YELLOW — forces shard placement, use with caution"
  - "Delete and restore index from snapshot: RED — data loss if snapshot is incomplete"
```

## Escalation Conditions
- Domain serves production search
- RED status persisting beyond 30 minutes
- Multiple primary shards unassigned across different indices
- Fix requires blue/green deployment or node scaling
- Corrupted shards requiring index deletion and snapshot restore

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Index data: search content and business data"
    - "Access policies: domain access configuration"
    - "Shard allocation details: cluster topology"
  handling: "Index data may contain sensitive search content. Do not expose index names or allocation details externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup (snapshot verification)
- NEVER suggest reducing node count below minimum for cluster health
- NEVER use allocate_stale_primary without understanding data loss implications
- NEVER ignore RED cluster status — it indicates data unavailability

## Phase 3 — Rollback
- If indices were deleted: restore from snapshot using _snapshot/<repo>/<snapshot>/_restore
- If allocation filters were removed: re-add the original allocation filter settings
- If shards were manually rerouted: allow cluster to rebalance naturally
- If nodes were added: nodes can be removed after shards are redistributed (via domain config update)

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
