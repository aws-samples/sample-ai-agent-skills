---
title: "A3 — Capacity Mode Issues"
description: "Diagnose Amazon Keyspaces capacity mode configuration and switching problems"
status: active
severity: MEDIUM
triggers:
  - "capacity mode"
  - "provisioned vs on-demand"
  - "billing mode change"
  - "cannot switch capacity"
  - "throughput mode"
owner: devops-agent
objective: "Resolve capacity mode configuration issues and optimize throughput settings"
context: "Keyspaces tables can use on-demand or provisioned capacity mode. Switching between modes can only be done once every 24 hours. On-demand starts with default limits. Provisioned requires explicit RCU/WCU settings and optionally auto-scaling."
---

## Phase 1 — Triage

MUST:
- Check current capacity mode: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table> --query 'capacitySpecification'`
- Check when last mode switch occurred: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateTable --max-results 10`
- Check current throughput metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Compare consumed vs provisioned capacity if in provisioned mode
- Check for throttling events: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check auto-scaling configuration if provisioned: `aws application-autoscaling describe-scalable-targets --service-namespace cassandra --resource-ids keyspace/<keyspace>/table/<table>`

MAY:
- Estimate costs for both modes based on current usage patterns
- Review traffic patterns to determine optimal mode

## Phase 2 — Remediate

MUST:
- Wait 24 hours between capacity mode switches
- Update capacity mode: `aws keyspaces update-table --keyspace-name <keyspace> --table-name <table> --capacity-specification throughputMode=PAY_PER_REQUEST`
- For provisioned mode, set appropriate RCU/WCU: `aws keyspaces update-table --keyspace-name <keyspace> --table-name <table> --capacity-specification throughputMode=PROVISIONED,readCapacityUnits=100,writeCapacityUnits=100`

SHOULD:
- Configure auto-scaling for provisioned tables
- Set up CloudWatch alarms for throttling events

MAY:
- Use on-demand for unpredictable workloads, provisioned for steady-state
- Implement capacity planning based on historical usage

## Common Issues

- symptoms: "Cannot switch capacity mode — 24 hour restriction"
  diagnosis: "Capacity mode was changed within the last 24 hours."
  resolution: "Wait 24 hours from the last mode switch. Check CloudTrail for the exact timestamp."

- symptoms: "Throttling on newly created on-demand table"
  diagnosis: "On-demand tables start with default limits (4,000 WCU / 12,000 RCU previous peak)."
  resolution: "Gradually increase traffic to allow on-demand to adapt. Or use provisioned mode with known capacity."

- symptoms: "Provisioned capacity too low but auto-scaling not responding"
  diagnosis: "Auto-scaling not configured or cooldown period active."
  resolution: "Configure auto-scaling targets and policies. Check cooldown settings."

## Output Format

```yaml
root_cause: "capacity_mode_issue — <specific_cause>"
evidence:
  - type: capacity_config
    content: "<current capacity mode and settings>"
  - type: throughput_metrics
    content: "<consumed vs provisioned capacity>"
  - type: throttle_events
    content: "<throttling event count>"
severity: MEDIUM
mitigation:
  immediate: "Adjust capacity settings or wait for mode switch cooldown"
  long_term: "Implement auto-scaling and capacity monitoring"
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
