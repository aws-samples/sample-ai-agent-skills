---
title: "B3 — Adaptive Capacity"
description: "Understand how adaptive capacity works, its limits, and monitoring"
status: active
severity: MEDIUM
triggers:
  - "adaptive capacity"
  - "automatic capacity redistribution"
  - "partition capacity rebalancing"
owner: devops-agent
objective: "Verify adaptive capacity is functioning and understand its limitations"
context: "Adaptive capacity is enabled automatically for all DynamoDB tables. It allows individual partitions to burst beyond their fair share of provisioned capacity by borrowing unused capacity from other partitions. Adaptive capacity works in real-time and can handle up to the full table-level throughput on a single partition. However, it cannot exceed the table-level provisioned capacity and does not help if the table itself is under-provisioned."
---

## Phase 1 — Triage

MUST:
- Check if throttling persists despite adaptive capacity (indicates table-level under-provisioning):
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
  - `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check consumed vs provisioned capacity to see if table-level capacity is the bottleneck
- Describe table capacity: `aws dynamodb describe-table --table-name <table> --query 'Table.ProvisionedThroughput'`

SHOULD:
- Enable Contributor Insights to see if traffic is concentrated: `aws dynamodb update-contributor-insights --table-name <table> --contributor-insights-action ENABLE`
- Compare per-partition throttling vs table-level throttling patterns
- Check if the table has been recently scaled down (adaptive capacity needs existing capacity to redistribute)

MAY:
- Review partition count estimates based on table size and throughput
- Check if the workload pattern has changed recently
- Analyze if the hot partition traffic exceeds total table-level capacity

## Phase 2 — Remediate

MUST:
- If throttling persists, increase table-level capacity — adaptive capacity cannot exceed table-level limits
- Ensure the table has sufficient total capacity for the aggregate workload
- Verify that the hot partition traffic does not exceed total table-level provisioned capacity

SHOULD:
- Combine adaptive capacity with good key design for best results
- Monitor Contributor Insights to track if adaptive capacity is handling the imbalance
- Consider on-demand mode which provides adaptive capacity with automatic scaling

MAY:
- Implement application-level caching for the hottest keys
- Use DAX to offload repeated reads from hot partitions
- Redesign the data model if adaptive capacity is consistently at its limits

## Common Issues

- symptoms: "Adaptive capacity is active but throttling still occurs"
  diagnosis: "The hot partition traffic exceeds the total table-level provisioned capacity. Adaptive capacity can only redistribute existing capacity, not create new capacity."
  resolution: "Increase table-level provisioned capacity to at least match the hot partition's needs."

- symptoms: "Brief throttling spikes followed by recovery"
  diagnosis: "Adaptive capacity is working but takes a few minutes to detect and redistribute. Initial spikes are expected."
  resolution: "This is normal behavior. Ensure application retries with backoff. If spikes are too frequent, increase base capacity."

- symptoms: "Adaptive capacity not helping with write-heavy hot partition"
  diagnosis: "Write-heavy partitions are harder to rebalance. Each partition is limited to 1,000 WCU even with adaptive capacity if other partitions also need capacity."
  resolution: "Implement write sharding to distribute writes across multiple partitions."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Increase table-level capacity | GREEN | Non-destructive capacity increase; immediate effect |
| Switch to on-demand mode | YELLOW | Billing mode change; 24-hour cooldown applies |
| Deploy DAX for hot key caching | YELLOW | Adds infrastructure dependency; requires application changes |
| Redesign data model | RED | Requires data migration; significant effort and risk |

## Escalation Conditions

- Adaptive capacity consistently at its limits on a production table
- Hot partition traffic exceeds total table-level provisioned capacity
- Throttling persists despite maximum provisioned capacity
- Multiple tables experiencing adaptive capacity limitations simultaneously
- Performance degradation impacts customer-facing SLAs

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (ProvisionedThroughput) | MEDIUM | Exposes capacity configuration |
| `get-metric-statistics` (throttle events) | LOW | Operational metrics only |
| `update-contributor-insights` | MEDIUM | Reveals access patterns and traffic distribution |

## Prohibited Actions

- NEVER suggest deleting a table to resolve adaptive capacity issues
- NEVER suggest disabling encryption to improve throughput
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER assume adaptive capacity alone can solve fundamental key design problems

## Phase 3 — Rollback

If capacity changes cause issues:
1. Revert provisioned capacity to previous values: `aws dynamodb update-table --table-name <table> --provisioned-throughput ReadCapacityUnits=<previous-rcu>,WriteCapacityUnits=<previous-wcu>`
2. If DAX was deployed and causes consistency issues, bypass DAX and route directly to DynamoDB
3. If billing mode was switched, note: 24-hour cooldown before switching back
4. Verify throttle events and latency return to pre-change baseline

## Output Format

```yaml
root_cause: "adaptive_capacity — <specific_cause>"
evidence:
  - type: throttle_pattern
    content: "<throttle events timeline>"
  - type: capacity_utilization
    content: "<consumed vs provisioned capacity>"
  - type: contributor_insights
    content: "<traffic distribution across keys>"
severity: MEDIUM
adaptive_capacity_analysis:
  table_level_capacity: "<RCU/WCU>"
  hot_partition_traffic: "<estimated RCU/WCU>"
  adaptive_capacity_sufficient: "yes | no"
  table_capacity_sufficient: "yes | no"
mitigation:
  immediate: "Increase table-level capacity if hot partition exceeds total capacity"
  long_term: "Improve key design to reduce reliance on adaptive capacity"
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
