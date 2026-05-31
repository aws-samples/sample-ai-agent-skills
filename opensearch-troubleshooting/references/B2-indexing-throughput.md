---
title: "B2 — OpenSearch Indexing Throughput"
description: "Diagnose and resolve indexing throughput degradation and slow indexing performance"
status: active
severity: HIGH
triggers:
  - "indexing slow"
  - "indexing throughput"
  - "IndexingRate"
  - "write rejected"
  - "429 error"
  - "bulk rejected"
owner: devops-agent
objective: "Identify bottlenecks in indexing throughput and restore optimal indexing performance"
context: "Indexing throughput depends on bulk request size, shard count, refresh interval, merge operations, JVM memory, disk I/O, and cluster resources. Throttled indexing (429 errors) indicates the cluster cannot keep up with the write rate. Common bottlenecks include undersized instances, too many shards, aggressive refresh intervals, and JVM memory pressure."
---

## Phase 1 — Triage

MUST:
- Check indexing rate: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name IndexingRate --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check 429 errors: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name 4xx --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check thread pool rejections: `curl -XGET "https://<endpoint>/_cat/thread_pool/write?v&h=node_name,active,queue,rejected,completed"`
- Check JVM memory pressure: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name JVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name CPUUtilization --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

SHOULD:
- Check disk I/O: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name WriteIOPS --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check index refresh interval: `curl -XGET "https://<endpoint>/<index>/_settings?pretty&filter_path=*.settings.index.refresh_interval"`
- Check merge activity: `curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,merges.current,merges.total"`
- Check bulk queue size: `curl -XGET "https://<endpoint>/_nodes/stats/thread_pool/write?pretty"`

MAY:
- Check indexing slow log: `curl -XGET "https://<endpoint>/<index>/_settings?pretty&filter_path=*.settings.index.indexing.slowlog"`
- Check segment count per index: `curl -XGET "https://<endpoint>/_cat/segments/<index>?v&s=size:desc" | head -20`

## Phase 2 — Remediate

MUST:
- Use _bulk API instead of individual document indexing (5-15 MB per request)
- If 429 errors: reduce indexing rate, increase cluster size, or add nodes
- If JVM pressure > 80%: scale cluster or reduce concurrent indexing

SHOULD:
- Increase refresh_interval during heavy indexing: `curl -XPUT "https://<endpoint>/<index>/_settings" -H 'Content-Type: application/json' -d '{"index":{"refresh_interval":"30s"}}'`
- Use multiple bulk indexing threads (2-4 per node)
- Disable replicas during initial bulk load, re-enable after: `curl -XPUT "https://<endpoint>/<index>/_settings" -H 'Content-Type: application/json' -d '{"index":{"number_of_replicas":0}}'`

MAY:
- Optimize mappings: disable _source for write-heavy indices if not needed for reindex
- Use index templates to set optimal settings for new indices
- Consider time-based indices with rollover for append-only workloads

## Common Issues

- symptoms: "429 Too Many Requests during bulk indexing"
  diagnosis: "Write thread pool queue is full. Cluster cannot keep up with write rate."
  resolution: "Reduce bulk request rate, add data nodes, or upgrade instance types."

- symptoms: "Indexing throughput drops over time"
  diagnosis: "Segment merges consuming I/O and CPU. Too many small segments."
  resolution: "Increase refresh_interval. Force merge during off-peak. Check disk I/O limits."

- symptoms: "Individual document indexing is very slow"
  diagnosis: "Per-document overhead (HTTP, routing, refresh) dominates."
  resolution: "Switch to _bulk API with 5-15 MB batches. Never index documents one at a time."

## Output Format

```yaml
root_cause: "indexing_throughput — <specific_cause>"
evidence:
  - type: indexing_metrics
    content: "<IndexingRate and 429 error count>"
  - type: thread_pool
    content: "<write thread pool rejections>"
  - type: resource_usage
    content: "<CPU, JVM, disk I/O metrics>"
severity: HIGH
mitigation:
  immediate: "Reduce indexing rate or scale cluster to handle write load"
  long_term: "Optimize bulk indexing, right-size cluster, implement index lifecycle"
```


## Safety Ratings
```
safety_ratings:
  - "Check indexing metrics and thread pool: GREEN — read-only diagnostics"
  - "Increase refresh_interval: YELLOW — delays search visibility of new documents"
  - "Disable replicas during bulk load: RED — removes data redundancy temporarily"
  - "Scale cluster: YELLOW — triggers blue/green deployment"
  - "Reduce indexing rate: GREEN — client-side change"
```

## Escalation Conditions
- Domain serves production search
- 429 errors blocking data ingestion pipeline
- Fix requires blue/green deployment
- JVM memory pressure above 80% during indexing
- Write thread pool rejections increasing

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Indexing metrics: data ingestion volume patterns"
    - "Index data: business content being indexed"
    - "Thread pool stats: cluster capacity details"
  handling: "Do not expose indexing volume patterns or thread pool details externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER disable replicas on production indices without a plan to re-enable
- NEVER index documents one at a time — always use _bulk API

## Phase 3 — Rollback
- If refresh_interval was increased: restore to original value (default 1s)
- If replicas were disabled: re-enable replicas immediately after bulk load completes
- If cluster was scaled: can be scaled back via domain config update
- If indexing rate was reduced: restore original ingestion rate

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
