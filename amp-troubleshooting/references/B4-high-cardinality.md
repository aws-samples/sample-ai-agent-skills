---
title: "B4 — High Cardinality"
description: "Diagnose and resolve high cardinality issues causing series explosion in AMP"
status: active
severity: HIGH
triggers:
  - "high cardinality"
  - "too many series"
  - "active series limit"
  - "label cardinality"
  - "series explosion"
  - "cardinality"
owner: devops-agent
objective: "Identify high-cardinality labels and reduce active series count"
context: "High cardinality is the #1 performance and cost issue in AMP. Each unique combination of metric name and label key-value pairs creates a distinct time series. Labels with many unique values (pod UIDs, request IDs, user IDs) cause exponential series growth, leading to throttling, slow queries, and increased costs."
---

## Phase 1 — Triage

MUST:
- Check active series count: `aws cloudwatch get-metric-statistics --namespace AWS/Prometheus --metric-name ActiveSeries --dimensions Name=WorkspaceId,Value=<workspace-id> --start-time $(date -u -d '6 hours ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Maximum`
- Query top metrics by series count (via Grafana or curl): `count({__name__=~".+"}) by (__name__)` — sort by count descending
- Check for high-cardinality labels on top metrics: `count(metric_name) by (label_name)` for each suspected label
- Check service quota for active series limit

SHOULD:
- Identify labels with unbounded values: pod UIDs, request IDs, timestamps, user IDs
- Compare current active series against the quota limit
- Check if series count has been growing over time (trend analysis)
- Review recently deployed applications for new high-cardinality metrics

MAY:
- Use `label_values(metric_name, label_name)` to enumerate unique values for suspected labels
- Check if recording rules can pre-aggregate high-cardinality metrics
- Review collector scrape configs for unnecessary targets or metrics

## Phase 2 — Remediate

MUST:
- Drop high-cardinality labels using metric_relabel_configs in Prometheus:
  ```yaml
  metric_relabel_configs:
    - source_labels: [__name__]
      regex: 'high_cardinality_metric'
      action: drop
    - regex: 'pod_uid|request_id|trace_id'
      action: labeldrop
  ```
- Or use ADOT processor to filter metrics before export
- Remove labels that carry no aggregation value (unique IDs, timestamps)

SHOULD:
- Implement allow-lists for metrics: only send metrics you actually query
- Reduce label values by bucketing (e.g., replace exact user IDs with user_type)
- Use recording rules to pre-aggregate and drop raw high-cardinality series
- Set up CloudWatch alarms at 80% of active series quota

MAY:
- Split workloads across multiple workspaces to distribute series
- Implement metric governance: require review before exposing new metrics
- Use Prometheus federation or ADOT filtering to reduce what reaches AMP

## Common Issues

- symptoms: "ActiveSeries approaching or exceeding workspace limit"
  diagnosis: "One or more metrics have labels with very high cardinality."
  resolution: "Identify top metrics by series count. Drop or relabel high-cardinality labels."

- symptoms: "Series count spikes during deployments"
  diagnosis: "Rolling deployments create new series for new pods while old pod series are still active."
  resolution: "This is transient (series expire after 15 min of no samples). Ensure headroom in series quota for deployment churn."

- symptoms: "Single metric accounts for millions of series"
  diagnosis: "Metric has a label with unbounded unique values (e.g., request_id, pod UID)."
  resolution: "Drop the offending label via metric_relabel_configs or remove it from the instrumentation."

- symptoms: "Queries on high-cardinality metrics are extremely slow"
  diagnosis: "Query engine must process millions of series, causing timeouts."
  resolution: "Reduce cardinality first. Use recording rules to pre-aggregate. Add label matchers to narrow query scope."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics (ActiveSeries): GREEN — read-only metric check"
  - "PromQL count queries: GREEN — read-only analysis"
  - "drop labels via metric_relabel_configs: YELLOW — permanently removes label data"
  - "drop metrics via relabel_configs: YELLOW — permanently stops collecting metrics"
  - "implement allow-lists: YELLOW — restricts which metrics are collected"
```

## Escalation Conditions

- Active series approaching or exceeding workspace limit
- Single metric accounts for millions of series, requiring application instrumentation changes
- Cardinality reduction requires coordination with application development teams
- High cardinality causing query timeouts on production dashboards

## Data Sensitivity

- Active series count: LOW — aggregated count
- PromQL cardinality queries: MEDIUM — reveals metric names and label patterns
- Metric relabel configurations: MEDIUM — reveals which metrics are being filtered

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER drop labels that are used in alert rules or dashboard queries
- NEVER implement allow-lists without auditing which metrics are used in dashboards and alerts

## Phase 3 — Rollback

1. If labels were dropped via relabel_configs, remove the labeldrop rules to resume collection
2. If metrics were dropped, remove the drop rules from collector configuration
3. If allow-lists were implemented too restrictively, expand them to include missing metrics
4. Verify dashboards and alerts function correctly after restoring metrics
5. Note: historical data for dropped labels/metrics cannot be recovered

## Output Format

```yaml
root_cause: "High cardinality — <specific_cause>"
evidence:
  - type: cloudwatch_metric
    content: "<ActiveSeries count vs quota>"
  - type: query_result
    content: "<top metrics by series count>"
severity: HIGH
mitigation:
  immediate: "Drop or relabel high-cardinality labels to reduce series count"
  long_term: "Implement metric governance, allow-lists, and cardinality monitoring"
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
