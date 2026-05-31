---
title: "E1 — OpenSearch Indexing Failures"
description: "Diagnose and resolve document indexing failures and rejected write requests"
status: active
severity: HIGH
triggers:
  - "indexing failed"
  - "index error"
  - "document rejected"
  - "write failed"
  - "403 write"
  - "400 bad request"
owner: devops-agent
objective: "Identify the cause of indexing failures and restore write operations"
context: "Indexing failures can be caused by read-only indices (flood stage watermark), mapping conflicts, access policy denials, FGAC permission issues, malformed documents, field limit exceeded, or cluster resource exhaustion (429 errors). The error response from OpenSearch contains the specific reason. Check both HTTP status code and response body."
---

## Phase 1 — Triage

MUST:
- Test indexing with a simple document: `curl -XPOST "https://<endpoint>/<index>/_doc" -H 'Content-Type: application/json' -d '{"test":"value"}'`
- Check for read-only blocks: `curl -XGET "https://<endpoint>/<index>/_settings?pretty&filter_path=*.settings.index.blocks"`
- Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`
- Check disk space: `curl -XGET "https://<endpoint>/_cat/allocation?v"`
- Check write thread pool: `curl -XGET "https://<endpoint>/_cat/thread_pool/write?v&h=node_name,active,queue,rejected"`

SHOULD:
- Check index mapping: `curl -XGET "https://<endpoint>/<index>/_mapping?pretty"`
- Check access policy: `aws opensearch describe-domain-config --domain-name <domain> --query 'DomainConfig.AccessPolicies'`
- Check 4xx and 5xx error metrics: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name 4xx --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`

MAY:
- Check field limit: `curl -XGET "https://<endpoint>/<index>/_settings?pretty&filter_path=*.settings.index.mapping.total_fields.limit"`
- Check indexing slow log for patterns
- Review CloudTrail for access policy changes

## Phase 2 — Remediate

MUST:
- If read-only block: free disk space and remove block: `curl -XPUT "https://<endpoint>/<index>/_settings" -H 'Content-Type: application/json' -d '{"index.blocks.read_only_allow_delete": null}'`
- If mapping conflict: fix document to match mapping or update mapping
- If access denied: update access policy or FGAC permissions
- If 429 errors: reduce indexing rate or scale cluster

SHOULD:
- Increase field limit if legitimately needed: `curl -XPUT "https://<endpoint>/<index>/_settings" -H 'Content-Type: application/json' -d '{"index.mapping.total_fields.limit": 2000}'`
- Use dynamic templates to control mapping for new fields
- Implement retry logic with exponential backoff for transient failures

MAY:
- Use ingest pipelines to validate and transform documents before indexing
- Set up dead letter queues for failed documents

## Common Issues

- symptoms: "403 Forbidden on index request"
  diagnosis: "Access policy or FGAC denying write access."
  resolution: "Check domain access policy and FGAC role mappings. Ensure IAM role has write permissions."

- symptoms: "400 Bad Request with mapper_parsing_exception"
  diagnosis: "Document field type conflicts with existing mapping."
  resolution: "Fix document to match mapping. See E2 for mapping conflict resolution."

- symptoms: "429 Too Many Requests"
  diagnosis: "Write thread pool queue full. Cluster overwhelmed."
  resolution: "Reduce indexing rate. Scale cluster. Use bulk API with backoff."

## Output Format

```yaml
root_cause: "indexing_failure — <specific_cause>"
evidence:
  - type: error_response
    content: "<HTTP status and error message>"
  - type: index_settings
    content: "<index blocks and mapping>"
  - type: cluster_resources
    content: "<disk space, thread pool, JVM>"
severity: HIGH
mitigation:
  immediate: "Fix the specific indexing error (block, mapping, access, capacity)"
  long_term: "Implement monitoring, proper mappings, and capacity planning"
```


## Safety Ratings
```
safety_ratings:
  - "Test indexing and check blocks: GREEN — read-only diagnostics"
  - "Check thread pool and metrics: GREEN — read-only monitoring"
  - "Remove read-only block: YELLOW — re-enables writes"
  - "Update access policy: YELLOW — changes domain access"
  - "Increase field limit: YELLOW — allows more fields, increases mapping complexity"
```

## Escalation Conditions
- Domain serves production search
- Indexing failures blocking data ingestion pipeline
- Read-only blocks from disk watermark breach
- Access policy changes needed affecting multiple services
- Mapping conflicts requiring reindex

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Index data: business content being indexed"
    - "Error responses: may contain document field values"
    - "Access policies: domain access configuration"
  handling: "Error responses may contain sensitive document data. Do not expose externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER remove read-only blocks without freeing disk space first
- NEVER increase field limit without understanding the JVM memory impact

## Phase 3 — Rollback
- If read-only block was removed: block re-applies automatically if disk fills again
- If access policy was updated: restore previous policy
- If field limit was increased: reduce back to previous value (existing fields remain)
- If mapping was updated: cannot change field types — reindex required

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
