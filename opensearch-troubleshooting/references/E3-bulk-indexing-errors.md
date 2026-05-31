---
title: "E3 — OpenSearch Bulk Indexing Errors"
description: "Diagnose and resolve errors during bulk indexing operations"
status: active
severity: HIGH
triggers:
  - "bulk error"
  - "bulk indexing"
  - "_bulk failed"
  - "413 payload"
  - "bulk rejected"
  - "bulk timeout"
owner: devops-agent
objective: "Identify and resolve bulk indexing errors to restore high-throughput data ingestion"
context: "The _bulk API is the recommended way to index large volumes of data. Bulk requests can fail partially (some documents succeed, others fail) or entirely. Common errors include 429 (thread pool full), 413 (payload too large), mapping conflicts, and timeout errors. The bulk response contains per-document status — always check the errors field. Optimal bulk size is 5-15 MB."
---

## Phase 1 — Triage

MUST:
- Test a small bulk request: `curl -XPOST "https://<endpoint>/_bulk" -H 'Content-Type: application/x-ndjson' -d '{"index":{"_index":"test"}}
{"field":"value"}
'`
- Check write thread pool: `curl -XGET "https://<endpoint>/_cat/thread_pool/write?v&h=node_name,active,queue,rejected"`
- Check 429 error count: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name 4xx --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check JVM memory pressure: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name JVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`

SHOULD:
- Check HTTP payload size limits: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.{InstanceType:ClusterConfig.InstanceType}'`
- Check for read-only blocks: `curl -XGET "https://<endpoint>/_all/_settings?pretty&filter_path=*.settings.index.blocks.read_only_allow_delete"`
- Check indexing rate metric: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name IndexingRate --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`

MAY:
- Check ingest pipeline errors if using pipelines
- Review bulk response for per-document error details

## Phase 2 — Remediate

MUST:
- If 429 errors: reduce bulk request rate, implement exponential backoff
- If 413 payload too large: reduce bulk request size to 5-15 MB
- If mapping errors: fix document schema or update mapping (see E2)
- If read-only block: resolve disk watermark issue (see C1)

SHOULD:
- Use optimal bulk size: 5-15 MB per request, 1000-5000 documents per batch
- Implement retry logic with exponential backoff for 429 and 5xx errors
- Use multiple parallel bulk threads (2-4 per data node)
- Increase refresh_interval during heavy bulk loading: `curl -XPUT "https://<endpoint>/<index>/_settings" -H 'Content-Type: application/json' -d '{"index":{"refresh_interval":"30s"}}'`

MAY:
- Disable replicas during initial bulk load for faster indexing
- Use index aliases to swap between old and new indices after bulk load
- Consider using OpenSearch Ingestion (OSI) pipelines for managed ingestion

## Common Issues

- symptoms: "Bulk response has errors:true with 429 status on some documents"
  diagnosis: "Write thread pool queue full. Cluster cannot keep up with bulk rate."
  resolution: "Reduce concurrent bulk requests. Add backoff. Scale cluster."

- symptoms: "413 Request Entity Too Large"
  diagnosis: "Bulk request exceeds HTTP payload limit (typically 100 MB)."
  resolution: "Reduce bulk request size to 5-15 MB. Split into smaller batches."

- symptoms: "Partial bulk failures with mapper_parsing_exception"
  diagnosis: "Some documents have fields that conflict with the index mapping."
  resolution: "Fix documents or update mapping. Check bulk response for specific failed documents."

## Output Format

```yaml
root_cause: "bulk_indexing_error — <specific_cause>"
evidence:
  - type: bulk_response
    content: "<bulk response errors and status codes>"
  - type: thread_pool
    content: "<write thread pool rejections>"
  - type: cluster_resources
    content: "<JVM, CPU, disk metrics>"
severity: HIGH
mitigation:
  immediate: "Fix bulk request size, implement backoff, or scale cluster"
  long_term: "Optimize bulk pipeline, right-size cluster for write throughput"
```


## Safety Ratings
```
safety_ratings:
  - "Check thread pool and metrics: GREEN — read-only diagnostics"
  - "Reduce bulk request rate: GREEN — client-side change"
  - "Increase refresh_interval: YELLOW — delays search visibility"
  - "Disable replicas during bulk load: RED — removes data redundancy"
  - "Scale cluster: YELLOW — triggers blue/green deployment"
```

## Escalation Conditions
- Domain serves production search
- Bulk indexing errors blocking data pipeline
- 429 errors indicating cluster capacity exhaustion
- Fix requires blue/green deployment
- Payload size limits requiring bulk request restructuring

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Bulk request payloads: business data being indexed"
    - "Error responses: may contain document field values"
    - "Thread pool stats: cluster capacity details"
  handling: "Bulk responses may contain sensitive document data. Do not expose externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER send bulk requests larger than 15 MB
- NEVER disable replicas without a plan to re-enable after bulk load

## Phase 3 — Rollback
- If refresh_interval was increased: restore to original value
- If replicas were disabled: re-enable immediately after bulk load
- If cluster was scaled: can be scaled back via domain config update
- If bulk request size was changed: revert client configuration

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
