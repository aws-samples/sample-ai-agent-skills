---
title: "C3 — Pagination Issues"
description: "Diagnose pagination problems with LastEvaluatedKey, 1MB limit, and consistent pagination"
status: active
severity: MEDIUM
triggers:
  - "pagination"
  - "LastEvaluatedKey"
  - "incomplete results"
  - "1MB limit"
  - "missing items in query"
owner: devops-agent
objective: "Resolve pagination issues and ensure complete, consistent result retrieval"
context: "DynamoDB returns up to 1MB of data per Query or Scan call. If more data exists, the response includes LastEvaluatedKey which must be passed as ExclusiveStartKey in the next request. FilterExpressions are applied AFTER the 1MB read, so a page may return fewer items than Limit but still have more pages. Consistent pagination requires handling LastEvaluatedKey correctly in loops."
---

## Phase 1 — Triage

MUST:
- Verify the application checks for LastEvaluatedKey in every response
- Check if FilterExpression is causing apparently empty pages (items read but filtered out)
- Verify ExclusiveStartKey is correctly passed from the previous LastEvaluatedKey
- Test a query to see pagination behavior: `aws dynamodb query --table-name <table> --key-condition-expression "PK = :pk" --expression-attribute-values '{":pk":{"S":"<value>"}}' --limit 10 --return-consumed-capacity TOTAL`

SHOULD:
- Check if the application handles empty pages (FilterExpression can return 0 items but still have LastEvaluatedKey)
- Verify consistent read settings across paginated requests
- Check if Limit is set appropriately for the use case

MAY:
- Test pagination behavior with a small Limit: `aws dynamodb query --table-name <table> --key-condition-expression "PK = :pk" --expression-attribute-values '{":pk":{"S":"<value>"}}' --limit 2 --return-consumed-capacity TOTAL`
- Check if the application correctly handles the case where LastEvaluatedKey is absent (final page)
- Review if parallel pagination is needed for large result sets

## Phase 2 — Remediate

MUST:
- Implement a pagination loop that continues while LastEvaluatedKey is present in the response
- Pass LastEvaluatedKey as ExclusiveStartKey in subsequent requests
- Handle the case where a page returns 0 items but LastEvaluatedKey is still present

SHOULD:
- Use consistent read settings across all pages of a paginated query
- Implement a maximum page count or item count to prevent runaway pagination
- Return LastEvaluatedKey to the client for client-driven pagination (API pagination tokens)

MAY:
- Encode LastEvaluatedKey as a base64 pagination token for API responses
- Implement cursor-based pagination for user-facing applications
- Cache intermediate pagination state for resumable operations

## Common Issues

- symptoms: "Query returns fewer items than expected"
  diagnosis: "Application stops paginating after the first page. LastEvaluatedKey is not checked or used."
  resolution: "Implement a loop that continues requesting pages while LastEvaluatedKey is present in the response."

- symptoms: "Pagination loop runs but returns empty pages"
  diagnosis: "FilterExpression filters out all items on a page. The 1MB of data was read but all items were filtered. LastEvaluatedKey is still present because more data exists."
  resolution: "This is expected behavior. Continue paginating. Consider moving the filter to a KeyConditionExpression or GSI to avoid empty pages."

- symptoms: "Pagination returns duplicate or missing items"
  diagnosis: "Table data changed between pagination requests. DynamoDB pagination is not snapshot-isolated."
  resolution: "For exact-once processing, use a consistent read and track processed items. For most use cases, eventual consistency is acceptable."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix pagination loop (LastEvaluatedKey) | GREEN | Application-level bug fix; no infrastructure change |
| Add maximum page count limit | GREEN | Application-level safety guard |
| Use consistent read settings | GREEN | Application-level configuration |
| Encode LastEvaluatedKey as pagination token | GREEN | Application-level API design change |

## Escalation Conditions

- Pagination bug causing incomplete data retrieval in production
- Runaway pagination consuming excessive RCU on a critical table
- Pagination inconsistency causing data integrity issues in downstream systems
- Client-facing API returning incomplete results due to pagination errors

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `query` (testing pagination) | MEDIUM | May expose actual data values in results |
| `describe-table` | MEDIUM | Exposes table schema and key design |

## Prohibited Actions

- NEVER suggest deleting a table to fix pagination issues
- NEVER suggest disabling encryption as part of pagination fixes
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER expose raw LastEvaluatedKey values in client-facing API responses (encode as opaque token)

## Phase 3 — Rollback

If pagination changes cause issues:
1. Revert application code to previous pagination implementation
2. If consistent read settings were changed, revert to previous consistency mode
3. If maximum page count was set too low, increase the limit or remove temporarily
4. Verify query results match expected data completeness after rollback

## Output Format

```yaml
root_cause: "pagination — <specific_cause>"
evidence:
  - type: query_response
    content: "<LastEvaluatedKey presence and item count>"
  - type: application_code
    content: "<pagination loop implementation>"
  - type: filter_analysis
    content: "<FilterExpression impact on page results>"
severity: MEDIUM
pagination_analysis:
  last_evaluated_key_handled: "yes | no"
  empty_pages_handled: "yes | no"
  filter_expression_used: "yes | no"
  consistent_read: "yes | no"
mitigation:
  immediate: "Fix pagination loop to handle LastEvaluatedKey correctly"
  long_term: "Move filter conditions to key conditions or GSI"
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
