---
title: "A2 — OpenSearch YELLOW Cluster Health"
description: "Diagnose and resolve YELLOW cluster health status caused by unassigned replica shards"
status: active
severity: MEDIUM
triggers:
  - "YELLOW cluster"
  - "cluster health yellow"
  - "unassigned replica"
  - "replica not allocated"
owner: devops-agent
objective: "Identify unassigned replica shards and restore cluster to GREEN (or confirm YELLOW is expected)"
context: "YELLOW cluster health means all primary shards are assigned but at least one replica is not. Data is available but redundancy is reduced. Single-node clusters are always YELLOW because replicas cannot be on the same node as the primary — this is expected. Multi-node clusters should be GREEN."
---

## Phase 1 — Triage

MUST:
- Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`
- Count nodes: `curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,node.role"`
- Identify unassigned replicas: `curl -XGET "https://<endpoint>/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason&s=state"`
- Get allocation explanation: `curl -XGET "https://<endpoint>/_cluster/allocation/explain?pretty" -H 'Content-Type: application/json' -d '{"index":"<index>","shard":0,"primary":false}'`
- Check domain configuration: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.{InstanceCount:ClusterConfig.InstanceCount,InstanceType:ClusterConfig.InstanceType,ZoneAwareness:ClusterConfig.ZoneAwarenessEnabled}'`

SHOULD:
- Check disk allocation: `curl -XGET "https://<endpoint>/_cat/allocation?v"`
- Check if single-node domain (YELLOW is expected): `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.ClusterConfig.InstanceCount'`
- Check for allocation filtering rules: `curl -XGET "https://<endpoint>/_cluster/settings?include_defaults=true&flat_settings=true&filter_path=*.cluster.routing.allocation.*"`

MAY:
- Check index replica settings: `curl -XGET "https://<endpoint>/<index>/_settings?pretty&filter_path=*.settings.index.number_of_replicas"`
- Review zone awareness configuration for multi-AZ

## Phase 2 — Remediate

MUST:
- If single-node cluster: YELLOW is expected — no action needed (or scale to 2+ nodes for GREEN)
- If multi-node with disk pressure: free disk space or add storage
- If insufficient nodes for replica count: reduce replicas or add nodes: `curl -XPUT "https://<endpoint>/<index>/_settings" -H 'Content-Type: application/json' -d '{"index":{"number_of_replicas":1}}'`

SHOULD:
- Enable zone awareness for multi-AZ redundancy
- Ensure replica count does not exceed (node_count - 1)
- Monitor FreeStorageSpace in CloudWatch

MAY:
- Set index.auto_expand_replicas for indices that should scale replicas with node count
- Review allocation awareness settings

## Common Issues

- symptoms: "Single-node domain is YELLOW"
  diagnosis: "Expected behavior. Replicas cannot be allocated to the same node as the primary."
  resolution: "No action needed for dev/test. Scale to 2+ nodes for production GREEN status."

- symptoms: "Multi-node cluster YELLOW after scaling down"
  diagnosis: "Fewer nodes than required for replica allocation."
  resolution: "Reduce number_of_replicas or scale back up to sufficient node count."

- symptoms: "YELLOW with disk watermark breach on some nodes"
  diagnosis: "High watermark (90%) prevents replica allocation to full nodes."
  resolution: "Free disk space, delete old indices, or add storage capacity."

## Output Format

```yaml
root_cause: "yellow_cluster — <specific_cause>"
evidence:
  - type: cluster_health
    content: "<_cluster/health output>"
  - type: node_count
    content: "<number of nodes and roles>"
  - type: unassigned_replicas
    content: "<_cat/shards showing unassigned replicas>"
severity: MEDIUM
mitigation:
  immediate: "Resolve replica allocation or confirm YELLOW is expected for single-node"
  long_term: "Right-size cluster with adequate nodes and storage for full replica allocation"
```


## Safety Ratings
```
safety_ratings:
  - "Check cluster health and nodes: GREEN — read-only API calls"
  - "Identify unassigned replicas: GREEN — read-only diagnostic"
  - "Reduce replica count: YELLOW — reduces data redundancy"
  - "Add nodes to cluster: YELLOW — domain configuration change, triggers blue/green deployment"
  - "Enable zone awareness: YELLOW — domain configuration change, triggers blue/green deployment"
```

## Escalation Conditions
- Domain serves production search
- YELLOW status on multi-node cluster (unexpected)
- Fix requires blue/green deployment
- Disk pressure causing replica allocation failures
- Zone awareness misconfiguration affecting redundancy

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Index data: search content"
    - "Node configuration: cluster topology"
    - "Replica settings: redundancy configuration"
  handling: "Do not expose cluster topology or node details externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER set replicas to 0 on production indices without understanding the durability impact
- NEVER ignore YELLOW on multi-node clusters — it indicates reduced redundancy

## Phase 3 — Rollback
- If replica count was reduced: increase replicas back to original value
- If nodes were added: can be removed via domain config update after rebalancing
- If zone awareness was enabled: can be disabled but requires domain config update
- If auto_expand_replicas was set: remove the setting to restore fixed replica count

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
