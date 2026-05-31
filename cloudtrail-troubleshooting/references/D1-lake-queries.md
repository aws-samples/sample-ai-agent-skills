---
title: "D1 — CloudTrail Lake Queries"
description: "Diagnose CloudTrail Lake query issues"
status: active
severity: MEDIUM
triggers:
  - "Lake query failed"
  - "query timeout"
  - "SQL error"
  - "query results empty"
owner: devops-agent
objective: "Fix CloudTrail Lake query issues and optimize query performance"
context: "CloudTrail Lake uses SQL to query event data stores. Queries can fail due to syntax errors, permission issues, empty data stores, or timeout. Lake queries are charged per GB scanned."
---

## Phase 1 — Triage

MUST:
- List event data stores: `aws cloudtrail list-event-data-stores`
- Check query status: `aws cloudtrail list-queries --event-data-store <eds-arn> --query-status FAILED`
- Get query results: `aws cloudtrail get-query-results --event-data-store <eds-arn> --query-id <query-id>`
- Verify event data store has events: `aws cloudtrail get-event-data-store --event-data-store <eds-arn>`

SHOULD:
- Check SQL syntax for common errors
- Verify time range in query matches data store retention
- Check IAM permissions for Lake queries

MAY:
- Review query execution statistics
- Optimize query to reduce scanned data

## Phase 2 — Remediate

MUST:
- Fix SQL syntax errors
- Ensure event data store contains events for the queried time range
- Grant cloudtrail:StartQuery and cloudtrail:GetQueryResults permissions

SHOULD:
- Use time-based filters to reduce scanned data and cost
- Test queries with LIMIT clause first
- Start query: `aws cloudtrail start-query --query-statement "SELECT * FROM <eds-id> WHERE eventTime > '2024-01-01' LIMIT 10"`

MAY:
- Create saved queries for common investigations
- Set up query result delivery to S3

## Common Issues

- symptoms: "Query returns no results"
  diagnosis: "Time range doesn't match available data or event type not in data store."
  resolution: "Check data store retention period. Verify event types included."

- symptoms: "Query timeout"
  diagnosis: "Query scanning too much data without filters."
  resolution: "Add time-based WHERE clause. Use specific event filters."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix SQL syntax errors | GREEN | Query-only, no state changes |
| Grant query permissions | YELLOW | IAM changes have broad impact potential |
| Start query with LIMIT clause | GREEN | Read-only query |

## Escalation Conditions

- Lake queries needed for active security investigation — escalate to security team for priority
- Query costs exceeding budget — escalate to cost management team
- Event data store empty when events are expected — escalate to trail configuration owner

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail list-event-data-stores` | HIGH | Exposes data store configuration |
| `aws cloudtrail get-query-results` | HIGH | Exposes CloudTrail event data |
| `aws cloudtrail start-query` | HIGH | Queries audit event data |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** run unbounded queries (no time filter) on large data stores — always use time-based WHERE clauses

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Granted query permissions | Remove the added IAM policy statements |
| No state-changing actions in query remediation | No rollback needed for query fixes |

## Output Format

```yaml
root_cause: "lake_query — <specific_cause>"
evidence:
  - type: query_status
    content: "<query status and error>"
  - type: data_store
    content: "<event data store configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix query syntax or permissions"
  long_term: "Optimize queries and create saved query templates"
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
  - command: "lookup-events"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-trail-status"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-trails"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling CloudTrail logging"
  - "NEVER suggest deleting trail logs"
  - "NEVER reduce logging scope in production"
