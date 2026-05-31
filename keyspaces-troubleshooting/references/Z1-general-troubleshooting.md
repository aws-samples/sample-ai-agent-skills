---
title: "Z1 — General Keyspaces Troubleshooting"
description: "Catch-all systematic investigation for Amazon Keyspaces issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Keyspaces issue"
  - "Keyspaces error"
  - "Keyspaces problem"
  - "Keyspaces not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Keyspaces issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Amazon Keyspaces issues that don't clearly match a specific runbook. It covers table health checks, capacity analysis, error pattern identification, and systematic elimination of common causes. Use this as a starting point when the issue category is unclear, then pivot to a specific runbook once the problem area is identified."
---

## Phase 1 — Triage

MUST:
- Check keyspace and table status: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table>`
- Check all error metrics:
  - `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name SystemErrors --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
  - `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name UserErrors --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check throttling metrics:
  - `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
  - `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check latency metrics:
  - `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name SuccessfulRequestLatency --dimensions Name=TableName,Value=<table> Name=Operation,Value=SELECT --start-time <start> --end-time <end> --period 300 --statistics Average`
  - `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name SuccessfulRequestLatency --dimensions Name=TableName,Value=<table> Name=Operation,Value=INSERT --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Check CloudTrail for recent changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cassandra.amazonaws.com --max-results 20`
- Check capacity configuration and auto-scaling: `aws application-autoscaling describe-scalable-targets --service-namespace cassandra --resource-ids keyspace/<keyspace>/table/<table>`
- Check TTL and PITR status: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table> --query '{TTL:ttl,PITR:pointInTimeRecovery}'`
- Check consumed capacity: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`

MAY:
- Check VPC endpoint configuration: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.cassandra`
- Check AWS Health Dashboard for service-level issues
- Review application logs for Keyspaces error patterns

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Table issues → A1 (creation), A2 (schema), A3 (capacity mode)
  - Performance → B1 (throttling), B2 (hot partitions), B3 (query performance)
  - CQL issues → C1 (compatibility), C2 (unsupported features)
  - Connectivity → D1 (connection failures), D2 (TLS/SigV4 auth)
  - Capacity → E1 (on-demand vs provisioned), E2 (auto-scaling)
  - TTL → F1 (configuration), F2 (expiration behavior)
  - Backup → G1 (PITR), G2 (snapshot restore)
  - Migration → H1 (Cassandra migration), H2 (CQLReplicator)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes that may have caused the issue
- Verify the fix resolves the issue without introducing new problems

MAY:
- Set up comprehensive monitoring to prevent recurrence
- Create a post-incident report for significant issues

## Common Issues

- symptoms: "Intermittent errors from Keyspaces"
  diagnosis: "SystemErrors metric shows internal service errors. These are transient."
  resolution: "Implement retry with exponential backoff. If persistent, contact AWS Support."

- symptoms: "Application suddenly slower but no errors"
  diagnosis: "Latency increase without errors. Could be capacity limits, hot partitions, or query changes."
  resolution: "Check SuccessfulRequestLatency per operation. Check throttling. Review recent query changes."

- symptoms: "General access denied errors"
  diagnosis: "IAM policy missing required cassandra: actions."
  resolution: "Check IAM policy. Verify SigV4 or service-specific credentials. See D2-tls-sigv4-auth."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: table_status
    content: "<table configuration and health>"
  - type: error_metrics
    content: "<SystemErrors, UserErrors, ThrottleEvents>"
  - type: latency_metrics
    content: "<SuccessfulRequestLatency per operation>"
  - type: capacity_metrics
    content: "<consumed vs provisioned capacity>"
severity: MEDIUM
general_analysis:
  table_status: "ACTIVE | CREATING | UPDATING | DELETING"
  capacity_mode: "PAY_PER_REQUEST | PROVISIONED"
  error_pattern: "<identified pattern>"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and alerting"
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
