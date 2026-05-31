---
title: "C1 — Query Timeouts"
description: "Diagnose and resolve AMP query timeout issues"
status: active
severity: HIGH
triggers:
  - "query timeout"
  - "query timed out"
  - "504"
  - "query took too long"
  - "slow query"
owner: devops-agent
objective: "Identify the cause of query timeouts and optimize query performance"
context: "AMP query endpoint has execution time limits. Queries that scan too many series, use wide time ranges, or lack label matchers can exceed these limits. High cardinality is the most common root cause of query timeouts."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch query error metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Prometheus --metric-name QueryErrors --dimensions Name=WorkspaceId,Value=<workspace-id> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Sum`
- Test a simple query to verify endpoint is responsive: `up` or `count({__name__=~".+"})`
- Check active series count — high series count correlates with slow queries
- Identify the specific query that is timing out

SHOULD:
- Check the time range of the failing query — wider ranges scan more data
- Check if the query uses regex matchers without anchoring (e.g., `{__name__=~".*"}`)
- Verify the query includes at least one label matcher to narrow scope
- Check if the issue is specific to certain metrics or affects all queries

MAY:
- Test the query with a narrower time range to confirm it's a data volume issue
- Check if recording rules could pre-compute the result
- Review Grafana dashboard queries for inefficient patterns

## Phase 2 — Remediate

MUST:
- Add label matchers to narrow query scope: `metric_name{env="prod",service="api"}` instead of `metric_name`
- Reduce query time range: use `[5m]` instead of `[1h]` where possible
- Use `rate()` or `increase()` with appropriate range vectors instead of raw counters

SHOULD:
- Create recording rules for frequently-used expensive queries
- Optimize regex matchers: use exact matches where possible
- Avoid `{__name__=~".*"}` — always specify metric name or use specific regex
- Use `topk()` or `bottomk()` to limit result set size

MAY:
- Split complex queries into simpler sub-queries
- Use `offset` modifier to query historical data in smaller chunks
- Implement query caching in Grafana

## Common Issues

- symptoms: "All queries timeout — even simple ones like 'up'"
  diagnosis: "Workspace may be overloaded or experiencing a service issue."
  resolution: "Check AWS Health Dashboard. Verify workspace status is ACTIVE. If persistent, contact AWS Support."

- symptoms: "Specific dashboard queries timeout but ad-hoc queries work"
  diagnosis: "Dashboard queries are too broad or use inefficient patterns."
  resolution: "Optimize dashboard queries: add label filters, reduce time ranges, use recording rules."

- symptoms: "Query works with 1h range but fails with 24h range"
  diagnosis: "Query scans too much data over the wider time range."
  resolution: "Use recording rules to pre-aggregate. Reduce resolution with step parameter. Add label matchers."

- symptoms: "Regex-based queries timeout"
  diagnosis: "Unanchored regex matches scan all series in the workspace."
  resolution: "Anchor regex patterns. Use exact label matchers where possible. Avoid {__name__=~'.*'}."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics (QueryErrors): GREEN — read-only metric check"
  - "test simple query (up): GREEN — read-only query"
  - "add label matchers to queries: GREEN — query optimization, no data changes"
  - "create recording rules: YELLOW — creates new time series in the workspace"
```

## Escalation Conditions

- All queries timing out, including simple ones like 'up'
- Query timeouts affect production alerting rules
- Dashboard queries cannot be optimized further without recording rules
- Workspace may be overloaded, requiring AWS Support engagement

## Data Sensitivity

- CloudWatch query metrics: LOW — aggregated error counts
- PromQL query analysis: MEDIUM — reveals query patterns and metric names
- Recording rule expressions: MEDIUM — reveals aggregation logic

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER use `{__name__=~".*"}` in production queries
- NEVER increase query timeout as a permanent fix for poorly optimized queries

## Phase 3 — Rollback

1. If recording rules were created and are producing incorrect data, delete the rule group namespace
2. If dashboard queries were modified, restore from dashboard JSON backup
3. If query optimizations broke alert rules, restore original alert expressions
4. Verify all dashboards and alerts function correctly after rollback
5. Document the query patterns that caused timeouts for future optimization

## Output Format

```yaml
root_cause: "Query timeout — <specific_cause>"
evidence:
  - type: cloudwatch_metric
    content: "<QueryErrors metric data>"
  - type: query_analysis
    content: "<problematic query and its characteristics>"
severity: HIGH
mitigation:
  immediate: "Optimize the specific query with label matchers and narrower time ranges"
  long_term: "Implement recording rules for expensive queries, enforce query best practices"
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
