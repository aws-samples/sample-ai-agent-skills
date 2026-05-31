---
title: "C1 — Query Performance Issues"
description: "Diagnose and optimize slow DynamoDB queries including key conditions and projections"
status: active
severity: MEDIUM
triggers:
  - "slow query"
  - "query performance"
  - "query latency"
  - "SuccessfulRequestLatency high"
  - "query optimization"
owner: devops-agent
objective: "Identify and resolve DynamoDB query performance bottlenecks"
context: "DynamoDB Query operates on a single partition and returns items matching the partition key and optional sort key condition. Queries return up to 1MB per call. Performance issues arise from scanning too many items, missing key conditions, large item sizes, or not using projections. FilterExpressions run AFTER data is read and do not reduce consumed capacity."
---

## Phase 1 — Triage

MUST:
- Check query latency: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name SuccessfulRequestLatency --dimensions Name=TableName,Value=<table> Name=Operation,Value=Query --start-time <start> --end-time <end> --period 60 --statistics Average`
- Check consumed RCU per query operation: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check returned item count vs scanned count: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReturnedItemCount --dimensions Name=TableName,Value=<table> Name=Operation,Value=Query --start-time <start> --end-time <end> --period 60 --statistics Average`
- Describe table and indexes: `aws dynamodb describe-table --table-name <table> --query 'Table.{KeySchema:KeySchema,GSIs:GlobalSecondaryIndexes[].{Name:IndexName,KeySchema:KeySchema},LSIs:LocalSecondaryIndexes[].{Name:IndexName,KeySchema:KeySchema}}'`

SHOULD:
- Check if FilterExpression is being used instead of KeyConditionExpression (wastes RCU)
- Verify that queries use appropriate indexes: `aws dynamodb describe-table --table-name <table> --query 'Table.GlobalSecondaryIndexes[].{Name:IndexName,KeySchema:KeySchema,Projection:Projection}'`
- Check item sizes — large items consume more RCU per query
- Review if ProjectionExpression is used to limit returned attributes

MAY:
- Test query with ReturnConsumedCapacity to measure actual RCU: `aws dynamodb query --table-name <table> --key-condition-expression "PK = :pk" --expression-attribute-values '{":pk":{"S":"<value>"}}' --return-consumed-capacity INDEXES`
- Check if a GSI or LSI would better serve the query pattern
- Analyze if the sort key condition is sufficiently selective

## Phase 2 — Remediate

MUST:
- Move filter conditions to KeyConditionExpression where possible (sort key conditions)
- Use ProjectionExpression to retrieve only needed attributes
- Ensure queries always specify a partition key in KeyConditionExpression

SHOULD:
- Create GSIs for frequently used query patterns that don't match the base table key
- Use sparse indexes — GSIs only contain items that have the indexed attributes
- Set appropriate Limit values to control page size

MAY:
- Use LSIs for queries that need strongly consistent reads with alternate sort keys
- Implement query result caching with DAX for repeated queries
- Denormalize data to avoid multiple queries for related items

## Common Issues

- symptoms: "Query returns few items but consumes high RCU"
  diagnosis: "FilterExpression is filtering out most items after they are read. RCU is consumed for all items read, not just returned."
  resolution: "Redesign the key schema or create a GSI so the filter condition becomes a key condition."

- symptoms: "Query latency increases over time"
  diagnosis: "Partition is growing — more items share the same partition key, so queries read more data."
  resolution: "Add a more selective sort key condition. Consider time-based partitioning (e.g., `PK#2024-01`)."

- symptoms: "Query on GSI is slower than expected"
  diagnosis: "GSI projection is KEYS_ONLY or INCLUDE but the query needs attributes not in the projection, causing fetches from the base table."
  resolution: "Update GSI projection to include needed attributes, or use ALL projection if many attributes are needed."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add ProjectionExpression | GREEN | Application-level optimization; no infrastructure change |
| Move filter to KeyConditionExpression | GREEN | Application-level query optimization |
| Set Limit values | GREEN | Application-level pagination control |
| Create GSI for query pattern | YELLOW | New index creation; consumes capacity during backfill on large tables |
| Create LSI (new table only) | RED | Requires table recreation and data migration |
| Deploy DAX for query caching | YELLOW | Adds infrastructure dependency; requires application changes |

## Escalation Conditions

- Query performance degradation impacts customer-facing SLA
- GSI creation needed on a table larger than 100GB (extended backfill)
- Query pattern requires LSI but table already exists (table recreation needed)
- Multiple query patterns require conflicting index designs
- SuccessfulRequestLatency exceeds acceptable thresholds for critical operations

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (KeySchema, GSIs, LSIs) | MEDIUM | Exposes table schema and index design |
| `get-metric-statistics` (latency) | LOW | Operational metrics only |
| `query` with ReturnConsumedCapacity | MEDIUM | May expose actual data values in results |

## Prohibited Actions

- NEVER suggest deleting a table to fix query performance
- NEVER suggest disabling encryption to reduce query latency
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER run unbounded queries on production tables without Limit parameter

## Phase 3 — Rollback

If query optimization changes cause issues:
1. Revert application code to previous query patterns
2. If a GSI was created and causes backpressure, delete it: `aws dynamodb update-table --table-name <table> --global-secondary-index-updates '[{"Delete":{"IndexName":"<gsi-name>"}}]'`
3. If DAX was deployed and causes stale reads, bypass DAX and route directly to DynamoDB
4. Verify query latency and RCU consumption return to pre-change baseline

## Output Format

```yaml
root_cause: "query_performance — <specific_cause>"
evidence:
  - type: query_latency
    content: "<SuccessfulRequestLatency metrics>"
  - type: consumed_capacity
    content: "<RCU consumed per query>"
  - type: query_pattern
    content: "<key conditions and filters used>"
severity: MEDIUM
query_analysis:
  operation: "Query"
  key_condition: "<partition key and sort key condition>"
  filter_expression: "<filter if any>"
  projection: "<projected attributes or ALL>"
  index_used: "<base table | GSI name | LSI name>"
  items_scanned_vs_returned: "<scanned> / <returned>"
mitigation:
  immediate: "Add ProjectionExpression and optimize key conditions"
  long_term: "Create appropriate GSIs and redesign key schema"
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
