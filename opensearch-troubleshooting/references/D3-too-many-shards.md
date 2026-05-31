---
title: "D3 — OpenSearch Too Many Shards"
description: "Diagnose and resolve excessive shard count causing cluster instability"
status: active
severity: HIGH
triggers:
  - "too many shards"
  - "shard limit"
  - "shard overhead"
  - "cluster state too large"
  - "max shards per node"
owner: devops-agent
objective: "Reduce shard count to sustainable levels and prevent future shard proliferation"
context: "Each shard consumes approximately 25 MB of heap memory, file handles, and cluster state overhead. Too many shards cause master node instability, slow cluster state updates, high JVM memory pressure, and degraded performance. AWS recommends max 1000 shards per node and 10-50 GB per shard. Common causes: too many small indices, excessive primary shard count, or missing index lifecycle management."
---

## Phase 1 — Triage

MUST:
- Check total shard count: `curl -XGET "https://<endpoint>/_cluster/health?pretty" | grep -E "active_primary_shards|active_shards|number_of_nodes"`
- Calculate shards per node: `curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,shards"`
- Check small indices: `curl -XGET "https://<endpoint>/_cat/indices?v&h=index,pri,rep,store.size,docs.count&s=store.size" | head -30`
- Check JVM memory pressure: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name JVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check master node pressure: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name MasterJVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

SHOULD:
- Check index templates for shard settings: `curl -XGET "https://<endpoint>/_index_template?pretty"`
- Check ISM policies: `curl -XGET "https://<endpoint>/_plugins/_ism/policies?pretty"`
- Identify indices with excessive primary shards for their size

MAY:
- Check cluster state size: `curl -XGET "https://<endpoint>/_cluster/state?pretty" | wc -c`
- Review index creation patterns (daily indices with too many shards)

## Phase 2 — Remediate

MUST:
- Delete old/unnecessary indices to reduce shard count
- Reindex small indices into consolidated indices with fewer shards
- Update index templates to use appropriate shard counts: `curl -XPUT "https://<endpoint>/_index_template/my-template" -H 'Content-Type: application/json' -d '{"index_patterns":["my-index-*"],"template":{"settings":{"number_of_shards":1,"number_of_replicas":1}}}'`

SHOULD:
- Implement ISM policies with rollover and delete actions
- Use index rollover with size-based conditions instead of time-based with fixed shards
- Reduce replica count on non-critical indices

MAY:
- Use shrink API to reduce primary shard count on existing indices: `curl -XPOST "https://<endpoint>/<index>/_shrink/<target-index>" -H 'Content-Type: application/json' -d '{"settings":{"index.number_of_shards":1}}'`
- Consider data streams for time-series data with automatic rollover
- Add nodes to increase total shard capacity

## Common Issues

- symptoms: "Thousands of small indices with 5 primary shards each"
  diagnosis: "Default shard count too high for small indices. Daily indices accumulating."
  resolution: "Set primary shards to 1 for small indices. Implement ISM delete policy."

- symptoms: "Master node JVM pressure high with many shards"
  diagnosis: "Cluster state too large. Each shard adds to master heap overhead."
  resolution: "Reduce total shard count. Delete old indices. Consolidate small indices."

- symptoms: "Index creation failing with shard limit error"
  diagnosis: "Cluster-level shard limit reached (cluster.max_shards_per_node)."
  resolution: "Delete old indices to free shard slots. Increase limit if cluster can handle it."

## Output Format

```yaml
root_cause: "too_many_shards — <specific_cause>"
evidence:
  - type: shard_count
    content: "<total shards, shards per node>"
  - type: small_indices
    content: "<indices with disproportionate shard count>"
  - type: jvm_pressure
    content: "<JVM and master JVM memory pressure>"
severity: HIGH
mitigation:
  immediate: "Delete old indices and consolidate small indices"
  long_term: "Implement ISM lifecycle, right-size shard counts in templates"
```


## Safety Ratings
```
safety_ratings:
  - "Check shard count and small indices: GREEN — read-only diagnostics"
  - "Delete old indices: RED — permanently removes data"
  - "Update index templates: GREEN — affects only new indices"
  - "Reindex into consolidated indices: YELLOW — resource-intensive, creates new indices"
  - "Shrink index: YELLOW — creates new index with fewer shards"
```

## Escalation Conditions
- Domain serves production search
- Master node instability due to large cluster state
- Shard limit reached blocking new index creation
- Fix requires significant index cleanup or consolidation
- JVM memory pressure from shard overhead

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Index names and shard counts: data structure"
    - "Cluster state size: infrastructure details"
    - "Index templates: configuration patterns"
  handling: "Do not expose index naming patterns or shard configuration externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER increase cluster.max_shards_per_node without addressing the root cause
- NEVER create new indices with excessive primary shards

## Phase 3 — Rollback
- If indices were deleted: restore from snapshot
- If index templates were updated: revert to previous template settings
- If indices were reindexed/consolidated: delete consolidated index and restore originals from snapshot
- If shrink was performed: delete shrunk index and restore original from snapshot

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
