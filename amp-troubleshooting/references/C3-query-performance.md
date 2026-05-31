---
title: "C3 — Query Performance"
description: "Diagnose and optimize slow PromQL query performance in AMP"
status: active
severity: MEDIUM
triggers:
  - "slow query"
  - "query performance"
  - "query latency"
  - "dashboard slow"
  - "Grafana slow"
owner: devops-agent
objective: "Identify query performance bottlenecks and optimize for faster execution"
context: "AMP query performance depends on the number of series scanned, time range, query complexity, and label cardinality. Slow queries are usually caused by scanning too many series, using wide time ranges without step optimization, or missing label matchers."
---

## Phase 1 — Triage

MUST:
- Identify the slow query (from Grafana, application, or user report)
- Check active series count — high series count impacts all queries
- Test the query with a narrow time range (last 5 minutes) to establish baseline
- Check if the query uses label matchers to narrow scope

SHOULD:
- Compare query execution time with and without specific label matchers
- Check if the query uses expensive operations: `{__name__=~".*"}`, unanchored regex, `count()` without `by`
- Verify Grafana dashboard settings: auto-refresh interval, time range, max data points
- Check if recording rules exist for frequently-used aggregations

MAY:
- Profile the query by breaking it into sub-expressions
- Check if the same query was faster in the past (regression analysis)
- Review CloudWatch metrics for overall workspace query load

## Phase 2 — Remediate

MUST:
- Add specific label matchers to reduce series scanned
- Use recording rules for expensive aggregations that run frequently:
  ```yaml
  groups:
    - name: pre-aggregated
      rules:
        - record: job:http_requests:rate5m
          expr: sum(rate(http_requests_total[5m])) by (job)
  ```
- Reduce time range where possible

SHOULD:
- Optimize Grafana dashboards: increase min step, reduce max data points
- Replace regex matchers with exact matches where possible
- Use `topk()` or `bottomk()` to limit result sets in dashboards
- Avoid `label_values()` on high-cardinality labels in Grafana variables

MAY:
- Implement query result caching in Grafana
- Split large dashboards into focused views
- Use Grafana's query inspector to analyze query timing

## Common Issues

- symptoms: "Grafana dashboard takes 30+ seconds to load"
  diagnosis: "Dashboard has too many panels with broad queries running simultaneously."
  resolution: "Optimize individual panel queries. Add label matchers. Use recording rules. Increase min step interval."

- symptoms: "Query is fast with 1h range but slow with 7d range"
  diagnosis: "Query scans proportionally more data over wider ranges."
  resolution: "Increase step parameter for wider ranges. Use recording rules for long-range aggregations."

- symptoms: "Simple metric query is slow despite low series count"
  diagnosis: "Workspace may be under heavy query load from other sources."
  resolution: "Check overall query load. Stagger dashboard refresh intervals. Consider query prioritization."

- symptoms: "Grafana variable queries are slow"
  diagnosis: "label_values() on high-cardinality labels scans all series."
  resolution: "Use static variable lists or query with specific metric name: label_values(specific_metric, label)."

## Safety Ratings

```
safety_ratings:
  - "identify slow query: GREEN — read-only analysis"
  - "test query with narrow time range: GREEN — read-only query"
  - "create recording rules: YELLOW — creates new time series in workspace"
  - "optimize Grafana dashboard settings: YELLOW — changes dashboard behavior"
```

## Escalation Conditions

- Slow queries affect production dashboard usability
- Query performance regression after workspace or collector changes
- Recording rules needed but require ruler configuration changes
- Overall workspace query load is too high, affecting all users

## Data Sensitivity

- Query analysis: MEDIUM — reveals query patterns and metric names
- CloudWatch workspace metrics: LOW — aggregated performance data
- Recording rule expressions: MEDIUM — reveals aggregation logic

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER use `label_values()` on high-cardinality labels in Grafana variables
- NEVER set dashboard auto-refresh to less than 30 seconds for production dashboards

## Phase 3 — Rollback

1. If recording rules were created and are incorrect, delete the rule group namespace
2. If dashboard settings were changed, restore from dashboard JSON backup
3. If Grafana variable queries were modified, restore original variable configuration
4. Verify dashboard load times are acceptable after rollback
5. Document the performance optimization attempts for future reference

## Output Format

```yaml
root_cause: "Query performance — <specific_cause>"
evidence:
  - type: query_analysis
    content: "<query pattern and execution characteristics>"
  - type: cloudwatch_metric
    content: "<ActiveSeries and query load metrics>"
severity: MEDIUM
mitigation:
  immediate: "Optimize the specific slow query with label matchers and recording rules"
  long_term: "Implement recording rules, optimize dashboards, enforce query best practices"
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
