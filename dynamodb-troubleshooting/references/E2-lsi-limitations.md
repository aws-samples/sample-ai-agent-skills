---
title: "E2 — LSI Limitations"
description: "Diagnose Local Secondary Index issues including 10GB partition limit and creation-time constraints"
status: active
severity: MEDIUM
triggers:
  - "LSI limitation"
  - "10GB limit"
  - "ItemCollectionSizeLimitExceededException"
  - "local secondary index"
  - "LSI partition size"
owner: devops-agent
objective: "Identify and resolve LSI-related limitations and errors"
context: "Local Secondary Indexes share the partition key with the base table but use a different sort key. LSIs must be created at table creation time and cannot be added later. The critical limitation is the 10GB item collection size limit: all items with the same partition key (across base table and all LSIs) cannot exceed 10GB. Exceeding this causes ItemCollectionSizeLimitExceededException. Maximum 5 LSIs per table."
---

## Phase 1 — Triage

MUST:
- Check for ItemCollectionSizeLimitExceededException in application logs
- List LSIs on the table: `aws dynamodb describe-table --table-name <table> --query 'Table.LocalSecondaryIndexes[].{IndexName:IndexName,KeySchema:KeySchema,Projection:Projection}'`
- Check item collection sizes: `aws dynamodb query --table-name <table> --key-condition-expression "PK = :pk" --expression-attribute-values '{":pk":{"S":"<value>"}}' --select COUNT --return-consumed-capacity TOTAL`
- Check table size: `aws dynamodb describe-table --table-name <table> --query 'Table.{ItemCount:ItemCount,TableSizeBytes:TableSizeBytes}'`

SHOULD:
- Check item collection metrics on write operations to monitor size
- Estimate item collection sizes for the largest partition keys
- Review if the data model can be restructured to reduce item collection sizes

MAY:
- Query with Select=COUNT to estimate items per partition key
- Check if some LSIs can be replaced with GSIs (no 10GB limit)
- Review if TTL can help keep item collections under 10GB

## Phase 2 — Remediate

MUST:
- If hitting 10GB limit, reduce item collection size by:
  - Archiving old items (delete or move to another table)
  - Reducing item sizes
  - Splitting large partition keys into sub-partitions
- If LSI is not needed, it cannot be removed without recreating the table

SHOULD:
- Monitor item collection sizes proactively with ReturnItemCollectionMetrics on writes
- Consider replacing LSIs with GSIs which have no 10GB limit
- Implement TTL to automatically expire old items

MAY:
- Redesign the table to avoid LSIs entirely (use GSIs instead)
- Implement a data archival strategy to keep item collections small
- Use DynamoDB Streams to maintain a separate table for historical data

## Common Issues

- symptoms: "ItemCollectionSizeLimitExceededException on write operations"
  diagnosis: "Items sharing the same partition key (across base table and all LSIs) exceed 10GB."
  resolution: "Delete old items, reduce item sizes, or split the partition key. Consider migrating to GSIs."

- symptoms: "Cannot add an LSI to an existing table"
  diagnosis: "LSIs can only be created at table creation time."
  resolution: "Create a new table with the desired LSIs and migrate data. Or use a GSI instead."

- symptoms: "LSI query returns stale data"
  diagnosis: "LSIs are always strongly consistent with the base table. If you see stale data, the issue is elsewhere."
  resolution: "Verify the query is hitting the correct LSI. Check if the application is caching old results."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Archive old items (delete expired data) | YELLOW | Permanent data removal; requires careful selection criteria |
| Reduce item sizes | GREEN | Application-level optimization; no data loss |
| Implement TTL | GREEN | Automatic cleanup; non-destructive until items expire |
| Replace LSI with GSI | RED | Requires table recreation and full data migration |
| Redesign table without LSIs | RED | Requires new table creation and data migration |

## Escalation Conditions

- ItemCollectionSizeLimitExceededException on a production write path
- LSI removal required (table recreation needed)
- Item collection approaching 10GB on a high-write table
- Multiple partition keys approaching the 10GB limit simultaneously
- Data archival required but no archival strategy exists

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (LSI config) | MEDIUM | Exposes table schema and index design |
| `query` (item collection size) | MEDIUM | May expose actual data values |
| `scan` (item collection analysis) | HIGH | Directly reads table data |

## Prohibited Actions

- NEVER suggest deleting a table to fix LSI limitations
- NEVER suggest disabling encryption as part of LSI fixes
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER delete items from a production table without explicit approval and backup

## Phase 3 — Rollback

If LSI remediation causes issues:
1. If items were archived/deleted, restore from backup (if available via PITR): `aws dynamodb restore-table-to-point-in-time --source-table-name <table> --target-table-name <table-restored> --restore-date-time <timestamp>`
2. If TTL was enabled and deleting needed items, disable TTL: `aws dynamodb update-time-to-live --table-name <table> --time-to-live-specification Enabled=false,AttributeName=<ttl-attribute>`
3. If a new table was created for migration, redirect traffic back to the original table
4. Verify item collection sizes are within limits after rollback

## Output Format

```yaml
root_cause: "lsi_limitation — <specific_cause>"
evidence:
  - type: lsi_config
    content: "<LSI definitions and projections>"
  - type: item_collection_size
    content: "<estimated size for affected partition keys>"
  - type: error_details
    content: "<ItemCollectionSizeLimitExceededException details>"
severity: MEDIUM
lsi_analysis:
  lsi_count: "<number of LSIs>"
  affected_partition_key: "<key value>"
  estimated_collection_size_gb: "<size>"
  exceeds_10gb_limit: "yes | no"
mitigation:
  immediate: "Archive or delete items to reduce collection size below 10GB"
  long_term: "Migrate to GSIs or redesign partition key strategy"
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
