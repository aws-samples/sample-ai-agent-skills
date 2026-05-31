---
title: "H2 — OpenSearch Dashboards Visualization Errors"
description: "Diagnose and resolve visualization rendering failures and query errors in Dashboards"
status: active
severity: MEDIUM
triggers:
  - "visualization error"
  - "Dashboards error"
  - "query failed"
  - "no results"
  - "Dashboards timeout"
  - "index pattern"
owner: devops-agent
objective: "Resolve Dashboards visualization errors and restore data display"
context: "Dashboards visualizations can fail due to index pattern misconfiguration, field mapping issues, query timeouts, FGAC permission restrictions, or cluster resource pressure. Visualizations execute OpenSearch queries under the hood — any query-level issue affects visualizations. Common problems include missing index patterns, time field misconfiguration, and aggregation errors on incompatible field types."
---

## Phase 1 — Triage

MUST:
- Check index pattern exists and is correct in Dashboards (Stack Management → Index Patterns)
- Test the underlying query directly: `curl -XGET "https://<endpoint>/<index>/_search?pretty&size=1"`
- Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`
- Check index exists and has data: `curl -XGET "https://<endpoint>/_cat/indices/<index>?v&h=index,docs.count,store.size"`
- Check JVM memory pressure: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name JVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

SHOULD:
- Check field mappings for the index: `curl -XGET "https://<endpoint>/<index>/_mapping?pretty"`
- Check if time field is correctly configured in the index pattern
- Check FGAC permissions for the Dashboards user (see F2)
- Check for query timeout settings

MAY:
- Check Dashboards saved objects: `curl -XGET "https://<endpoint>/_dashboards/api/saved_objects/_find?type=visualization&per_page=100"`
- Check browser console for JavaScript errors
- Review Dashboards slow query log

## Phase 2 — Remediate

MUST:
- If index pattern missing: recreate in Stack Management → Index Patterns
- If time field wrong: update index pattern with correct timestamp field
- If query timeout: optimize the query or increase timeout
- If FGAC blocking: grant read permissions on the index (see F2)

SHOULD:
- Refresh index pattern field list after mapping changes
- Use date histogram with appropriate interval for time-based visualizations
- Limit aggregation cardinality to prevent memory issues

MAY:
- Use Dashboards query profiler to identify slow visualization queries
- Create optimized indices or materialized views for complex dashboards
- Set up Dashboards reporting for scheduled exports

## Common Issues

- symptoms: "No results found in visualization"
  diagnosis: "Time range filter excludes all data, or index pattern does not match any indices."
  resolution: "Adjust time range. Verify index pattern matches existing indices with data."

- symptoms: "Visualization shows error: Fielddata is disabled on text fields"
  diagnosis: "Aggregation on a text field requires fielddata (expensive) or a keyword sub-field."
  resolution: "Use the .keyword sub-field for aggregations instead of the text field."

- symptoms: "Visualization timeout error"
  diagnosis: "Query too complex or cluster under resource pressure."
  resolution: "Simplify query, reduce time range, or scale cluster. Check JVM pressure."

## Output Format

```yaml
root_cause: "visualization_error — <specific_cause>"
evidence:
  - type: index_pattern
    content: "<index pattern configuration>"
  - type: query_test
    content: "<direct query result>"
  - type: cluster_resources
    content: "<JVM, CPU metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix index pattern, field mapping, or query"
  long_term: "Optimize dashboards, set up monitoring for visualization health"
```


## Safety Ratings
```
safety_ratings:
  - "Check index patterns and mappings: GREEN — read-only diagnostics"
  - "Test underlying queries: GREEN — read-only search"
  - "Recreate index pattern: GREEN — metadata-only change"
  - "Refresh field list: GREEN — metadata update"
  - "Optimize queries: GREEN — client-side changes"
```

## Escalation Conditions
- Domain serves production search
- Dashboard visualizations broken for business users
- Query timeouts from cluster resource pressure
- FGAC permissions blocking visualization queries
- Index pattern changes affecting multiple dashboards

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Visualization queries: search terms and filters"
    - "Dashboard data: business analytics"
    - "Index patterns: data structure information"
  handling: "Dashboards display business data. Restrict access to authorized users."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER delete saved objects (dashboards, visualizations) without backup
- NEVER enable fielddata on text fields without understanding memory impact

## Phase 3 — Rollback
- If index pattern was recreated: restore previous pattern configuration
- If saved objects were modified: restore from Dashboards export backup
- If query timeout was changed: revert to previous timeout value

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
