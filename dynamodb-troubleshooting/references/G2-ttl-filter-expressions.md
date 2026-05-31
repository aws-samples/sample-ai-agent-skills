---
title: "G2 — TTL Filter Expressions"
description: "Filter expired items in queries and scans while waiting for TTL deletion"
status: active
severity: MEDIUM
triggers:
  - "expired items in query"
  - "TTL filter"
  - "stale items"
  - "filter expired"
  - "TTL query results"
owner: devops-agent
objective: "Implement proper filtering of expired items in application queries"
context: "DynamoDB TTL does not delete items instantly — deletion can take up to 48 hours after expiration. During this window, expired items still appear in Query and Scan results. Applications must filter expired items client-side or use FilterExpression to exclude them. FilterExpression does not reduce consumed capacity but prevents expired items from being returned. GSI queries also return expired items until they are deleted from the base table."
---

## Phase 1 — Triage

MUST:
- Verify TTL is enabled: `aws dynamodb describe-time-to-live --table-name <table>`
- Check if expired items appear in query results: `aws dynamodb query --table-name <table> --key-condition-expression "PK = :pk" --expression-attribute-values '{":pk":{"S":"<value>"}}' --projection-expression "#ttl" --expression-attribute-names '{"#ttl":"<ttl-attribute>"}'`
- Verify the current epoch time vs TTL values in returned items
- Check if the application filters expired items

SHOULD:
- Check how many expired items are in the table
- Review all query patterns to ensure they filter expired items
- Check if GSI queries also return expired items

MAY:
- Estimate the RCU cost of reading expired items
- Check if a cleanup job could supplement TTL for faster removal
- Review if the application handles expired items gracefully

## Phase 2 — Remediate

MUST:
- Add FilterExpression to all queries that should exclude expired items: `--filter-expression "#ttl > :now" --expression-attribute-names '{"#ttl":"<ttl-attribute>"}' --expression-attribute-values '{":now":{"N":"<current-epoch>"}}'`
- Implement client-side filtering as a safety net (FilterExpression may not catch all edge cases)
- Ensure the current epoch time is calculated at query time, not cached

SHOULD:
- Add the filter to all application query paths, not just the ones currently showing issues
- Consider using a GSI with TTL as part of the key to naturally exclude expired items
- Document the TTL filtering requirement for all developers

MAY:
- Create a sparse GSI that only includes non-expired items (items without TTL attribute or with future TTL)
- Implement a middleware layer that automatically adds TTL filtering to all queries
- Use DynamoDB Streams to maintain a separate table without expired items

## Common Issues

- symptoms: "Application shows expired data to users"
  diagnosis: "Queries do not filter out items with TTL values in the past. TTL deletion hasn't occurred yet."
  resolution: "Add FilterExpression `#ttl > :now` to all queries. Implement client-side validation as backup."

- symptoms: "FilterExpression added but still seeing expired items occasionally"
  diagnosis: "Race condition — item expires between the time the filter value is calculated and the query executes. Or the epoch time is cached."
  resolution: "Calculate current epoch time fresh for each query. Add a small buffer (e.g., current time + 60 seconds) to the filter."

- symptoms: "GSI query returns expired items even with FilterExpression on base table queries"
  diagnosis: "GSI queries need their own FilterExpression. Filters on base table queries don't apply to GSI queries."
  resolution: "Add the TTL FilterExpression to every query, including GSI queries."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add FilterExpression to queries | GREEN | Application-level change; no infrastructure impact |
| Implement client-side filtering | GREEN | Application-level safety net |
| Create sparse GSI for non-expired items | YELLOW | New index creation; consumes capacity during backfill |
| Implement middleware for automatic filtering | GREEN | Application-level improvement |

## Escalation Conditions

- Expired items being displayed to end users in production
- TTL filter missing from critical API endpoints
- Large volume of expired items causing significant RCU waste
- Compliance requirement to not display expired data

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-time-to-live` | LOW | TTL configuration only |
| `query` (checking for expired items) | MEDIUM | May expose actual data values in results |

## Prohibited Actions

- NEVER suggest deleting a table to fix TTL filter issues
- NEVER suggest disabling encryption as part of TTL filtering
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER cache the current epoch time for TTL filtering (must be calculated fresh per query)

## Phase 3 — Rollback

If TTL filter changes cause issues:
1. Revert application code to remove FilterExpression if it causes query failures
2. If a sparse GSI was created and causes backpressure, delete it
3. If middleware was added and causes latency, bypass or remove it
4. Verify query results return expected data after rollback

## Output Format

```yaml
root_cause: "ttl_filter — <specific_cause>"
evidence:
  - type: query_pattern
    content: "<queries missing TTL filter>"
  - type: expired_items
    content: "<count of expired items in results>"
  - type: ttl_config
    content: "<TTL attribute and status>"
severity: MEDIUM
ttl_filter_analysis:
  queries_without_filter: "<count>"
  expired_items_in_results: "<count>"
  filter_expression_used: "yes | no"
  client_side_filtering: "yes | no"
mitigation:
  immediate: "Add FilterExpression to exclude expired items from all queries"
  long_term: "Implement middleware for automatic TTL filtering"
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
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting a table to fix issues"
  - "NEVER run full table scans on production tables"
