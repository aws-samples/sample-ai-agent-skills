---
title: "F2 — Query Performance Issues"
description: "Diagnose AWS Config advanced query performance and timeout issues"
status: active
severity: MEDIUM
triggers:
  - "query timeout"
  - "query slow"
  - "query performance"
  - "query taking too long"
  - "select resource config timeout"
owner: devops-agent
objective: "Identify and resolve Config advanced query performance issues including timeouts"
context: "Config advanced queries can time out when querying large datasets, using complex conditions, or scanning many resource types. Query optimization involves narrowing scope, using indexed fields, and paginating results."
---

## Phase 1 — Triage

MUST:
- Identify the slow query and its scope
- Check the number of configuration items: `aws configservice get-discovered-resource-counts`
- Verify the query uses specific resource types in WHERE clause
- Check if the query uses indexed fields (resourceType, resourceId, accountId, awsRegion)
- Test a simplified version of the query for baseline performance

SHOULD:
- Check if the query scans across all resource types (no WHERE resourceType filter)
- Verify the query does not use SELECT * (not supported, but complex field lists can be slow)
- Check for aggregator queries which scan across accounts and regions

MAY:
- Compare query execution time with and without specific conditions
- Check if the account has an unusually large number of configuration items

## Phase 2 — Remediate

MUST:
- Add a resourceType filter to the WHERE clause to narrow scope
- Use indexed fields in WHERE conditions for faster filtering
- Paginate results using NextToken for large result sets

SHOULD:
- Break complex queries into multiple simpler queries
- Limit the number of fields in the SELECT clause
- Use specific resource IDs when possible

MAY:
- Export configuration data to S3 and use Athena for complex analytics
- Schedule queries during off-peak hours for large datasets

## Common Issues

- symptoms: "Query times out with no results returned"
  diagnosis: "The query scans too many configuration items without a resourceType filter."
  resolution: "Add WHERE resourceType = '<specific-type>' to narrow the scan scope."

- symptoms: "Query returns partial results with truncation"
  diagnosis: "Results exceed the maximum page size (100 items). Pagination is required."
  resolution: "Use the NextToken from the response to paginate through all results."

- symptoms: "Aggregator query is extremely slow"
  diagnosis: "The query scans across all accounts and regions in the aggregator, multiplying the dataset."
  resolution: "Add accountId and awsRegion filters to narrow the aggregator query scope."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Optimize query | GREEN | Query improvement; no state change |
| Add resource type filters | GREEN | Query improvement; no state change |

## Escalation Conditions

- Recorder configuration changes (if needed for query scope)

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `select-resource-config` | MEDIUM | Resource configuration data |
| `get-discovered-resource-counts` | LOW | Aggregate resource counts |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Query optimization | N/A — query improvement only |

## Output Format

```yaml
root_cause: "query_performance — <specific_cause>"
evidence:
  - type: query
    content: "<the slow query>"
  - type: resource_counts
    content: "<discovered resource counts>"
  - type: optimization
    content: "<suggested query optimization>"
severity: MEDIUM
mitigation:
  immediate: "Optimize the query with resource type filters and indexed fields"
  long_term: "Use Athena for complex analytics on exported configuration data"
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
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"
