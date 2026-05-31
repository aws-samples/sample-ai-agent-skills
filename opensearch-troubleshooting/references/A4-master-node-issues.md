---
title: "A4 — OpenSearch Master Node Issues"
description: "Diagnose dedicated master node instability, overload, or failure"
status: active
severity: HIGH
triggers:
  - "master node"
  - "master not reachable"
  - "cluster unstable"
  - "master overloaded"
  - "MasterCPUUtilization"
  - "MasterJVMMemoryPressure"
owner: devops-agent
objective: "Identify and resolve master node issues to restore cluster stability"
context: "Dedicated master nodes manage cluster state, shard allocation, index creation/deletion, and mapping updates. When master nodes are overloaded or unavailable, the entire cluster becomes unstable — new indices cannot be created, shards cannot be allocated, and cluster state updates stall. Master node issues often manifest as slow index creation, delayed shard allocation, or cluster health not updating."
---

## Phase 1 — Triage

MUST:
- Check master node identity: `curl -XGET "https://<endpoint>/_cat/master?v"`
- Check master node metrics: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name MasterCPUUtilization --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Check master JVM pressure: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name MasterJVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check master reachability: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name MasterReachableFromNode --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 60 --statistics Minimum`
- Check domain master config: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.ClusterConfig.{DedicatedMasterEnabled:DedicatedMasterEnabled,DedicatedMasterCount:DedicatedMasterCount,DedicatedMasterType:DedicatedMasterType}'`

SHOULD:
- Check pending cluster tasks: `curl -XGET "https://<endpoint>/_cat/pending_tasks?v"`
- Check total shard count (affects master heap): `curl -XGET "https://<endpoint>/_cluster/health?pretty" | grep -E "active_primary_shards|active_shards"`
- Check total index count: `curl -XGET "https://<endpoint>/_cat/indices?v" | wc -l`
- Check node stats for master: `curl -XGET "https://<endpoint>/_nodes/stats/jvm,os?pretty"`

MAY:
- Check hot threads on master: `curl -XGET "https://<endpoint>/_nodes/hot_threads"`
- Review cluster state size: `curl -XGET "https://<endpoint>/_cluster/state?pretty" | wc -c`

## Phase 2 — Remediate

MUST:
- If master CPU > 80%: upgrade master instance type
- If master JVM pressure > 80%: upgrade to larger master instance type or reduce cluster state size (fewer indices/shards)
- If no dedicated masters: enable 3 dedicated master nodes
- If master instance type too small for shard count: upgrade (c6g.large for < 100K shards, c6g.xlarge for larger)

SHOULD:
- Reduce total shard count by consolidating small indices or reducing replicas
- Delete unused indices to reduce cluster state
- Use ISM policies to manage index lifecycle and prevent index accumulation

MAY:
- Consider index templates with fewer primary shards for new indices
- Review and optimize mapping complexity (deeply nested mappings increase cluster state)

## Common Issues

- symptoms: "Slow index creation and shard allocation"
  diagnosis: "Master node overloaded with too many pending tasks."
  resolution: "Upgrade master instance type. Reduce cluster state size (fewer indices/shards)."

- symptoms: "MasterJVMMemoryPressure consistently above 85%"
  diagnosis: "Cluster state too large for master heap. Too many indices or shards."
  resolution: "Upgrade master instance type for more heap. Delete old indices. Reduce shard count."

- symptoms: "MasterReachableFromNode intermittently drops to 0"
  diagnosis: "Master node GC pauses or network issues causing temporary unreachability."
  resolution: "Upgrade master instance type. Check for large cluster state operations."

## Output Format

```yaml
root_cause: "master_node_issue — <specific_cause>"
evidence:
  - type: master_metrics
    content: "<MasterCPUUtilization and MasterJVMMemoryPressure>"
  - type: master_config
    content: "<dedicated master configuration>"
  - type: cluster_state
    content: "<shard count, index count, pending tasks>"
severity: HIGH
mitigation:
  immediate: "Upgrade master instance type or reduce cluster state size"
  long_term: "Right-size master nodes, implement index lifecycle management"
```


## Safety Ratings
```
safety_ratings:
  - "Check master metrics and configuration: GREEN — read-only API calls"
  - "Check pending tasks and shard count: GREEN — read-only diagnostics"
  - "Upgrade master instance type: YELLOW — triggers blue/green deployment"
  - "Delete old indices to reduce cluster state: RED — permanently removes data"
  - "Reduce shard count: YELLOW — requires index deletion or consolidation"
```

## Escalation Conditions
- Domain serves production search
- MasterJVMMemoryPressure consistently above 85%
- MasterReachableFromNode intermittently dropping to 0
- Fix requires blue/green deployment for master instance type change
- Cluster state too large requiring significant index cleanup

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Master node metrics: cluster health indicators"
    - "Shard and index counts: cluster capacity data"
    - "Cluster state size: infrastructure details"
  handling: "Do not expose master node metrics or cluster state details externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER downgrade master instance type when JVM pressure is already high
- NEVER ignore sustained MasterJVMMemoryPressure above 85%

## Phase 3 — Rollback
- If master instance type was upgraded: can be downgraded via domain config update (triggers blue/green)
- If indices were deleted to reduce cluster state: restore from snapshot if needed
- If ISM policies were added: remove policies if they cause unintended index operations

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
