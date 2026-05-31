---
title: "B3 — OpenSearch JVM Memory Pressure"
description: "Diagnose and resolve high JVM memory pressure causing performance degradation"
status: active
severity: HIGH
triggers:
  - "JVM memory"
  - "JVMMemoryPressure"
  - "heap pressure"
  - "out of memory"
  - "OOM"
  - "circuit breaker"
owner: devops-agent
objective: "Identify the cause of high JVM memory pressure and reduce it to safe levels (< 80%)"
context: "OpenSearch uses JVM heap for field data caches, query caches, request caches, indexing buffers, and cluster state. JVM heap is automatically set to half the instance RAM (max ~32 GB). When JVMMemoryPressure exceeds 80%, GC becomes aggressive causing latency. Above 92%, circuit breakers trip and reject requests. Sustained pressure above 85% requires immediate action."
---

## Phase 1 — Triage

MUST:
- Check JVM memory pressure: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name JVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check old gen JVM pressure: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name OldGenJVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check node JVM stats: `curl -XGET "https://<endpoint>/_nodes/stats/jvm?pretty"`
- Check field data usage: `curl -XGET "https://<endpoint>/_cat/fielddata?v&s=size:desc"`
- Check circuit breaker stats: `curl -XGET "https://<endpoint>/_nodes/stats/breaker?pretty"`

SHOULD:
- Check total shard count: `curl -XGET "https://<endpoint>/_cluster/health?pretty" | grep -E "active_primary_shards|active_shards"`
- Check cache sizes: `curl -XGET "https://<endpoint>/_nodes/stats/indices/query_cache,request_cache,fielddata?pretty"`
- Check segment memory: `curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,segments.memory,segments.count,heap.percent"`
- Check instance type and heap size: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.ClusterConfig.{InstanceType:InstanceType,InstanceCount:InstanceCount}'`

MAY:
- Check for large aggregations in slow logs
- Check indexing buffer size: `curl -XGET "https://<endpoint>/_cluster/settings?include_defaults=true&flat_settings=true&filter_path=*.indices.memory.index_buffer_size"`

## Phase 2 — Remediate

MUST:
- If field data is large: clear field data cache: `curl -XPOST "https://<endpoint>/_cache/clear?fielddata=true"`
- If too many shards: reduce shard count by deleting old indices or consolidating
- If sustained > 85%: scale to larger instance type or add nodes

SHOULD:
- Use doc_values (default) instead of fielddata for sorting/aggregations
- Reduce number of shards per node (each shard uses ~25 MB heap)
- Force merge read-only indices to reduce segment count and memory

MAY:
- Adjust circuit breaker settings if requests are being rejected prematurely
- Reduce query cache size if cache is consuming excessive heap
- Consider moving to instances with more RAM for larger heap

## Common Issues

- symptoms: "JVMMemoryPressure consistently above 85%"
  diagnosis: "Heap is too small for the workload. Too many shards, large field data, or heavy queries."
  resolution: "Scale to larger instance type. Reduce shard count. Clear field data cache."

- symptoms: "Circuit breaker tripping with 429 errors"
  diagnosis: "A single request would exceed the circuit breaker limit (typically 95% of heap)."
  resolution: "Reduce query complexity. Break large aggregations into smaller queries. Scale cluster."

- symptoms: "JVM pressure spikes during bulk indexing"
  diagnosis: "Indexing buffer and segment creation consuming heap."
  resolution: "Reduce bulk request size. Increase refresh_interval. Add nodes to distribute load."

## Output Format

```yaml
root_cause: "jvm_memory_pressure — <specific_cause>"
evidence:
  - type: jvm_metrics
    content: "<JVMMemoryPressure and OldGenJVMMemoryPressure>"
  - type: heap_usage
    content: "<node JVM stats and field data>"
  - type: shard_count
    content: "<total shard count and per-node distribution>"
severity: HIGH
mitigation:
  immediate: "Clear caches, reduce shard count, or scale instance type"
  long_term: "Right-size cluster, optimize shard strategy, monitor JVM metrics"
```


## Safety Ratings
```
safety_ratings:
  - "Check JVM metrics and caches: GREEN — read-only diagnostics"
  - "Clear field data cache: YELLOW — temporary latency spike as cache rebuilds"
  - "Delete old indices: RED — permanently removes data"
  - "Scale to larger instance type: YELLOW — triggers blue/green deployment"
  - "Force merge read-only indices: YELLOW — resource-intensive I/O operation"
```

## Escalation Conditions
- Domain serves production search
- JVMMemoryPressure sustained above 85%
- Circuit breakers tripping causing request rejections
- Fix requires blue/green deployment for instance type change
- Too many shards requiring significant index cleanup

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "JVM heap breakdown: cluster capacity details"
    - "Field data and cache sizes: index structure information"
    - "Shard count: cluster topology"
  handling: "Do not expose JVM metrics or shard topology externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER ignore JVMMemoryPressure above 92% — circuit breakers will trip
- NEVER increase field data cache size when JVM pressure is already high

## Phase 3 — Rollback
- If field data cache was cleared: cache rebuilds automatically — no rollback needed
- If indices were deleted: restore from snapshot if needed
- If instance type was upgraded: can be downgraded via domain config update
- If circuit breaker settings were adjusted: restore default values

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
