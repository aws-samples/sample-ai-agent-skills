---
title: "A2 — DynamoDB Write Throttling"
description: "Diagnose WriteThrottleEvents including hot key detection and WCU analysis"
status: active
severity: HIGH
triggers:
  - "WriteThrottleEvents"
  - "write throttling"
  - "ProvisionedThroughputExceededException write"
  - "write capacity exceeded"
owner: devops-agent
objective: "Identify the cause of write throttling and restore healthy write throughput"
context: "DynamoDB throttles writes when consumed WCU exceeds provisioned WCU at the partition level. Each partition supports up to 1,000 WCU. Items up to 1KB consume 1 WCU. Transactional writes consume 2 WCU per item. Hot keys are the most common cause of write throttling even when aggregate capacity appears sufficient."
---

## Phase 1 — Triage

MUST:
- Check WriteThrottleEvents metric: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check consumed vs provisioned WCU: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedWriteCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check provisioned WCU: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ProvisionedWriteCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Describe the table: `aws dynamodb describe-table --table-name <table> --query 'Table.{BillingMode:BillingModeSummary.BillingMode,WriteCapacity:ProvisionedThroughput.WriteCapacityUnits}'`

SHOULD:
- Enable and check Contributor Insights for hot keys: `aws dynamodb update-contributor-insights --table-name <table> --contributor-insights-action ENABLE`
- Check SystemErrors metric for internal failures: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name SystemErrors --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check if GSI write throttling is causing base table throttling (see A3)
- Verify auto-scaling configuration: `aws application-autoscaling describe-scalable-targets --service-namespace dynamodb --resource-ids table/<table>`

MAY:
- Analyze write patterns for temporal hot spots (e.g., batch jobs at specific times)
- Check if conditional writes are causing excessive retries
- Review item sizes to ensure WCU calculations are correct

## Phase 2 — Remediate

MUST:
- If table-level WCU is insufficient, increase capacity: `aws dynamodb update-table --table-name <table> --provisioned-throughput ReadCapacityUnits=<current-rcu>,WriteCapacityUnits=<new-wcu>`
- If hot key detected, implement write sharding (add random suffix to partition key)
- Ensure application retries with exponential backoff on ThrottlingException

SHOULD:
- Configure auto-scaling for WCU: `aws application-autoscaling register-scalable-target --service-namespace dynamodb --resource-id table/<table> --scalable-dimension dynamodb:table:WriteCapacityUnits --min-capacity <min> --max-capacity <max>`
- Reduce item sizes to minimize WCU consumption per write
- Batch writes using BatchWriteItem with backoff for UnprocessedItems

MAY:
- Consider on-demand mode for unpredictable write patterns
- Implement write buffering in the application to smooth out spikes
- Use SQS as a write buffer to control write rate

## Common Issues

- symptoms: "WriteThrottleEvents but consumed WCU is well below provisioned"
  diagnosis: "Hot partition key — writes concentrated on a single partition exceeding its 1,000 WCU limit."
  resolution: "Use Contributor Insights to identify the hot key. Implement write sharding by appending a random number (0-N) to the partition key."

- symptoms: "Write throttling only during specific time windows"
  diagnosis: "Batch jobs or scheduled tasks create write spikes that exceed provisioned capacity."
  resolution: "Spread batch writes over time, use SQS to buffer, or temporarily increase WCU before batch runs."

- symptoms: "Write throttling increased after adding a GSI"
  diagnosis: "GSI writes consume additional WCU. If the GSI is under-provisioned, it causes backpressure on the base table."
  resolution: "Ensure GSI has sufficient WCU. See A3-gsi-backpressure for details."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Increase provisioned WCU | GREEN | Non-destructive capacity increase; immediate effect |
| Implement write sharding | GREEN | Application-level key design change; no infrastructure risk |
| Configure auto-scaling for WCU | GREEN | Adds automatic capacity management; no data impact |
| Reduce item sizes | GREEN | Application-level optimization |
| Switch to on-demand mode | YELLOW | Billing mode change; 24-hour cooldown applies |
| Use SQS as write buffer | YELLOW | Adds new infrastructure dependency; requires application changes |

## Escalation Conditions

- Table-level WCU increase exceeds 2x current provisioned capacity
- Hot key detected on a production table serving critical write path
- Write throttling persists after capacity increase (partition-level bottleneck)
- Contributor Insights reveals a single key consuming >50% of write traffic
- Write throttling impacts data ingestion SLA or causes upstream backpressure

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` | MEDIUM | Exposes table schema, key design, and capacity configuration |
| `get-metric-statistics` (CloudWatch) | LOW | Operational metrics only; no customer data |
| `update-contributor-insights` | MEDIUM | May reveal access patterns and hot key values |
| `describe-scalable-targets` | LOW | Auto-scaling configuration only |

## Prohibited Actions

- NEVER suggest deleting a table to fix write throttling issues
- NEVER suggest disabling encryption to improve write throughput
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER reduce provisioned WCU below current consumed capacity without analysis

## Phase 3 — Rollback

If write capacity changes cause issues:
1. Revert provisioned WCU to previous value: `aws dynamodb update-table --table-name <table> --provisioned-throughput ReadCapacityUnits=<current-rcu>,WriteCapacityUnits=<previous-wcu>`
2. If auto-scaling was modified, restore previous scaling policy with original target utilization
3. If billing mode was switched to on-demand, note: cannot switch back for 24 hours
4. If write sharding was implemented, revert application code and remove sharded keys if needed
5. Verify throttle events return to baseline after rollback

## Output Format

```yaml
root_cause: "write_throttling — <specific_cause>"
evidence:
  - type: cloudwatch_metric
    content: "<WriteThrottleEvents count and pattern>"
  - type: capacity_analysis
    content: "<consumed vs provisioned WCU>"
  - type: contributor_insights
    content: "<hot key data>"
severity: HIGH
throttle_analysis:
  table_level_wcu: "<provisioned>"
  consumed_wcu: "<peak consumed>"
  throttle_events: "<count in period>"
  hot_key_detected: "yes | no"
  hot_key_value: "<key value if detected>"
mitigation:
  immediate: "Increase provisioned WCU or implement write sharding"
  long_term: "Redesign partition key strategy and configure auto-scaling"
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
