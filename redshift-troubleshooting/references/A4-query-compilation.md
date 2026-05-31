---
title: "A4 — Query Compilation Delays"
description: "Diagnose and resolve query compilation overhead in Redshift"
status: active
severity: MEDIUM
triggers:
  - "query compilation"
  - "compile time"
  - "first query slow"
  - "cold start query"
  - "compilation cache"
owner: devops-agent
objective: "Identify query compilation overhead and optimize compilation cache usage"
context: "Redshift compiles queries into C++ code and caches the compiled segments. First execution of a new query pattern incurs compilation overhead (seconds to minutes for complex queries). Subsequent executions of the same pattern use the cache. Cluster restarts clear the compilation cache."
---

## Phase 1 — Triage

MUST:
- Check compilation time for specific query: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, segment, compile, starttime FROM svl_compile WHERE query = <query-id> ORDER BY segment"`
- Check if query was compiled or used cache: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, segment, compile FROM svl_compile WHERE query = <query-id> AND compile = 1"`
- Check recent cluster restart (clears cache): `aws redshift describe-events --source-identifier <cluster-id> --source-type cluster --duration 1440`
- Compare first vs subsequent execution times for the same query pattern

SHOULD:
- Check total compilation time vs execution time ratio
- Identify queries with many segments (complex queries compile slower)
- Check if maintenance window recently restarted the cluster

MAY:
- Review query patterns for parameterization opportunities
- Check if prepared statements are being used
- Analyze compilation cache hit rate across workload

## Phase 2 — Remediate

MUST:
- If post-restart compilation: this is expected — cache warms up over time, no action needed
- If complex query with many segments: simplify query or break into smaller steps
- If unique query patterns (no cache reuse): parameterize queries to increase cache hits

SHOULD:
- Use prepared statements for repeated query patterns
- Warm up compilation cache after maintenance by running key queries
- Avoid dynamic SQL that generates unique query text each time

MAY:
- Schedule a cache warm-up script after planned maintenance
- Monitor svl_compile for compilation patterns
- Consider materialized views to reduce complex query compilation

## Common Issues

- symptoms: "First query after cluster restart takes 30+ seconds, subsequent runs take 2 seconds"
  diagnosis: "Compilation cache was cleared by restart. First execution compiles all segments."
  resolution: "Expected behavior. Cache warms up automatically. Run key queries after restart to pre-warm."

- symptoms: "Every execution of a query is slow, even repeated runs"
  diagnosis: "Query text changes each time (e.g., embedded literals instead of parameters)."
  resolution: "Parameterize queries. Use prepared statements. Avoid embedding unique values in SQL text."

## Output Format

```yaml
root_cause: "Query compilation — <specific_cause>"
evidence:
  - type: system_table
    content: "<svl_compile output showing compile times>"
severity: MEDIUM
mitigation:
  immediate: "Pre-warm cache or parameterize queries"
  long_term: "Use prepared statements, implement cache warm-up after maintenance"
```


## Safety Ratings
```
safety_ratings:
  - "Query svl_compile: GREEN — read-only system table query"
  - "Check cluster events: GREEN — read-only API call"
  - "Run key queries to warm cache: GREEN — read-only queries, no state change"
  - "Use prepared statements: GREEN — client-side optimization"
  - "Schedule cache warm-up script: GREEN — runs read-only queries after maintenance"
```

## Escalation Conditions
- Cluster serves production analytics
- Compilation delays impacting time-sensitive dashboards after maintenance
- Complex queries with many segments causing excessive compilation time
- Frequent cluster restarts clearing compilation cache

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Query text in svl_compile: may contain sensitive SQL patterns"
    - "Compilation patterns: reveal query workload characteristics"
  handling: "Do not expose query patterns or compilation details externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER restart a cluster solely to clear compilation cache
- NEVER run cache warm-up queries that modify data

## Phase 3 — Rollback
- Cache warm-up queries are read-only — no rollback needed
- If prepared statements were introduced: revert client code to previous query patterns
- If materialized views were created for compilation reduction: drop the materialized views if not needed

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"
