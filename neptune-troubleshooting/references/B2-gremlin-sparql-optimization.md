---
title: "B2 — Gremlin/SPARQL Optimization"
description: "Optimize Gremlin and SPARQL query patterns on Neptune"
status: active
severity: MEDIUM
triggers:
  - "gremlin slow"
  - "sparql slow"
  - "optimize query"
  - "query optimization"
owner: devops-agent
objective: "Optimize Gremlin and SPARQL queries for Neptune"
context: "Neptune has specific query optimization patterns for both Gremlin and SPARQL. Understanding Neptune's query engine and indexing strategy is key to performance."
---

## Phase 1 — Triage

MUST:
- Profile the slow Gremlin query: `curl -X POST https://<endpoint>:8182/gremlin -d '{"gremlin":"<query>.profile()"}'`
- Or explain SPARQL query: `curl -X POST https://<endpoint>:8182/sparql -d 'query=EXPLAIN <sparql-query>'`
- Check if query uses full graph scan: look for "Neptune steps" in profile output
- Verify query is using indexed lookups vs scans

SHOULD:
- Check Gremlin query errors: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name GremlinErrors --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check SPARQL query errors: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name SparqlErrors --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Review query patterns for anti-patterns

MAY:
- Test query variations for performance comparison
- Check Neptune engine version for query optimizer improvements
- Review Neptune best practices documentation

## Phase 2 — Remediate

MUST:
- Rewrite queries to avoid full graph scans
- Use .has() with specific property values for Gremlin lookups
- Use FILTER with indexed predicates for SPARQL

SHOULD:
- Batch small queries into larger traversals where possible
- Use .valueMap() instead of .properties() for Gremlin
- Use OPTIONAL patterns carefully in SPARQL (can cause cartesian products)

MAY:
- Implement application-level query caching
- Consider materialized views for common query patterns

## Common Issues

- symptoms: "Gremlin g.V() without filters is slow"
  diagnosis: "Unfiltered vertex scan reads entire graph."
  resolution: "Always start traversals with .hasLabel() and .has() to use indexes."

- symptoms: "SPARQL query with multiple OPTIONAL clauses is slow"
  diagnosis: "Multiple OPTIONAL patterns can create cartesian products."
  resolution: "Restructure query to minimize OPTIONAL usage or use subqueries."

- symptoms: "Gremlin .drop() on large result sets is slow"
  diagnosis: "Dropping many elements in a single query is resource-intensive."
  resolution: "Batch deletes using .limit() with repeated execution."

## Output Format

```yaml
root_cause: "query_optimization — <specific_cause>"
evidence:
  - type: query_profile
    content: "<profile/explain output>"
  - type: query_pattern
    content: "<original vs optimized query>"
severity: MEDIUM
mitigation:
  immediate: "Rewrite query using optimized patterns"
  long_term: "Review data model for query-friendly structure"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — optimization work focuses on query structure
- Service Disruption Risk: MEDIUM — query changes can affect application behavior
- Compliance Impact: LOW — query optimization is an operational concern

## Escalation Conditions
- Optimized queries return different results than original queries
- Query optimization causes application-level errors
- Gremlin or SPARQL syntax changes break client compatibility
- Query plan changes cause unexpected resource consumption
- Optimization changes affect data consistency

## Data Sensitivity
- Query syntax and patterns — INTERNAL
- Graph schema and data model — INTERNAL
- Query execution statistics — INTERNAL
- Application query templates — INTERNAL
- Never expose actual graph data values in optimization examples

## Prohibited Actions
- Do NOT modify production queries without testing in a staging environment
- Do NOT remove query filters that enforce access control
- Do NOT bypass Neptune query limits for optimization testing
- Do NOT use profile or explain endpoints on production during peak hours
- Do NOT change query serialization settings without understanding consistency impact
- Do NOT optimize queries in ways that bypass application-level security checks

## Phase 3 — Rollback
- Revert application query changes to previous versions
- Restore query templates to pre-optimization state
- Verify query results match expected output after rollback
- Re-enable any query safeguards that were temporarily disabled
- Document optimization attempts and their outcomes
- Update query best practices documentation

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
