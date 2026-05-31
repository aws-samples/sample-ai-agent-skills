---
title: "A1 — DynamoDB Read Throttling"
description: "Diagnose and resolve ReadThrottleEvents including partition analysis and RCU consumption"
status: active
severity: HIGH
triggers:
  - "ReadThrottleEvents"
  - "read throttling"
  - "ProvisionedThroughputExceededException read"
  - "read capacity exceeded"
owner: devops-agent
objective: "Identify the cause of read throttling and restore healthy read throughput"
context: "DynamoDB throttles reads when consumed RCU exceeds provisioned RCU at the partition level. Each partition supports up to 3,000 RCU. Even if table-level RCU is sufficient, a hot partition can cause throttling. Eventually consistent reads consume half the RCU of strongly consistent reads. Items up to 4KB consume 1 RCU (strongly consistent) or 0.5 RCU (eventually consistent)."
---

## Phase 1 — Triage

MUST:
- Check ReadThrottleEvents metric: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check consumed vs provisioned RCU: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check provisioned RCU: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ProvisionedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Describe the table for current capacity settings: `aws dynamodb describe-table --table-name <table> --query 'Table.{BillingMode:BillingModeSummary.BillingMode,ReadCapacity:ProvisionedThroughput.ReadCapacityUnits,WriteCapacity:ProvisionedThroughput.WriteCapacityUnits}'`

SHOULD:
- Check per-partition throttling with CloudWatch Contributor Insights: `aws dynamodb describe-contributor-insights --table-name <table>`
- Enable Contributor Insights if not active: `aws dynamodb update-contributor-insights --table-name <table> --contributor-insights-action ENABLE`
- Check GSI read throttling: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> Name=GlobalSecondaryIndexName,Value=<gsi-name> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check if auto-scaling is configured: `aws application-autoscaling describe-scalable-targets --service-namespace dynamodb --resource-ids table/<table>`

MAY:
- Analyze access patterns with CloudWatch Contributor Insights for most-accessed keys
- Check if DAX (DynamoDB Accelerator) could offload read traffic
- Review application code for unnecessary strongly consistent reads

## Phase 2 — Remediate

MUST:
- If table-level RCU is insufficient, increase provisioned capacity: `aws dynamodb update-table --table-name <table> --provisioned-throughput ReadCapacityUnits=<new-rcu>,WriteCapacityUnits=<current-wcu>`
- If hot partition detected, review and improve partition key design
- Ensure application uses exponential backoff on throttled requests

SHOULD:
- Switch unnecessary strongly consistent reads to eventually consistent (halves RCU)
- Configure auto-scaling with appropriate target utilization: `aws application-autoscaling register-scalable-target --service-namespace dynamodb --resource-id table/<table> --scalable-dimension dynamodb:table:ReadCapacityUnits --min-capacity <min> --max-capacity <max>`
- Use ProjectionExpression to retrieve only needed attributes (reduces item size and RCU)

MAY:
- Consider switching to on-demand billing mode for unpredictable workloads
- Deploy DAX for read-heavy workloads with repeated access patterns
- Implement application-level caching to reduce DynamoDB reads

## Common Issues

- symptoms: "ReadThrottleEvents spike but ConsumedReadCapacityUnits is below provisioned"
  diagnosis: "Hot partition — a single partition is exceeding its 3,000 RCU limit while table-level capacity appears sufficient."
  resolution: "Use Contributor Insights to identify hot keys. Redesign partition key for better distribution or add a random suffix for write sharding."

- symptoms: "Throttling occurs during batch read operations"
  diagnosis: "BatchGetItem sends too many requests to the same partition simultaneously."
  resolution: "Distribute batch reads across partitions. Process UnprocessedKeys with exponential backoff."

- symptoms: "Read throttling after table restore or migration"
  diagnosis: "Restored table may have different partition layout. Traffic pattern doesn't match new partition distribution."
  resolution: "Gradually ramp up traffic to allow DynamoDB to split partitions. Consider pre-warming with temporary higher capacity."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Increase provisioned RCU | GREEN | Non-destructive capacity increase; takes effect immediately |
| Switch to eventually consistent reads | GREEN | Application-level change; reduces RCU consumption |
| Configure auto-scaling | GREEN | Adds automatic capacity management; no data impact |
| Use ProjectionExpression | GREEN | Application-level optimization; no infrastructure change |
| Switch to on-demand billing mode | YELLOW | Billing mode change; can only switch once per 24 hours |
| Deploy DAX | YELLOW | Adds new infrastructure component; requires application changes |

## Escalation Conditions

- Table-level RCU increase exceeds 2x current provisioned capacity
- Hot partition detected on a table with more than 100GB of data
- Throttling persists after capacity increase (indicates partition-level issue)
- Contributor Insights reveals a single key consuming >50% of traffic
- Read throttling impacts a production SLA or customer-facing service

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` | MEDIUM | Exposes table schema, key design, and capacity configuration |
| `get-metric-statistics` (CloudWatch) | LOW | Operational metrics only; no customer data |
| `describe-contributor-insights` | MEDIUM | May reveal access patterns and hot key values |
| `describe-scalable-targets` | LOW | Auto-scaling configuration only |

## Prohibited Actions

- NEVER suggest deleting a table to fix read throttling issues
- NEVER suggest disabling encryption to reduce read latency
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER reduce provisioned RCU below current consumed capacity without analysis

## Phase 3 — Rollback

If read capacity changes cause issues:
1. Revert provisioned RCU to previous value: `aws dynamodb update-table --table-name <table> --provisioned-throughput ReadCapacityUnits=<previous-rcu>,WriteCapacityUnits=<current-wcu>`
2. If auto-scaling was modified, restore previous scaling policy: `aws application-autoscaling put-scaling-policy --service-namespace dynamodb --resource-id table/<table> --scalable-dimension dynamodb:table:ReadCapacityUnits --policy-name <policy-name> --policy-type TargetTrackingScaling --target-tracking-scaling-policy-configuration '{"TargetValue":<previous-target>,"PredefinedMetricSpecification":{"PredefinedMetricType":"DynamoDBReadCapacityUtilization"}}'`
3. If billing mode was switched to on-demand, note: cannot switch back for 24 hours — plan accordingly
4. Verify throttle events return to baseline after rollback

## Output Format

```yaml
root_cause: "read_throttling — <specific_cause>"
evidence:
  - type: cloudwatch_metric
    content: "<ReadThrottleEvents count and pattern>"
  - type: capacity_analysis
    content: "<consumed vs provisioned RCU>"
  - type: contributor_insights
    content: "<hot key or partition data>"
severity: HIGH
throttle_analysis:
  table_level_rcu: "<provisioned>"
  consumed_rcu: "<peak consumed>"
  throttle_events: "<count in period>"
  hot_partition_detected: "yes | no"
mitigation:
  immediate: "Increase provisioned RCU or switch to on-demand"
  long_term: "Improve partition key design and enable auto-scaling"
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
