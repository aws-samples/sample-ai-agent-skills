---
title: "E3 — Index Projection Issues"
description: "Diagnose fetches from base table due to projection type (ALL vs KEYS_ONLY vs INCLUDE)"
status: active
severity: MEDIUM
triggers:
  - "projection"
  - "fetch from base table"
  - "GSI projection"
  - "KEYS_ONLY"
  - "extra reads"
owner: devops-agent
objective: "Optimize index projections to avoid unnecessary base table fetches"
context: "GSI and LSI projections determine which attributes are copied to the index. KEYS_ONLY projects only key attributes. INCLUDE projects specified non-key attributes. ALL projects all attributes. When a query requests attributes not in the projection, DynamoDB fetches them from the base table (for LSIs) or the query fails to return them (for GSIs). ALL projections increase storage and write costs. KEYS_ONLY minimizes costs but may require additional queries."
---

## Phase 1 — Triage

MUST:
- Check index projections: `aws dynamodb describe-table --table-name <table> --query 'Table.GlobalSecondaryIndexes[].{IndexName:IndexName,Projection:Projection}'`
- Check LSI projections: `aws dynamodb describe-table --table-name <table> --query 'Table.LocalSecondaryIndexes[].{IndexName:IndexName,Projection:Projection}'`
- Identify which attributes the application queries need from each index
- Check if queries on KEYS_ONLY indexes request non-key attributes

SHOULD:
- Compare query attributes with index projection to identify missing attributes
- Check consumed RCU on the base table during GSI/LSI queries (indicates fetches)
- Review if the projection type can be changed (GSIs can be recreated with different projections)

MAY:
- Test query with ReturnConsumedCapacity=INDEXES to see per-index consumption: `aws dynamodb query --table-name <table> --index-name <index> --key-condition-expression "GSI_PK = :pk" --expression-attribute-values '{":pk":{"S":"<value>"}}' --return-consumed-capacity INDEXES`
- Calculate storage cost difference between ALL and INCLUDE projections
- Review if multiple queries could be consolidated with a broader projection

## Phase 2 — Remediate

MUST:
- If queries need attributes not in the projection, either:
  - Recreate the GSI with the needed attributes in the projection
  - Use ProjectionExpression to only request attributes that are in the index
  - Accept the base table fetch cost if it's infrequent

SHOULD:
- Use INCLUDE projection with the specific attributes needed by queries
- Avoid ALL projection unless most attributes are needed (increases storage and write costs)
- Use KEYS_ONLY for indexes used primarily for existence checks or key lookups

MAY:
- Create multiple GSIs with different projections for different query patterns
- Denormalize data to avoid needing many projected attributes
- Use DAX to cache query results and reduce repeated fetches

## Common Issues

- symptoms: "Query on GSI returns items but missing some attributes"
  diagnosis: "GSI projection is KEYS_ONLY or INCLUDE and the requested attributes are not projected."
  resolution: "Add the needed attributes to the GSI projection (requires recreating the GSI) or query the base table."

- symptoms: "LSI query consumes more RCU than expected"
  diagnosis: "LSI projection doesn't include all requested attributes. DynamoDB fetches missing attributes from the base table, consuming additional RCU."
  resolution: "Add needed attributes to the LSI projection (requires table recreation) or accept the extra cost."

- symptoms: "High write costs on a GSI with ALL projection"
  diagnosis: "ALL projection copies every attribute to the GSI. Every base table write that changes any attribute triggers a GSI write."
  resolution: "Change to INCLUDE projection with only the attributes needed for queries on this GSI."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add ProjectionExpression to queries | GREEN | Application-level optimization; no infrastructure change |
| Recreate GSI with INCLUDE projection | YELLOW | Requires GSI deletion and recreation; backfill on large tables |
| Recreate GSI with ALL projection | YELLOW | Increases storage and write costs; requires backfill |
| Deploy DAX for query caching | YELLOW | Adds infrastructure dependency; requires application changes |

## Escalation Conditions

- GSI projection change needed on a table larger than 100GB (extended backfill)
- Multiple GSIs require projection changes simultaneously
- Projection mismatch causing excessive base table fetches on critical read path
- GSI recreation would exceed the 20 GSI per table limit during transition

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (GSI/LSI projections) | MEDIUM | Exposes index design and projected attributes |
| `query` with ReturnConsumedCapacity=INDEXES | MEDIUM | May expose actual data values in results |

## Prohibited Actions

- NEVER suggest deleting a table to fix projection issues
- NEVER suggest disabling encryption to reduce index overhead
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER use ALL projection on a GSI without understanding the write amplification cost

## Phase 3 — Rollback

If projection changes cause issues:
1. If a GSI was deleted for recreation, recreate with the original projection type
2. If application queries were changed, revert to previous query patterns
3. If DAX was deployed and causes issues, bypass DAX and route directly to DynamoDB
4. Verify query results and RCU consumption return to expected levels after rollback

## Output Format

```yaml
root_cause: "projection — <specific_cause>"
evidence:
  - type: index_projection
    content: "<projection type and included attributes>"
  - type: query_attributes
    content: "<attributes requested by queries>"
  - type: consumed_capacity
    content: "<RCU consumed per index and base table>"
severity: MEDIUM
projection_analysis:
  index_name: "<index name>"
  projection_type: "ALL | KEYS_ONLY | INCLUDE"
  projected_attributes: "<list>"
  missing_attributes: "<attributes not in projection>"
  base_table_fetches: "yes | no"
mitigation:
  immediate: "Adjust queries to only request projected attributes"
  long_term: "Recreate index with appropriate projection type"
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
