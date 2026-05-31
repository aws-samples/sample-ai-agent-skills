---
title: "I2 — OpenSearch Serverless Capacity and Scaling"
description: "Diagnose and resolve OpenSearch Serverless capacity limits and scaling issues"
status: active
severity: MEDIUM
triggers:
  - "Serverless capacity"
  - "OCU"
  - "Serverless scaling"
  - "Serverless limits"
  - "Serverless throttling"
  - "Serverless performance"
owner: devops-agent
objective: "Resolve Serverless capacity issues and optimize scaling configuration"
context: "OpenSearch Serverless uses OCUs (OpenSearch Compute Units) for capacity. Each OCU provides a combination of compute, memory, and storage. Serverless automatically scales OCUs based on workload. Account-level OCU limits apply. Indexing and search have separate OCU allocations. Minimum is 2 OCUs for indexing and 2 for search per collection. Capacity limits can cause throttling (429 errors)."
---

## Phase 1 — Triage

MUST:
- Check account capacity limits: `aws opensearchserverless get-account-settings`
- Check collection details: `aws opensearchserverless batch-get-collection --names <collection-name>`
- Check CloudWatch metrics for OCU usage: `aws cloudwatch get-metric-statistics --namespace AWS/AOSS --metric-name SearchOCU --dimensions Name=CollectionName,Value=<collection> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check indexing OCU: `aws cloudwatch get-metric-statistics --namespace AWS/AOSS --metric-name IndexingOCU --dimensions Name=CollectionName,Value=<collection> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check for throttling: `aws cloudwatch get-metric-statistics --namespace AWS/AOSS --metric-name 4xx --dimensions Name=CollectionName,Value=<collection> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check total OCU usage across all collections
- Check ingestion rate and search rate metrics
- Check storage utilization: `aws cloudwatch get-metric-statistics --namespace AWS/AOSS --metric-name StorageUsedInBytes --dimensions Name=CollectionName,Value=<collection> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

MAY:
- Compare OCU usage patterns over time to identify scaling trends
- Check if collection type (search, time-series, vector) is optimal for workload

## Phase 2 — Remediate

MUST:
- If hitting account OCU limits: request limit increase via AWS Support
- If throttling: reduce request rate or wait for auto-scaling to add OCUs
- If storage limit reached: delete old data or request limit increase

SHOULD:
- Set appropriate max OCU capacity for the account: `aws opensearchserverless update-account-settings --capacity-limits '{"maxIndexingCapacityInOCU":<max>,"maxSearchCapacityInOCU":<max>}'`
- Optimize queries to reduce search OCU consumption
- Use bulk indexing to optimize indexing OCU usage

MAY:
- Consider splitting workloads across multiple collections
- Use time-series collection type for log/event data (optimized for append-only)
- Implement client-side request throttling with backoff

## Common Issues

- symptoms: "429 errors during peak indexing"
  diagnosis: "Indexing OCU limit reached. Auto-scaling cannot add more OCUs."
  resolution: "Increase account OCU limits. Reduce indexing rate. Use bulk API."

- symptoms: "Search latency increasing over time"
  diagnosis: "Search OCU allocation not scaling fast enough for growing data."
  resolution: "Check OCU limits. Optimize queries. Consider data lifecycle management."

- symptoms: "Collection at minimum OCU but costs too high"
  diagnosis: "Minimum 2 indexing + 2 search OCUs per collection even when idle."
  resolution: "Consolidate collections if possible. Serverless has a minimum cost floor."

## Output Format

```yaml
root_cause: "serverless_capacity — <specific_cause>"
evidence:
  - type: ocu_usage
    content: "<SearchOCU and IndexingOCU metrics>"
  - type: account_limits
    content: "<account OCU limits>"
  - type: throttling
    content: "<4xx error count>"
severity: MEDIUM
mitigation:
  immediate: "Increase OCU limits or reduce request rate"
  long_term: "Optimize workload, implement lifecycle management, right-size capacity"
```


## Safety Ratings
```
safety_ratings:
  - "Check OCU usage and account limits: GREEN — read-only API calls"
  - "Check throttling metrics: GREEN — read-only monitoring"
  - "Update account OCU limits: YELLOW — increases potential cost"
  - "Reduce request rate: GREEN — client-side change"
  - "Request limit increase: GREEN — no immediate infrastructure change"
```

## Escalation Conditions
- Domain serves production search
- Throttling (429 errors) blocking data ingestion or search
- Account OCU limits reached
- Storage limits approaching
- Cost concerns from OCU usage

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "OCU usage metrics: capacity and cost data"
    - "Account limits: infrastructure configuration"
    - "Collection metrics: workload patterns"
  handling: "Do not expose OCU usage or cost details externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER set OCU limits to maximum without cost approval
- NEVER ignore sustained throttling — it indicates capacity issues

## Phase 3 — Rollback
- If OCU limits were increased: reduce back to previous limits
- If request rate was reduced: restore original rate after capacity is available
- If collections were consolidated: split back if needed (requires new collection creation)

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
