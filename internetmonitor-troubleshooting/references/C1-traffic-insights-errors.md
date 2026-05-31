---
title: "C1 — Traffic Insights Errors"
description: "Diagnose traffic insights query failures and data issues"
status: active
severity: MEDIUM
triggers: ["query failed", "no traffic data", "insights not available", "start-query error"]
owner: devops-agent
objective: "Resolve traffic insights query errors and missing data issues"
context: "Traffic insights use the start-query and get-query-results APIs to retrieve detailed measurements, top locations, and traffic patterns."
---

## Phase 1 — Triage

MUST:
- Verify monitor exists and is ACTIVE: `aws internetmonitor get-monitor --monitor-name <name>`
- Check ProcessingStatus — must be OK for queries to return data
- Attempt a basic query: `aws internetmonitor start-query --monitor-name <name> --query-type MEASUREMENTS --filter-parameters '{"StartTime":"<time>","EndTime":"<time>"}'`

SHOULD:
- Verify the time range is within the data retention period
- Check if the query type is valid (MEASUREMENTS, TOP_LOCATIONS, TOP_LOCATION_DETAILS)
- Review IAM permissions for internetmonitor:StartQuery and internetmonitor:GetQueryResults

MAY:
- Check CloudTrail for StartQuery errors
- Verify the monitor has been active long enough to accumulate data

## Phase 2 — Remediate

MUST:
- Fix time range parameters to fall within data retention period
- Ensure monitor has ACTIVE status and OK ProcessingStatus
- Grant required IAM permissions for query APIs

SHOULD:
- Use appropriate query types for the information needed
- Paginate results using NextToken for large result sets

MAY:
- Set up scheduled queries for regular reporting
- Export query results to S3 for long-term analysis

## Common Issues

- symptoms: "start-query returns empty results despite active monitor"
  diagnosis: "Time range may be outside data retention or monitor may have INSUFFICIENT_DATA."
  resolution: "Verify ProcessingStatus is OK. Adjust time range to when the monitor was actively collecting data. Check that monitored resources had traffic during the queried period."

## Safety Ratings

```
safety_ratings:
  - "get-monitor: GREEN — read-only monitor details"
  - "start-query: GREEN — initiates read-only data query"
  - "get-query-results: GREEN — retrieves read-only query results"
  - "get-query-status: GREEN — checks query execution status"
```

## Escalation Conditions

- Queries consistently fail with internal errors despite valid parameters
- ProcessingStatus stuck in error state and not recovering
- Query results show data gaps that do not correlate with traffic patterns
- Query API returns throttling errors during normal usage

## Data Sensitivity

- `aws internetmonitor start-query`: LOW — initiates a query, no sensitive data in request
- `aws internetmonitor get-query-results`: MEDIUM — reveals traffic patterns, city-networks, and performance data
- `aws internetmonitor get-monitor`: LOW — returns monitor configuration

## Prohibited Actions

- NEVER assume empty query results mean no traffic — check ProcessingStatus first
- NEVER run unbounded time range queries that could return excessive data
- NEVER share raw query results containing city-network traffic patterns without authorization
- NEVER modify the monitor configuration to troubleshoot query issues

## Phase 3 — Rollback

1. No destructive operations in query troubleshooting — queries are read-only
2. If IAM permissions were broadened for troubleshooting, scope them back to minimum required
3. If scheduled queries were created during investigation, remove them if not needed
4. Document query patterns that work and those that fail for future reference
5. Verify query functionality is restored before closing the investigation

## Output Format

```yaml
root_cause: "traffic_insights — <specific_cause>"
evidence:
  - type: query_error
    content: "<error details and query parameters>"
severity: MEDIUM
mitigation:
  immediate: "Fix query parameters or monitor status"
  long_term: "Document valid query patterns and set up automated reporting"
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
