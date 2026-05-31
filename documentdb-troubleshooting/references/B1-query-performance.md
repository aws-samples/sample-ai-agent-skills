---
title: "B1 — Query Performance Issues"
description: "Diagnose slow queries and query performance degradation on DocumentDB"
status: active
severity: HIGH
triggers:
  - "slow query"
  - "query timeout"
  - "query performance"
  - "slow operation"
  - "high latency query"
owner: devops-agent
objective: "Identify and resolve query performance issues on DocumentDB clusters"
context: "Query performance issues are commonly caused by missing indexes, collection scans, inefficient aggregation pipelines, large result sets, or instance resource constraints. The DocumentDB profiler and explain() are the primary diagnostic tools."
---

## Phase 1 — Triage

MUST:
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name CPUUtilization --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check active operations: `mongo --tls --host <endpoint> --eval "db.currentOp({'secs_running': {$gt: 5}})"`
- Enable profiler if not enabled: check profiler parameter in cluster parameter group
- Check OpcountersQuery and OpcountersCommand metrics

SHOULD:
- Run explain on slow queries: `mongo --tls --host <endpoint> --eval "db.collection.find({query}).explain('executionStats')"`
- Check for collection scans (COLLSCAN in explain output)
- Review DocumentDB slow query log (profiler output)
- Check ReadLatency and WriteLatency metrics: `aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name ReadLatency --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

MAY:
- Analyze query patterns for optimization opportunities
- Check for lock contention with db.currentOp()
- Review aggregation pipeline efficiency

## Phase 2 — Remediate

MUST:
- Create indexes for queries performing collection scans
- Kill long-running operations if they are blocking: `mongo --eval "db.killOp(<opid>)"`
- Verify queries use appropriate indexes via explain()

SHOULD:
- Enable the profiler to capture slow queries (set profiler threshold in parameter group)
- Optimize aggregation pipelines (filter early, project only needed fields)
- Use read preference secondaryPreferred for read-heavy queries

MAY:
- Implement query result pagination to limit result set sizes
- Add reader instances to distribute read load

## Common Issues

- symptoms: "Queries suddenly slow after data growth"
  diagnosis: "Queries are performing collection scans because indexes are missing for the query pattern."
  resolution: "Create compound indexes matching the query filter and sort fields. Use explain() to verify."

- symptoms: "Aggregation pipeline times out"
  diagnosis: "Pipeline processes too much data without early filtering or uses unsupported stages."
  resolution: "Add $match stage early in the pipeline. Verify all stages are supported in your engine version."

- symptoms: "High CPU with moderate query volume"
  diagnosis: "Queries are inefficient (collection scans, large sorts, regex without anchors)."
  resolution: "Optimize queries with proper indexes. Use anchored regex (^prefix). Limit sort result sets."

## Output Format

```yaml
root_cause: "query_performance — <specific_cause>"
evidence:
  - type: explain_output
    content: "<query explain with execution stats>"
  - type: profiler
    content: "<slow query profiler entries>"
  - type: metrics
    content: "<CPU, latency, opcounters>"
severity: HIGH
mitigation:
  immediate: "Create missing indexes and optimize slow queries"
  long_term: "Implement query monitoring and index management practices"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Query analysis may expose data patterns and schema |
| Blast Radius | HIGH — Index changes and query kills affect all database users |
| Rollback Complexity | MEDIUM — Index creation blocks writes; index drops are irreversible |
| Compliance Scope | MEDIUM — Performance issues may affect application SLAs |

## Escalation Conditions

- Escalate to application owners before killing long-running operations
- Escalate to DBA team before creating indexes on production collections
- Escalate to operations if performance degradation affects application SLAs

## Data Sensitivity

- **Query patterns** reveal application behavior and data access patterns
- **Profiler output** may contain query parameters with sensitive data
- **Explain output** reveals index and collection structure

## PROHIBITED Actions

- Do NOT kill operations without understanding the application impact
- Do NOT create indexes on production during peak hours (blocks writes)
- Do NOT run explain() with production data in troubleshooting artifacts
- Do NOT share profiler output containing sensitive query parameters

## Phase 3 — Rollback

- Drop incorrectly created indexes: `db.collection.dropIndex("<index-name>")`
- Restore previous query patterns if application code was changed
- Revert parameter group changes if profiler settings were modified
- Monitor performance metrics after rollback to confirm stability

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
