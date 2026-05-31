---
title: "B2 — Partition Key Design Issues"
description: "Diagnose and fix partition key design problems including composite keys and write sharding"
status: active
severity: MEDIUM
triggers:
  - "key design"
  - "partition key"
  - "bad key design"
  - "key distribution"
  - "write sharding"
owner: devops-agent
objective: "Evaluate and improve partition key design for optimal data distribution"
context: "Partition key design is the most critical factor in DynamoDB performance. A good partition key has high cardinality, uniform request distribution, and uniform data distribution. Common anti-patterns include using timestamps, status codes, or low-cardinality attributes as partition keys. Composite keys combine multiple attributes to increase cardinality."
---

## Phase 1 — Triage

MUST:
- Describe the table key schema: `aws dynamodb describe-table --table-name <table> --query 'Table.KeySchema'`
- Check table size and item count: `aws dynamodb describe-table --table-name <table> --query 'Table.{ItemCount:ItemCount,TableSizeBytes:TableSizeBytes}'`
- Check for throttling that indicates poor key distribution:
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Enable Contributor Insights to analyze key access patterns: `aws dynamodb update-contributor-insights --table-name <table> --contributor-insights-action ENABLE`
- Sample items to assess partition key cardinality: `aws dynamodb scan --table-name <table> --limit 100 --projection-expression "#pk" --expression-attribute-names '{"#pk":"<partition-key-name>"}' --return-consumed-capacity TOTAL`
- Review GSI key schemas for similar issues: `aws dynamodb describe-table --table-name <table> --query 'Table.GlobalSecondaryIndexes[].{Name:IndexName,KeySchema:KeySchema}'`

MAY:
- Export table data to analyze key distribution offline
- Use CloudWatch Contributor Insights reports to identify access patterns
- Review application queries to understand access patterns

## Phase 2 — Remediate

MUST:
- If partition key has low cardinality, redesign using composite keys
- If using monotonically increasing keys (timestamps, auto-increment), add a high-cardinality prefix
- Document the new key design and migration plan

SHOULD:
- Use composite partition keys: `<tenant-id>#<entity-type>` for multi-tenant applications
- Use sort keys effectively for range queries and hierarchical data
- Implement write sharding for known hot keys: append `#<shard-id>` where shard-id is 0 to N-1

MAY:
- Use DynamoDB Streams to migrate data to a new table with better key design
- Implement a scatter-gather read pattern for sharded keys
- Consider single-table design patterns for related entities

## Common Issues

- symptoms: "Using auto-incrementing ID as partition key"
  diagnosis: "Sequential IDs concentrate writes on the latest partition. Only the most recent partition receives traffic."
  resolution: "Use UUIDs, or prefix with a hash of another attribute. Use the sequential ID as a sort key instead."

- symptoms: "Using date/timestamp as partition key"
  diagnosis: "All current writes go to today's partition. Historical partitions are idle."
  resolution: "Use a different attribute as partition key (e.g., user-id, device-id). Use timestamp as sort key for time-range queries."

- symptoms: "Using status field (ACTIVE/INACTIVE) as partition key"
  diagnosis: "Only 2-3 distinct values means 2-3 partitions. Most items are ACTIVE, creating a massive hot partition."
  resolution: "Use a high-cardinality attribute as partition key. Create a GSI with status as partition key if you need to query by status."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Contributor Insights | GREEN | Read-only analysis; no data impact |
| Implement write sharding for hot keys | YELLOW | Application-level change; requires scatter-gather reads |
| Create GSI with better key design | YELLOW | New index creation; consumes capacity during backfill |
| Redesign partition key (new table) | RED | Requires full data migration; significant effort and downtime risk |
| Use DynamoDB Streams for migration | YELLOW | Adds streaming infrastructure; requires careful orchestration |

## Escalation Conditions

- Key redesign required on a production table larger than 100GB
- Multiple tables affected by the same key design anti-pattern
- Key redesign impacts multiple consuming applications
- Migration plan requires extended maintenance window
- Data model change affects compliance or audit requirements

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (KeySchema) | MEDIUM | Exposes table key design and schema |
| `scan` (sampling partition keys) | HIGH | May expose actual data values and key patterns |
| `update-contributor-insights` | MEDIUM | Reveals access patterns and key distribution |
| `describe-table` (GSI KeySchema) | MEDIUM | Exposes index key design |

## Prohibited Actions

- NEVER suggest deleting a table to fix key design issues
- NEVER suggest disabling encryption as part of key redesign
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER perform a full table scan on a production table without rate limiting

## Phase 3 — Rollback

If key design changes cause issues:
1. If write sharding was implemented, revert application code to original key format
2. If a new GSI was created and causes backpressure, delete the GSI: `aws dynamodb update-table --table-name <table> --global-secondary-index-updates '[{"Delete":{"IndexName":"<gsi-name>"}}]'`
3. If data was migrated to a new table, redirect traffic back to the original table
4. If DynamoDB Streams were enabled for migration, disable after rollback: `aws dynamodb update-table --table-name <table> --stream-specification StreamEnabled=false`
5. Verify application behavior returns to pre-change baseline

## Output Format

```yaml
root_cause: "key_design — <specific_cause>"
evidence:
  - type: key_schema
    content: "<current partition key and sort key>"
  - type: cardinality_analysis
    content: "<distinct key count and distribution>"
  - type: access_pattern
    content: "<how the table is queried>"
severity: MEDIUM
key_design_analysis:
  partition_key: "<attribute name and type>"
  sort_key: "<attribute name and type or none>"
  cardinality: "low | medium | high"
  distribution: "uniform | skewed | highly_skewed"
  anti_pattern: "<identified anti-pattern or none>"
mitigation:
  immediate: "Implement write sharding for hot keys"
  long_term: "Redesign partition key with composite key strategy"
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
