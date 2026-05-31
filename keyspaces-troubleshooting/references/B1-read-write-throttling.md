---
title: "B1 — Read/Write Throttling"
description: "Diagnose Amazon Keyspaces read and write throttling events"
status: active
severity: HIGH
triggers:
  - "throttling"
  - "ReadThrottleEvents"
  - "WriteThrottleEvents"
  - "capacity exceeded"
  - "throughput exceeded"
  - "request rate too high"
owner: devops-agent
objective: "Identify the cause of throttling and restore normal throughput"
context: "Keyspaces throttles requests when consumed capacity exceeds provisioned capacity (provisioned mode) or when traffic exceeds the adaptive limits (on-demand mode). Throttling can occur at the table level or partition level. Hot partitions are a common cause even when overall capacity is sufficient."
---

## Phase 1 — Triage

MUST:
- Check read throttle events: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check write throttle events: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check consumed vs provisioned capacity:
  - `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
  - `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ConsumedWriteCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check capacity mode: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table> --query 'capacitySpecification'`

SHOULD:
- Check per-connection request rate: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name PerConnectionRequestRateExceeded --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check auto-scaling status if provisioned: `aws application-autoscaling describe-scalable-targets --service-namespace cassandra --resource-ids keyspace/<keyspace>/table/<table>`
- Analyze traffic patterns for spikes

MAY:
- Enable CloudWatch Contributor Insights equivalent for partition-level analysis
- Review application retry logic and backoff strategy

## Phase 2 — Remediate

MUST:
- For provisioned mode: increase RCU/WCU or enable auto-scaling
- For on-demand mode: gradually ramp traffic to allow adaptation
- Implement exponential backoff with jitter in application retry logic

SHOULD:
- Investigate hot partitions if throttling occurs with low overall utilization
- Configure auto-scaling with appropriate target utilization (70% recommended)
- Set up CloudWatch alarms for throttle events

MAY:
- Consider switching capacity mode if current mode doesn't fit traffic pattern
- Redesign partition key for better distribution

## Common Issues

- symptoms: "Throttling despite low overall consumed capacity"
  diagnosis: "Hot partition — single partition receiving disproportionate traffic."
  resolution: "Redesign partition key for even distribution. See B2-hot-partitions runbook."

- symptoms: "Throttling on newly created on-demand table"
  diagnosis: "On-demand starts with 4,000 WCU / 12,000 RCU previous peak. Traffic exceeds 2x this limit."
  resolution: "Gradually increase traffic over time. Or switch to provisioned with known capacity."

- symptoms: "Auto-scaling not responding fast enough"
  diagnosis: "Auto-scaling has cooldown periods and reacts to CloudWatch alarms with delay."
  resolution: "Increase minimum capacity. Reduce scale-in cooldown. Consider on-demand for spiky workloads."

## Output Format

```yaml
root_cause: "throttling — <specific_cause>"
evidence:
  - type: throttle_metrics
    content: "<ReadThrottleEvents and WriteThrottleEvents counts>"
  - type: capacity_metrics
    content: "<consumed vs provisioned capacity>"
  - type: capacity_mode
    content: "<on-demand or provisioned configuration>"
severity: HIGH
mitigation:
  immediate: "Increase capacity or implement backoff retries"
  long_term: "Optimize partition key design and configure auto-scaling"
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
