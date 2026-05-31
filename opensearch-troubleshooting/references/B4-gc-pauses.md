---
title: "B4 — OpenSearch Garbage Collection Pauses"
description: "Diagnose and resolve JVM garbage collection pauses causing latency spikes"
status: active
severity: HIGH
triggers:
  - "GC pause"
  - "garbage collection"
  - "stop the world"
  - "latency spike"
  - "JVMGCOldCollectionCount"
  - "JVMGCOldCollectionTime"
owner: devops-agent
objective: "Identify the cause of excessive GC pauses and reduce their frequency and duration"
context: "OpenSearch runs on the JVM and is subject to garbage collection pauses. Young generation GC is frequent but fast (milliseconds). Old generation GC (major GC) can cause significant pauses (seconds) that block all operations. Frequent old gen GC indicates heap pressure — the JVM is struggling to free memory. Sustained old gen GC leads to cluster instability and potential node disconnection."
---

## Phase 1 — Triage

MUST:
- Check GC metrics: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name JVMGCOldCollectionCount --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check GC time: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name JVMGCOldCollectionTime --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check JVM memory pressure: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name JVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check node JVM details: `curl -XGET "https://<endpoint>/_nodes/stats/jvm?pretty"`
- Check cluster health during GC events: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`

SHOULD:
- Check field data and cache sizes: `curl -XGET "https://<endpoint>/_nodes/stats/indices/fielddata,query_cache,request_cache?pretty"`
- Check segment memory usage: `curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,segments.memory,segments.count,heap.percent"`
- Check hot threads during GC: `curl -XGET "https://<endpoint>/_nodes/hot_threads"`
- Correlate GC events with latency spikes in SearchLatency and IndexingLatency metrics

MAY:
- Check young gen GC stats for comparison
- Review index mapping complexity (nested objects increase heap usage)

## Phase 2 — Remediate

MUST:
- If old gen GC count > 5 per minute: immediate action required — scale cluster
- Clear field data cache if large: `curl -XPOST "https://<endpoint>/_cache/clear?fielddata=true"`
- Scale to larger instance type for more heap memory

SHOULD:
- Reduce shard count to lower per-shard heap overhead
- Force merge read-only indices to reduce segment count
- Optimize queries to reduce heap allocation (avoid large aggregations, deep pagination)

MAY:
- Adjust indices.fielddata.cache.size to limit field data heap usage
- Consider using doc_values exclusively for sorting and aggregations
- Review and reduce concurrent search/indexing load during GC pressure

## Common Issues

- symptoms: "Old gen GC count increasing, latency spikes every few minutes"
  diagnosis: "Heap is nearly full. Old gen GC running frequently to reclaim memory."
  resolution: "Scale to larger instance type. Reduce heap consumers (field data, shards, caches)."

- symptoms: "Node disconnects during long GC pause"
  diagnosis: "GC pause exceeds cluster fault detection timeout. Node appears dead to master."
  resolution: "Upgrade instance type for more heap. Reduce workload on affected node."

- symptoms: "GC pauses correlate with large aggregation queries"
  diagnosis: "Aggregations allocating large amounts of heap for bucket creation."
  resolution: "Limit aggregation cardinality. Use composite aggregation for high-cardinality fields."

## Output Format

```yaml
root_cause: "gc_pauses — <specific_cause>"
evidence:
  - type: gc_metrics
    content: "<JVMGCOldCollectionCount and JVMGCOldCollectionTime>"
  - type: jvm_pressure
    content: "<JVMMemoryPressure and heap breakdown>"
  - type: heap_consumers
    content: "<field data, caches, segments memory>"
severity: HIGH
mitigation:
  immediate: "Scale instance type or clear caches to reduce heap pressure"
  long_term: "Right-size cluster, optimize shard count, implement query guardrails"
```


## Safety Ratings
```
safety_ratings:
  - "Check GC metrics and JVM stats: GREEN — read-only diagnostics"
  - "Clear field data cache: YELLOW — temporary latency spike"
  - "Scale to larger instance type: YELLOW — triggers blue/green deployment"
  - "Reduce shard count: YELLOW — requires index deletion or consolidation"
  - "Force merge indices: YELLOW — resource-intensive I/O operation"
```

## Escalation Conditions
- Domain serves production search
- Old gen GC count > 5 per minute (immediate action required)
- Node disconnections during GC pauses
- Fix requires blue/green deployment
- GC pauses causing search latency SLA violations

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "GC metrics: cluster health indicators"
    - "Heap breakdown: capacity and performance data"
  handling: "Do not expose GC metrics or heap details externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER ignore frequent old gen GC — it leads to node instability
- NEVER add more load to a cluster experiencing GC pressure

## Phase 3 — Rollback
- If field data cache was cleared: cache rebuilds automatically
- If instance type was upgraded: can be downgraded via domain config update
- If indices were deleted or consolidated: restore from snapshot if needed
- If force merge was performed: non-destructive, no rollback needed

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
