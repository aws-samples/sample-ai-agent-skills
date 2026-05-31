---
title: "B2 — Hot Partitions"
description: "Diagnose Amazon Keyspaces hot partition issues causing uneven throughput"
status: active
severity: HIGH
triggers:
  - "hot partition"
  - "uneven distribution"
  - "single partition throttled"
  - "partition key design"
  - "skewed traffic"
owner: devops-agent
objective: "Identify hot partitions and recommend partition key redesign"
context: "Keyspaces distributes data across partitions using the partition key hash. A single hot partition can cause throttling even when overall table capacity is sufficient. This follows the same principles as DynamoDB partition design. Composite partition keys and write sharding can help distribute load."
---

## Phase 1 — Triage

MUST:
- Check throttling with low overall utilization: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check table schema for partition key design: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table>`
- Analyze consumed capacity patterns for spikes: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ConsumedWriteCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`

SHOULD:
- Review application query patterns for partition key usage
- Check if partition key has low cardinality (few unique values)
- Analyze write patterns for temporal clustering on specific keys

MAY:
- Sample queries to identify most-accessed partition keys
- Review data model documentation for access pattern analysis

## Phase 2 — Remediate

MUST:
- Redesign partition key for higher cardinality and even distribution
- Avoid monotonically increasing keys (timestamps, sequential IDs) as sole partition keys
- Use composite partition keys to increase distribution

SHOULD:
- Implement write sharding (add random suffix to partition key)
- Use scatter-gather pattern for reads across sharded partitions
- Test new key design with representative workload before migration

MAY:
- Consider time-bucketing for time-series data
- Implement application-level caching to reduce hot key reads

## Common Issues

- symptoms: "Throttling on specific operations but not others"
  diagnosis: "Specific partition keys receiving disproportionate traffic."
  resolution: "Add a shard suffix to the partition key. Use composite keys for better distribution."

- symptoms: "Sequential ID as partition key causing write hotspot"
  diagnosis: "Monotonically increasing partition keys concentrate writes on a single partition."
  resolution: "Use UUID, hash prefix, or composite key instead of sequential IDs."

- symptoms: "Time-series data with timestamp partition key"
  diagnosis: "All current writes go to the same time-based partition."
  resolution: "Use time-bucketing with a shard component: partition_key = (device_id, time_bucket)."

## Output Format

```yaml
root_cause: "hot_partition — <specific_cause>"
evidence:
  - type: partition_key_design
    content: "<current partition key schema>"
  - type: throttle_pattern
    content: "<throttling with low overall utilization>"
  - type: access_pattern
    content: "<identified hot key pattern>"
severity: HIGH
mitigation:
  immediate: "Increase capacity to absorb hot partition traffic"
  long_term: "Redesign partition key for even distribution"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never drop keyspaces or tables containing production data
- Never modify capacity mode on tables during peak traffic without load testing
- Never alter TTL settings without understanding data retention requirements
- Never change encryption settings on tables with active workloads
- Never disable point-in-time recovery on production tables

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore tables from point-in-time recovery if data is accidentally modified
- Revert capacity mode changes if throttling increases after modification
- Re-enable previous auto-scaling configurations if performance degrades

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
