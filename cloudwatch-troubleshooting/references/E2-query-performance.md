---
title: "E2 — Log Insights Query Performance"
description: "Diagnose slow or timing out Logs Insights queries"
status: active
severity: MEDIUM
triggers:
  - "query slow"
  - "query timeout"
  - "query performance"
  - "query taking long"
  - "15 minute timeout"
owner: devops-agent
objective: "Optimize CloudWatch Logs Insights queries that are slow or timing out"
context: "Logs Insights queries have a hard 15-minute timeout. Query performance depends on the volume of data scanned, query complexity, number of log groups queried, and whether filters are applied early. Queries scan all data in the selected time range across selected log groups."
---

## Phase 1 — Triage

MUST:
- Check the time range of the query (wider = more data scanned)
- Count the number of log groups being queried simultaneously
- Check if `filter` commands are placed early in the query
- Verify the query isn't scanning unnecessarily large log groups

SHOULD:
- Check the bytes scanned metric for the query
- Identify if `stats` aggregation is over high-cardinality fields
- Check if `parse` with regex is applied to all records unnecessarily

MAY:
- Compare query execution time across different time ranges
- Check concurrent query limits (max 30 concurrent queries per account per region)

## Phase 2 — Remediate

MUST:
- Narrow the time range to the minimum needed
- Place `filter` commands as early as possible in the query pipeline
- Reduce the number of log groups queried simultaneously

SHOULD:
- Use `limit` clause to cap results when full scan isn't needed
- Avoid `stats` on high-cardinality fields (use `filter` first to reduce data)
- Use `@logStream` filter if the relevant data is in specific log streams

MAY:
- Split complex queries into multiple simpler queries
- Use CloudWatch Logs export to S3 + Athena for very large analytical queries
- Schedule queries during off-peak hours

## Common Issues

- symptoms: "Query times out after 15 minutes"
  diagnosis: "Query scans too much data across a wide time range or many log groups."
  resolution: "Narrow time range, add early filters, reduce log groups, or use limit clause."

- symptoms: "Query is slow but doesn't timeout"
  diagnosis: "Stats aggregation over high-cardinality field or complex regex parse."
  resolution: "Filter data before stats. Simplify regex patterns. Reduce cardinality."

- symptoms: "Concurrent query limit reached"
  diagnosis: "More than 30 queries running simultaneously in the account/region."
  resolution: "Wait for running queries to complete. Stagger query execution."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Narrow time range | GREEN | Query optimization |
| Add early filters | GREEN | Query optimization |
| Use LIMIT clause | GREEN | Query optimization |
| Reduce log groups queried | GREEN | Query optimization |

## Escalation Conditions

- Query timeouts blocking incident investigation — escalate for priority
- Concurrent query limit reached during incident — escalate to operations team
- Large-scale log analysis needed — escalate to data engineering team for S3 export + Athena

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| Log Insights query results | HIGH | May contain application log data including PII |
| Bytes scanned metrics | LOW | Numeric data only |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** run unbounded queries on large log groups during peak hours

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| No state-changing actions in query optimization | No rollback needed |

## Output Format

```yaml
root_cause: "query_performance — <specific_cause>"
evidence:
  - type: query
    content: "<the slow query>"
  - type: scan_volume
    content: "<bytes scanned and time range>"
severity: MEDIUM
mitigation:
  immediate: "Narrow time range, add early filters, use limit"
  long_term: "Optimize query patterns, consider S3 export + Athena for large analyses"
```

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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
