---
title: "B1 — Hot Partitions"
description: "Diagnose hot partition issues using partition key distribution analysis and CloudWatch Contributor Insights"
status: active
severity: CRITICAL
triggers:
  - "hot partition"
  - "hot key"
  - "uneven distribution"
  - "partition throttling"
  - "single partition overloaded"
owner: devops-agent
objective: "Identify hot partitions and implement strategies to distribute traffic evenly"
context: "Each DynamoDB partition supports up to 3,000 RCU, 1,000 WCU, and 10GB of data. When a single partition key receives disproportionate traffic, that partition becomes 'hot' and throttles requests even if table-level capacity is sufficient. CloudWatch Contributor Insights reveals the most accessed partition keys. Adaptive capacity can partially mitigate hot partitions but has limits."
---

## Phase 1 — Triage

MUST:
- Enable Contributor Insights: `aws dynamodb update-contributor-insights --table-name <table> --contributor-insights-action ENABLE`
- Check Contributor Insights status: `aws dynamodb describe-contributor-insights --table-name <table>`
- Check throttle events alongside consumed capacity to detect partition-level throttling
- Check ReadThrottleEvents: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check WriteThrottleEvents: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`

SHOULD:
- Analyze the partition key cardinality — low cardinality keys cause hot partitions
- Check table item count and size distribution: `aws dynamodb describe-table --table-name <table> --query 'Table.{ItemCount:ItemCount,TableSizeBytes:TableSizeBytes}'`
- Review the data model for partition key design issues
- Check if specific access patterns concentrate on a few keys

MAY:
- Run a table scan with sampling to analyze key distribution: `aws dynamodb scan --table-name <table> --select COUNT --segment 0 --total-segments 10`
- Use CloudWatch Contributor Insights rules to identify top contributors
- Check if the table has split partitions recently by monitoring partition count trends

## Phase 2 — Remediate

MUST:
- Redesign partition key for better distribution if cardinality is low
- Implement write sharding for hot write keys: append a random suffix (0 to N-1) to the partition key
- For hot read keys, implement a scatter-gather pattern: read from all sharded keys and merge results

SHOULD:
- Use composite partition keys (e.g., `<tenant>#<date>`) to increase cardinality
- Distribute time-series data across partitions by adding a shard prefix
- Cache frequently read hot keys using DAX or application-level caching

MAY:
- Consider using a separate table for high-traffic items
- Implement a fan-out pattern using SQS/SNS for write-heavy hot keys
- Use DynamoDB Streams to maintain denormalized views with better key distribution

## Common Issues

- symptoms: "Throttling despite table having 10x the needed capacity"
  diagnosis: "Single partition key receives majority of traffic. Table-level capacity is irrelevant when one partition is overloaded."
  resolution: "Use Contributor Insights to identify the hot key. Implement write sharding or redesign the partition key."

- symptoms: "Hot partition on a timestamp-based partition key"
  diagnosis: "Using timestamp as partition key causes all current writes to hit the same partition."
  resolution: "Never use monotonically increasing values as partition keys. Use a composite key like `<entity-type>#<shard-id>` with timestamp as sort key."

- symptoms: "Hot partition on a low-cardinality key like status or region"
  diagnosis: "Few distinct partition key values means few partitions, concentrating traffic."
  resolution: "Add a high-cardinality prefix or suffix to the partition key. Use composite keys to increase distribution."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Contributor Insights | GREEN | Read-only analysis; no data impact |
| Implement write sharding | YELLOW | Application-level change; requires code changes and scatter-gather reads |
| Cache hot keys with DAX | YELLOW | Adds infrastructure dependency; requires application changes |
| Redesign partition key | RED | Requires data migration to new table; significant effort and risk |
| Use separate table for high-traffic items | RED | Architectural change; requires application refactoring |

## Escalation Conditions

- Hot partition on a table larger than 100GB (key redesign requires migration)
- Single partition key consuming >70% of total table traffic
- Hot partition causing cascading failures in dependent services
- Partition key redesign required on a table with multiple consumers
- Adaptive capacity insufficient to handle the traffic imbalance

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` | MEDIUM | Exposes table schema and key design |
| `update-contributor-insights` | MEDIUM | Reveals access patterns and hot key values |
| `get-metric-statistics` (throttle events) | LOW | Operational metrics only |
| `scan` (sampling) | HIGH | May expose actual data values in the table |

## Prohibited Actions

- NEVER suggest deleting a table to fix hot partition issues
- NEVER suggest disabling encryption to improve partition throughput
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER expose actual partition key values in logs or reports without data owner approval

## Phase 3 — Rollback

If hot partition remediation causes issues:
1. If write sharding was implemented, revert application code to use original key format
2. If DAX was deployed and causes consistency issues, bypass DAX and route directly to DynamoDB
3. If a separate table was created, redirect traffic back to the original table
4. If capacity was increased to compensate, revert to previous provisioned values
5. Re-enable Contributor Insights to verify traffic patterns after rollback

## Output Format

```yaml
root_cause: "hot_partition — <specific_cause>"
evidence:
  - type: contributor_insights
    content: "<top accessed partition keys>"
  - type: throttle_pattern
    content: "<throttle events vs consumed capacity>"
  - type: key_analysis
    content: "<partition key cardinality and distribution>"
severity: CRITICAL
partition_analysis:
  hot_key_value: "<identified hot key>"
  hot_key_traffic_pct: "<percentage of total traffic>"
  partition_key_cardinality: "low | medium | high"
  adaptive_capacity_active: "yes | no"
mitigation:
  immediate: "Implement write sharding or caching for the hot key"
  long_term: "Redesign partition key strategy for uniform distribution"
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
