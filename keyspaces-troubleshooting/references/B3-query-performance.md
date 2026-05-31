---
title: "B3 — Query Performance"
description: "Diagnose Amazon Keyspaces query performance degradation"
status: active
severity: MEDIUM
triggers:
  - "slow query"
  - "query timeout"
  - "high latency"
  - "SELECT performance"
  - "query taking too long"
owner: devops-agent
objective: "Identify causes of query performance issues and optimize query patterns"
context: "Keyspaces query performance depends on partition key usage, result set size, consistency level, and page size. Queries without partition key predicates perform full table scans. Large result sets require pagination. ALLOW FILTERING is restricted and should be avoided."
---

## Phase 1 — Triage

MUST:
- Check query latency metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name SuccessfulRequestLatency --dimensions Name=TableName,Value=<table> Name=Operation,Value=SELECT --start-time <start> --end-time <end> --period 300 --statistics Average,p99`
- Check consumed read capacity for query cost: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify table schema and indexes: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table>`
- Check for system errors: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name SystemErrors --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Review the CQL query for partition key usage
- Check if queries use ALLOW FILTERING (should be avoided)
- Verify page size settings in the client driver
- Check if secondary indexes exist and are being used

MAY:
- Profile query patterns in the application
- Compare latency across different query types (SELECT, INSERT, UPDATE)

## Phase 2 — Remediate

MUST:
- Ensure queries include partition key in WHERE clause
- Remove ALLOW FILTERING and redesign query to use partition key
- Use pagination (TOKEN function or driver auto-pagination) for large result sets

SHOULD:
- Limit result set size with LIMIT clause
- Use clustering key ordering to avoid sorting
- Create secondary indexes for frequently queried non-key columns (with limitations)

MAY:
- Implement application-level caching for frequently accessed data
- Denormalize data model for read-heavy access patterns

## Common Issues

- symptoms: "Query timeout on large table without partition key filter"
  diagnosis: "Full table scan consuming excessive read capacity and exceeding timeout."
  resolution: "Add partition key to WHERE clause. Use TOKEN-based pagination for full scans."

- symptoms: "High latency on queries returning many rows"
  diagnosis: "Large result set without pagination."
  resolution: "Set appropriate page size in driver. Use LIMIT clause. Implement pagination."

- symptoms: "Inconsistent query latency"
  diagnosis: "Variable partition sizes or intermittent throttling."
  resolution: "Check for hot partitions (B2). Ensure even data distribution. Monitor throttle events."

## Output Format

```yaml
root_cause: "query_performance — <specific_cause>"
evidence:
  - type: latency_metrics
    content: "<SuccessfulRequestLatency average and p99>"
  - type: query_pattern
    content: "<CQL query analysis>"
  - type: capacity_consumption
    content: "<read capacity consumed per query>"
severity: MEDIUM
mitigation:
  immediate: "Optimize query to use partition key and pagination"
  long_term: "Redesign data model for efficient query patterns"
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
