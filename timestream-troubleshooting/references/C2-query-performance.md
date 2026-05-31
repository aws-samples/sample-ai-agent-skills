---
title: "C2 — Query Performance Issues"
description: "Diagnose Timestream slow query performance"
status: active
severity: MEDIUM
triggers:
  - "slow query"
  - "query timeout"
  - "query performance"
  - "high query latency"
owner: devops-agent
objective: "Identify and resolve query performance issues"
context: "Query performance depends on data volume, time range, query complexity, and whether data is in memory or magnetic store. Memory store queries are faster. Broad time ranges and missing filters cause full table scans."
---

## Phase 1 — Triage

MUST:
- Check query execution time and data scanned
- Verify the query includes a time filter (WHERE time > ago(...))
- Check if the query is scanning memory or magnetic store data
- Review CloudWatch query latency metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Timestream --metric-name SuccessfulRequestLatency --dimensions Name=DatabaseName,Value=<database> Name=Operation,Value=Query --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Check if dimension filters are used to narrow the scan
- Verify the query doesn't use SELECT * on large tables
- Check for expensive operations (ORDER BY, GROUP BY on high-cardinality columns)

MAY:
- Compare query performance across different time ranges
- Check account-level query concurrency

## Phase 2 — Remediate

MUST:
- Add time filters to limit the scan range
- Add dimension filters to narrow the data scanned
- Select only needed columns instead of SELECT *

SHOULD:
- Use scheduled queries to pre-aggregate data for common queries
- Optimize GROUP BY to use indexed dimensions
- Limit result sets with LIMIT clause

MAY:
- Adjust memory store retention to keep frequently queried data in memory
- Create materialized views via scheduled queries

## Common Issues

- symptoms: "Query takes minutes to complete"
  diagnosis: "No time filter — scanning entire table history."
  resolution: "Add WHERE time > ago(1h) or appropriate time range."

- symptoms: "Query fast for recent data but slow for historical"
  diagnosis: "Historical data in magnetic store has higher latency."
  resolution: "Use scheduled queries to pre-aggregate historical data."

- symptoms: "Query timeout at 60 seconds"
  diagnosis: "Query too complex or scanning too much data."
  resolution: "Narrow time range, add dimension filters, reduce result set."

## Output Format

```yaml
root_cause: "query_performance_issue — <specific_cause>"
evidence:
  - type: query_latency
    content: "<query execution time>"
  - type: data_scanned
    content: "<amount of data scanned>"
  - type: query_plan
    content: "<query structure and filters>"
severity: MEDIUM
mitigation:
  immediate: "Optimize query with filters and limits"
  long_term: "Use scheduled queries for common aggregations"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Time-series data may contain IoT sensor readings and telemetry
- Query results may reveal operational patterns and metrics
- Scheduled query outputs may aggregate sensitive measurements
- Classification: CONFIDENTIAL - operational telemetry and measurement data

## Prohibited Actions

- NEVER delete Timestream tables containing production time-series data
- NEVER reduce memory store retention below the current data window
- NEVER modify magnetic store retention without data lifecycle review
- NEVER cancel scheduled queries that feed production dashboards

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
