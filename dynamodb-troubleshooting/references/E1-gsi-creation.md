---
title: "E1 — GSI Creation Issues"
description: "Diagnose GSI backfill problems, status monitoring, and capacity during creation"
status: active
severity: MEDIUM
triggers:
  - "GSI creation"
  - "GSI backfill"
  - "GSI stuck creating"
  - "GSI status"
  - "index creation slow"
owner: devops-agent
objective: "Monitor and resolve issues during GSI creation and backfill"
context: "When creating a GSI on an existing table, DynamoDB backfills the index with existing data. During backfill, the GSI status is CREATING. Backfill consumes read capacity from the base table and write capacity on the GSI. You can create only one GSI at a time. The table remains fully available during GSI creation. Large tables can take hours to backfill. Maximum 20 GSIs per table."
---

## Phase 1 — Triage

MUST:
- Check GSI status: `aws dynamodb describe-table --table-name <table> --query 'Table.GlobalSecondaryIndexes[].{IndexName:IndexName,IndexStatus:IndexStatus,Backfilling:Backfilling}'`
- Check table status: `aws dynamodb describe-table --table-name <table> --query 'Table.TableStatus'`
- Check consumed capacity during backfill: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check for throttling during backfill: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Monitor backfill progress by checking GSI item count periodically: `aws dynamodb describe-table --table-name <table> --query 'Table.GlobalSecondaryIndexes[].{IndexName:IndexName,ItemCount:ItemCount}'`
- Check if base table reads are being throttled during backfill
- Verify GSI provisioned capacity is sufficient for backfill writes

MAY:
- Estimate backfill time based on table size and GSI write capacity
- Check if other GSI operations are pending (only one at a time)
- Review if the GSI key design is correct before waiting for a long backfill

## Phase 2 — Remediate

MUST:
- Wait for GSI status to become ACTIVE before querying it
- Ensure sufficient capacity on both base table (reads) and GSI (writes) during backfill
- If GSI creation is stuck, check for capacity issues or contact AWS Support

SHOULD:
- Temporarily increase base table RCU and GSI WCU during backfill for faster completion
- Monitor OnlineIndexPercentageProgress if available
- Plan GSI creation during low-traffic periods

MAY:
- Delete and recreate the GSI if the key design is wrong (faster than waiting for a bad backfill)
- Use on-demand mode during GSI creation to avoid capacity planning
- Create GSIs on new tables before loading data to avoid backfill

## Common Issues

- symptoms: "GSI stuck in CREATING status for hours"
  diagnosis: "Large table backfill takes time proportional to table size. Insufficient GSI WCU slows backfill."
  resolution: "Increase GSI WCU temporarily. For very large tables, backfill can take hours. Monitor progress via item count."

- symptoms: "Base table throttling during GSI creation"
  diagnosis: "GSI backfill reads from the base table, consuming RCU."
  resolution: "Increase base table RCU during backfill. The backfill process uses background capacity but can impact the table."

- symptoms: "Cannot create a second GSI"
  diagnosis: "DynamoDB allows only one GSI creation/deletion at a time per table."
  resolution: "Wait for the current GSI operation to complete before starting another."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Increase base table RCU during backfill | GREEN | Non-destructive capacity increase; temporary |
| Increase GSI WCU during backfill | GREEN | Non-destructive capacity increase; temporary |
| Wait for GSI to become ACTIVE | GREEN | No action required; monitoring only |
| Switch to on-demand during creation | YELLOW | Billing mode change; 24-hour cooldown |
| Delete and recreate GSI | RED | Destroys index data; requires full backfill again |

## Escalation Conditions

- GSI creation on a table larger than 100GB (extended backfill duration)
- GSI backfill causing throttling on production read/write operations
- GSI stuck in CREATING status for more than 24 hours
- Multiple GSI operations queued on the same table
- GSI creation blocking other required table modifications

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (GSI status) | MEDIUM | Exposes table schema and index configuration |
| `get-metric-statistics` (capacity during backfill) | LOW | Operational metrics only |

## Prohibited Actions

- NEVER suggest deleting a table to fix GSI creation issues
- NEVER suggest disabling encryption during GSI creation
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER attempt to create a second GSI while one is already being created

## Phase 3 — Rollback

If GSI creation causes issues:
1. If GSI is still in CREATING status and causing problems, delete it: `aws dynamodb update-table --table-name <table> --global-secondary-index-updates '[{"Delete":{"IndexName":"<gsi-name>"}}]'`
2. If base table RCU was increased for backfill, revert to previous value after GSI creation completes or is cancelled
3. If GSI WCU was temporarily increased, revert after backfill completes
4. If billing mode was switched, note: 24-hour cooldown before switching back
5. Verify base table operations return to normal after GSI deletion

## Output Format

```yaml
root_cause: "gsi_creation — <specific_cause>"
evidence:
  - type: gsi_status
    content: "<GSI status and backfill progress>"
  - type: capacity_impact
    content: "<base table and GSI capacity during backfill>"
  - type: table_size
    content: "<table item count and size>"
severity: MEDIUM
gsi_creation_analysis:
  gsi_name: "<index name>"
  gsi_status: "CREATING | ACTIVE | DELETING"
  backfilling: "true | false"
  table_item_count: "<count>"
  estimated_completion: "<estimate>"
mitigation:
  immediate: "Increase capacity during backfill"
  long_term: "Plan GSI creation during low-traffic periods"
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
