---
title: "A3 — GSI Backpressure on Base Table"
description: "Diagnose GSI throttling causing base table write failures"
status: active
severity: CRITICAL
triggers:
  - "GSI throttling"
  - "GSI backpressure"
  - "base table writes failing"
  - "GSI write throttle"
  - "index throttling"
owner: devops-agent
objective: "Identify and resolve GSI throttling that propagates write failures to the base table"
context: "When a GSI is under-provisioned, DynamoDB throttles writes to the base table to prevent the GSI from falling behind. This is called GSI backpressure. The base table write will be rejected even if the base table has sufficient WCU. Each GSI has its own provisioned capacity independent of the base table. GSI writes are asynchronous but DynamoDB enforces backpressure to keep the GSI in sync."
---

## Phase 1 — Triage

MUST:
- Check GSI throttle events: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> Name=GlobalSecondaryIndexName,Value=<gsi-name> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check base table throttle events: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- List all GSIs and their capacity: `aws dynamodb describe-table --table-name <table> --query 'Table.GlobalSecondaryIndexes[].{IndexName:IndexName,Status:IndexStatus,WriteCapacity:ProvisionedThroughput.WriteCapacityUnits,ReadCapacity:ProvisionedThroughput.ReadCapacityUnits}'`
- Check consumed WCU per GSI: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedWriteCapacityUnits --dimensions Name=TableName,Value=<table> Name=GlobalSecondaryIndexName,Value=<gsi-name> --start-time <start> --end-time <end> --period 60 --statistics Sum`

SHOULD:
- Compare base table WCU with each GSI WCU — GSI should have at least as much WCU as the base table
- Check GSI auto-scaling configuration: `aws application-autoscaling describe-scalable-targets --service-namespace dynamodb --resource-ids table/<table>/index/<gsi-name>`
- Check if the GSI key creates a hot partition (different key distribution than base table)

MAY:
- Review whether all GSIs are necessary — unused GSIs still consume WCU
- Check GSI projection type — ALL projections write more data per GSI update
- Analyze if the GSI key design causes write amplification

## Phase 2 — Remediate

MUST:
- Increase GSI WCU to match or exceed base table write rate: `aws dynamodb update-table --table-name <table> --global-secondary-index-updates '[{"Update":{"IndexName":"<gsi-name>","ProvisionedThroughput":{"ReadCapacityUnits":<rcu>,"WriteCapacityUnits":<new-wcu>}}}]'`
- Ensure GSI auto-scaling is configured with appropriate limits
- Verify GSI status is ACTIVE (not CREATING or UPDATING)

SHOULD:
- Set GSI WCU auto-scaling to match base table scaling: `aws application-autoscaling register-scalable-target --service-namespace dynamodb --resource-id table/<table>/index/<gsi-name> --scalable-dimension dynamodb:index:WriteCapacityUnits --min-capacity <min> --max-capacity <max>`
- Review GSI key design for hot partition potential
- Use KEYS_ONLY or INCLUDE projection instead of ALL to reduce write amplification

MAY:
- Delete unused GSIs to eliminate unnecessary write overhead
- Consider on-demand mode which auto-scales both table and GSI capacity
- Implement write sharding on the GSI partition key if it has hot partitions

## Common Issues

- symptoms: "Base table writes throttled but base table WCU is sufficient"
  diagnosis: "GSI backpressure — one or more GSIs are under-provisioned, causing DynamoDB to throttle base table writes."
  resolution: "Increase GSI WCU to at least match the base table WCU. Enable auto-scaling on all GSIs."

- symptoms: "Throttling after adding a new GSI"
  diagnosis: "New GSI was created with insufficient WCU, or the GSI backfill process is consuming capacity."
  resolution: "Increase GSI WCU. During backfill, the GSI needs extra capacity. Wait for status to become ACTIVE."

- symptoms: "GSI WCU matches base table but GSI still throttles"
  diagnosis: "GSI partition key has worse distribution than base table key, creating hot partitions in the GSI."
  resolution: "Redesign GSI key to improve distribution. Consider composite sort key or write sharding."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Increase GSI WCU | GREEN | Non-destructive capacity increase; immediate effect |
| Configure GSI auto-scaling | GREEN | Adds automatic capacity management; no data impact |
| Use KEYS_ONLY or INCLUDE projection | YELLOW | Requires GSI recreation; causes temporary backfill |
| Delete unused GSIs | RED | Irreversible data removal from index; requires recreation and backfill |
| Switch to on-demand mode | YELLOW | Billing mode change; 24-hour cooldown; affects all GSIs |

## Escalation Conditions

- GSI creation or deletion on tables larger than 100GB (long-running backfill)
- GSI backpressure causing base table write failures in production
- Multiple GSIs simultaneously under-provisioned on a critical table
- GSI WCU increase exceeds 3x current provisioned capacity
- Backpressure impacts upstream services or data pipelines

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (GSI details) | MEDIUM | Exposes table schema, GSI key design, and projections |
| `get-metric-statistics` (GSI metrics) | LOW | Operational metrics only |
| `describe-scalable-targets` (GSI) | LOW | Auto-scaling configuration only |

## Prohibited Actions

- NEVER suggest deleting a table to fix GSI backpressure issues
- NEVER suggest disabling encryption on the table or GSI
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER delete a GSI on a large table without understanding the backfill time for recreation

## Phase 3 — Rollback

If GSI capacity changes cause issues:
1. Revert GSI WCU to previous value: `aws dynamodb update-table --table-name <table> --global-secondary-index-updates '[{"Update":{"IndexName":"<gsi-name>","ProvisionedThroughput":{"ReadCapacityUnits":<rcu>,"WriteCapacityUnits":<previous-wcu>}}}]'`
2. If GSI auto-scaling was modified, restore previous scaling policy and target utilization
3. If a GSI was deleted, recreate it (note: backfill will take time proportional to table size)
4. If billing mode was switched, note: 24-hour cooldown before switching back
5. Monitor both GSI and base table throttle events after rollback

## Output Format

```yaml
root_cause: "gsi_backpressure — <specific_cause>"
evidence:
  - type: gsi_throttle_events
    content: "<GSI WriteThrottleEvents>"
  - type: base_table_throttle_events
    content: "<base table WriteThrottleEvents>"
  - type: capacity_comparison
    content: "<base table WCU vs GSI WCU>"
severity: CRITICAL
gsi_analysis:
  affected_gsi: "<gsi-name>"
  gsi_wcu: "<provisioned>"
  gsi_consumed_wcu: "<peak consumed>"
  base_table_wcu: "<provisioned>"
  backpressure_detected: "yes | no"
mitigation:
  immediate: "Increase GSI WCU to match or exceed base table write rate"
  long_term: "Enable auto-scaling on all GSIs and review GSI key design"
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
