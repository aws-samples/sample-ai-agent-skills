---
title: "E1 — On-Demand vs Provisioned"
description: "Diagnose Amazon Keyspaces capacity mode selection and optimization"
status: active
severity: MEDIUM
triggers:
  - "on-demand vs provisioned"
  - "capacity planning"
  - "cost optimization"
  - "which capacity mode"
  - "billing mode"
owner: devops-agent
objective: "Help select the optimal capacity mode and resolve mode-specific issues"
context: "On-demand mode automatically scales but costs more per request and has initial throughput limits. Provisioned mode is cheaper for predictable workloads but requires capacity planning and auto-scaling configuration. Mode switching has a 24-hour cooldown. On-demand starts with 4,000 WCU / 12,000 RCU previous peak."
---

## Phase 1 — Triage

MUST:
- Check current capacity mode: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table> --query 'capacitySpecification'`
- Analyze traffic patterns: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 3600 --statistics Sum,Maximum`
- Check for throttling: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 3600 --statistics Sum`

SHOULD:
- Calculate average and peak throughput over the last 7 days
- Compare cost estimates for both modes based on actual usage
- Check if auto-scaling is configured for provisioned mode

MAY:
- Review application traffic patterns (steady vs spiky vs unpredictable)
- Analyze cost breakdown in AWS Cost Explorer for Keyspaces

## Phase 2 — Remediate

MUST:
- Choose on-demand for: unpredictable traffic, new applications, development/test
- Choose provisioned for: steady-state traffic, cost-sensitive workloads, predictable patterns
- Wait 24 hours between mode switches

SHOULD:
- Configure auto-scaling for provisioned tables with 70% target utilization
- Set up CloudWatch alarms for capacity monitoring
- Review capacity mode quarterly based on usage trends

MAY:
- Use on-demand for initial launch, switch to provisioned once patterns are established
- Implement different modes for different tables based on their access patterns

## Common Issues

- symptoms: "High costs on on-demand table with steady traffic"
  diagnosis: "On-demand pricing is higher per request than well-provisioned capacity."
  resolution: "Switch to provisioned mode with auto-scaling for steady-state workloads."

- symptoms: "Throttling on provisioned table during traffic spikes"
  diagnosis: "Provisioned capacity insufficient for peak traffic. Auto-scaling too slow."
  resolution: "Increase provisioned capacity or switch to on-demand for spiky workloads."

- symptoms: "Cannot switch mode — 24 hour restriction"
  diagnosis: "Mode was changed within the last 24 hours."
  resolution: "Wait for the 24-hour cooldown. Plan mode switches during maintenance windows."

## Output Format

```yaml
root_cause: "capacity_mode — <specific_cause>"
evidence:
  - type: current_mode
    content: "<on-demand or provisioned configuration>"
  - type: traffic_pattern
    content: "<average and peak throughput analysis>"
  - type: cost_analysis
    content: "<estimated costs for both modes>"
severity: MEDIUM
mitigation:
  immediate: "Adjust capacity mode or settings for current needs"
  long_term: "Implement capacity monitoring and periodic mode review"
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
