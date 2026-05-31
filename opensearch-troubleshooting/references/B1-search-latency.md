---
title: "B1 — OpenSearch Search Latency"
description: "Diagnose and resolve high search latency and slow query performance"
status: active
severity: HIGH
triggers:
  - "search latency"
  - "slow search"
  - "slow query"
  - "SearchLatency"
  - "search timeout"
  - "query performance"
owner: devops-agent
objective: "Identify the cause of high search latency and optimize query performance"
context: "Search latency in OpenSearch depends on query complexity, index size, shard count, field data usage, JVM memory pressure, and cluster resource utilization. High latency can be caused by expensive queries (wildcards, deep aggregations, large result sets), insufficient resources, too many shards, or JVM GC pressure. Slow logs help identify specific slow queries."
---

## Phase 1 — Triage

MUST:
- Check search latency metric: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name SearchLatency --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Average,p99`
- Check search rate: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name SearchRate --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name CPUUtilization --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Check JVM memory pressure: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name JVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check hot threads: `curl -XGET "https://<endpoint>/_nodes/hot_threads"`

SHOULD:
- Check node stats for search thread pool: `curl -XGET "https://<endpoint>/_nodes/stats/thread_pool/search?pretty"`
- Check slow log settings: `curl -XGET "https://<endpoint>/<index>/_settings?pretty&filter_path=*.settings.index.search.slowlog"`
- Check index stats: `curl -XGET "https://<endpoint>/<index>/_stats/search?pretty"`
- Check field data usage: `curl -XGET "https://<endpoint>/_cat/fielddata?v&s=size:desc"`

MAY:
- Profile a specific query: `curl -XGET "https://<endpoint>/<index>/_search" -H 'Content-Type: application/json' -d '{"profile":true,"query":{"match":{"field":"value"}}}'`
- Check segment count: `curl -XGET "https://<endpoint>/_cat/segments/<index>?v&s=size:desc"`
- Check cache stats: `curl -XGET "https://<endpoint>/_nodes/stats/indices/query_cache,request_cache?pretty"`

## Phase 2 — Remediate

MUST:
- If JVM pressure > 80%: scale cluster or reduce field data cache
- If CPU > 80%: add data nodes or upgrade instance type
- If slow queries identified: optimize query structure (avoid leading wildcards, deep pagination, large aggregations)

SHOULD:
- Enable slow logs to identify problematic queries: `curl -XPUT "https://<endpoint>/<index>/_settings" -H 'Content-Type: application/json' -d '{"index.search.slowlog.threshold.query.warn":"5s","index.search.slowlog.threshold.query.info":"2s","index.search.slowlog.threshold.fetch.warn":"1s"}'`
- Force merge read-only indices to reduce segment count: `curl -XPOST "https://<endpoint>/<index>/_forcemerge?max_num_segments=1"`
- Use doc_values instead of fielddata for sorting/aggregations on text fields

MAY:
- Implement search result caching with request_cache
- Use index sorting for frequently filtered fields
- Consider index aliases with filtered routing for large indices

## Common Issues

- symptoms: "Search latency spikes during indexing"
  diagnosis: "Indexing and search compete for resources. Refresh operations cause segment creation."
  resolution: "Increase refresh_interval during heavy indexing. Scale cluster to separate workloads."

- symptoms: "Wildcard queries taking seconds"
  diagnosis: "Leading wildcard queries (e.g., *term) scan all terms in the index."
  resolution: "Use ngram tokenizer or reverse field for suffix matching. Avoid leading wildcards."

- symptoms: "Deep pagination (from: 10000) is slow"
  diagnosis: "OpenSearch must fetch and sort all documents up to from+size across all shards."
  resolution: "Use search_after or scroll API for deep pagination. Avoid large from values."

## Output Format

```yaml
root_cause: "search_latency — <specific_cause>"
evidence:
  - type: latency_metrics
    content: "<SearchLatency and SearchRate>"
  - type: resource_usage
    content: "<CPU, JVM, thread pool stats>"
  - type: slow_queries
    content: "<slow log entries or profiled queries>"
severity: HIGH
mitigation:
  immediate: "Optimize slow queries or scale cluster resources"
  long_term: "Implement slow log monitoring, optimize index mappings, right-size shards"
```


## Safety Ratings
```
safety_ratings:
  - "Check latency metrics and hot threads: GREEN — read-only diagnostics"
  - "Enable slow logs: GREEN — adds logging without affecting queries"
  - "Force merge read-only indices: YELLOW — resource-intensive I/O operation"
  - "Scale cluster (add nodes/upgrade type): YELLOW — triggers blue/green deployment"
  - "Clear field data cache: YELLOW — may cause temporary latency spike as cache rebuilds"
```

## Escalation Conditions
- Domain serves production search
- Search latency exceeding SLA thresholds
- Fix requires blue/green deployment
- JVM memory pressure above 80% causing GC-related latency
- Slow queries from critical application paths

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Slow log entries: query text with search terms"
    - "Search results: business data"
    - "Field data usage: index structure details"
  handling: "Slow logs may contain user search queries. Restrict access to slow log data."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER force merge indices that are actively being written to
- NEVER enable fielddata on text fields in production without understanding memory impact

## Phase 3 — Rollback
- If slow logs were enabled: disable slow log settings on the index
- If field data cache was cleared: cache rebuilds automatically — no rollback needed
- If force merge was performed: cannot be undone, but is non-destructive
- If cluster was scaled: can be scaled back via domain config update

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
