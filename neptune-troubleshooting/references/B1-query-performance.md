---
title: "B1 — Query Performance"
description: "Diagnose Neptune query performance issues"
status: active
severity: HIGH
triggers:
  - "slow queries"
  - "query timeout"
  - "query performance"
  - "high latency queries"
owner: devops-agent
objective: "Identify and resolve Neptune query performance bottlenecks"
context: "Neptune query performance depends on instance class, data model design, query patterns, and buffer cache efficiency. Both Gremlin and SPARQL queries can experience performance issues."
---

## Phase 1 — Triage

MUST:
- Check query latency metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name GremlinRequestsPerSec --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check SPARQL requests: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name SparqlRequestsPerSec --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name CPUUtilization --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check buffer cache hit ratio: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name BufferCacheHitRatio --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Profile slow Gremlin query: `curl -X POST https://<endpoint>:8182/gremlin -d '{"gremlin":"g.V().hasLabel(\"person\").out().profile()"}'`
- Check SPARQL explain: `curl -X POST https://<endpoint>:8182/sparql -d 'query=EXPLAIN SELECT ...'`
- Check query timeout parameter: `aws neptune describe-db-cluster-parameters --db-cluster-parameter-group-name <param-group> --filters Name=parameter-name,Values=neptune_query_timeout`
- Monitor freeable memory for cache pressure

MAY:
- Enable audit logging for query analysis
- Check for concurrent query contention
- Review Neptune slow query logs if enabled

## Phase 2 — Remediate

MUST:
- Optimize identified slow queries using profile/explain output
- Scale instance class if buffer cache hit ratio is low
- Increase query timeout if legitimate long queries are being killed

SHOULD:
- Add appropriate indexes or restructure traversal patterns
- Use read replicas to distribute read load
- Implement query result caching at application level

MAY:
- Enable Neptune streams for change-driven processing
- Consider Neptune Serverless for variable workloads

## Common Issues

- symptoms: "Queries timing out after 120 seconds"
  diagnosis: "Default query timeout is 120 seconds."
  resolution: "Increase neptune_query_timeout parameter or optimize the query."

- symptoms: "Low buffer cache hit ratio"
  diagnosis: "Working set exceeds instance memory."
  resolution: "Scale to larger instance class with more memory."

- symptoms: "Full graph scans on simple lookups"
  diagnosis: "Query pattern not using indexed properties efficiently."
  resolution: "Use .has() steps with indexed properties. Avoid unfiltered g.V() traversals."

## Output Format

```yaml
root_cause: "query_performance — <specific_cause>"
evidence:
  - type: metrics
    content: "<latency, CPU, cache hit ratio>"
  - type: query_profile
    content: "<profile/explain output>"
severity: HIGH
mitigation:
  immediate: "Optimize slow queries or scale instance"
  long_term: "Review data model and query patterns"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — query performance tuning does not expose data
- Service Disruption Risk: MEDIUM — poorly optimized queries can affect cluster performance
- Compliance Impact: LOW — performance issues are operational concerns

## Escalation Conditions
- Query latency exceeds SLA thresholds consistently
- Long-running queries cause cluster-wide performance degradation
- Query performance degrades after Neptune engine upgrade
- Memory pressure from queries causes instance instability
- Query timeout rate exceeds acceptable thresholds

## Data Sensitivity
- Query patterns and execution plans — INTERNAL
- Query text may contain data references — CONFIDENTIAL
- Performance metrics — INTERNAL
- Slow query logs — INTERNAL (may contain CONFIDENTIAL data)
- Never expose actual graph data in performance reports

## Prohibited Actions
- Do NOT run unoptimized full-graph traversals on production clusters
- Do NOT disable query timeout limits on production instances
- Do NOT execute expensive analytical queries on the primary writer instance
- Do NOT ignore slow query log warnings
- Do NOT increase query memory limits without understanding impact
- Do NOT run EXPLAIN queries with sensitive data in production logs

## Phase 3 — Rollback
- Revert query parameter changes that degraded performance
- Cancel long-running queries that are affecting cluster health
- Restore previous parameter group settings if changes caused issues
- Redirect read queries back to reader endpoints if routing was changed
- Document query performance baselines before and after changes
- Update query optimization guidelines based on findings

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
