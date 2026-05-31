---
title: "B3 — Rate Limiting"
description: "Diagnose and resolve AMP ingestion rate limiting and throttling"
status: active
severity: HIGH
triggers:
  - "rate limit"
  - "throttling"
  - "429"
  - "Too Many Requests"
  - "ingestion limit"
  - "samples per second"
owner: devops-agent
objective: "Identify the cause of rate limiting and restore normal ingestion throughput"
context: "AMP workspaces have per-workspace ingestion rate limits (samples per second) and active series limits. Exceeding either causes HTTP 429 responses on the remote write endpoint. Rate limits are soft limits that can be increased via Service Quotas."
---

## Phase 1 — Triage

MUST:
- Check ingestion rate: `aws cloudwatch get-metric-statistics --namespace AWS/Prometheus --metric-name IngestionRate --dimensions Name=WorkspaceId,Value=<workspace-id> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Maximum`
- Check active series count: `aws cloudwatch get-metric-statistics --namespace AWS/Prometheus --metric-name ActiveSeries --dimensions Name=WorkspaceId,Value=<workspace-id> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Maximum`
- Check current service quotas: `aws service-quotas get-service-quota --service-code aps --quota-code L-6B340E60`
- Check collector logs for HTTP 429 responses

SHOULD:
- Compare current ingestion rate against the quota limit
- Check if rate limiting is sustained or bursty
- Identify which collectors are contributing most to ingestion volume
- Check if new scrape targets were recently added

MAY:
- Check CloudWatch IngestionErrors metric for correlation with rate limiting
- Review scrape configuration for unnecessary metrics or high-frequency scraping
- Check if multiple collectors are scraping the same targets (duplicates)

## Phase 2 — Remediate

MUST:
- If ingestion rate exceeds quota: request increase via `aws service-quotas request-service-quota-increase --service-code aps --quota-code L-6B340E60 --desired-value <new-limit>`
- If active series limit exceeded: reduce cardinality (see B4-high-cardinality)
- Configure collector retry with exponential backoff for 429 responses

SHOULD:
- Reduce scrape interval for non-critical metrics (15s → 30s or 60s)
- Drop unnecessary metrics using metric_relabel_configs in Prometheus or processors in ADOT
- Deduplicate scrape targets if multiple collectors scrape the same endpoints
- Stagger collector scrape times to smooth ingestion rate

MAY:
- Split high-volume workloads across multiple AMP workspaces
- Implement pre-aggregation using recording rules to reduce raw series count
- Set up CloudWatch alarms at 80% of quota to get early warning

## Common Issues

- symptoms: "Collector logs show repeated HTTP 429 responses"
  diagnosis: "Ingestion rate exceeds workspace quota."
  resolution: "Request quota increase. Reduce scrape frequency or drop unnecessary metrics."

- symptoms: "Sudden spike in 429 errors after deploying new application"
  diagnosis: "New application exposes high-cardinality metrics, spiking active series."
  resolution: "Review new application's metrics. Drop high-cardinality labels via relabel_configs."

- symptoms: "429 errors during specific time windows (e.g., deployments)"
  diagnosis: "Burst of new series during deployments as old and new pods coexist."
  resolution: "This is transient. Ensure collector has retry logic. Consider increasing quota headroom."

- symptoms: "Active series near limit but ingestion rate is low"
  diagnosis: "High cardinality — many unique series with low sample frequency."
  resolution: "Focus on reducing cardinality, not ingestion rate. See B4-high-cardinality."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics (IngestionRate/ActiveSeries): GREEN — read-only metric check"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
  - "request-service-quota-increase: YELLOW — changes account limits"
  - "reduce scrape interval: YELLOW — changes data collection frequency"
  - "drop metrics via relabel_configs: YELLOW — permanently stops collecting specific metrics"
```

## Escalation Conditions

- Rate limiting causing sustained data loss in production monitoring
- Quota increase needed urgently and cannot wait for standard processing
- Multiple workspaces hitting rate limits simultaneously
- New application deployment caused sudden ingestion spike

## Data Sensitivity

- CloudWatch ingestion metrics: LOW — aggregated rate and count data
- Collector scrape configuration: MEDIUM — reveals monitored targets and metrics
- Service quota values: LOW — account limit metadata

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER drop metrics without confirming they are not used in alerts or dashboards
- NEVER disable collector retry logic to reduce ingestion rate

## Phase 3 — Rollback

1. If scrape interval was changed, restore original interval in collector configuration
2. If metrics were dropped via relabel_configs, remove the drop rules to resume collection
3. If collector sharding was changed, restore original shard configuration
4. Verify ingestion rate returns to normal by checking CloudWatch IngestionRate metric
5. Confirm no critical metrics were lost by querying recently dropped metric names

## Output Format

```yaml
root_cause: "Rate limiting — <specific_cause>"
evidence:
  - type: cloudwatch_metric
    content: "<IngestionRate and ActiveSeries values vs quota>"
  - type: collector_logs
    content: "<429 error frequency and pattern>"
severity: HIGH
mitigation:
  immediate: "Request quota increase and/or reduce ingestion volume"
  long_term: "Implement metric filtering, cardinality controls, and quota monitoring alarms"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
