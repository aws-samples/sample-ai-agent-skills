---
title: "A4 — On-Demand Mode Throttling"
description: "Diagnose throttling in on-demand mode including scaling limits, 2x previous peak, and pre-warming"
status: active
severity: HIGH
triggers:
  - "on-demand throttling"
  - "on-demand capacity exceeded"
  - "throttling after switching to on-demand"
  - "new table throttling"
owner: devops-agent
objective: "Identify and resolve throttling in on-demand billing mode"
context: "On-demand mode automatically scales to handle traffic but has limits. A new on-demand table starts with capacity for up to 4,000 WCU and 12,000 RCU. After that, DynamoDB can instantly accommodate up to 2x the previous peak traffic. If traffic exceeds 2x the previous peak, throttling can occur while DynamoDB scales. Tables switched from provisioned to on-demand retain the previous provisioned capacity as the starting point."
---

## Phase 1 — Triage

MUST:
- Confirm the table is in on-demand mode: `aws dynamodb describe-table --table-name <table> --query 'Table.BillingModeSummary'`
- Check throttle events: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check write throttle events: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check consumed capacity trends to identify traffic spikes: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check if the table was recently created or switched to on-demand (initial capacity limits apply)
- Review traffic patterns for sudden spikes exceeding 2x previous peak
- Check if the table was recently switched from provisioned mode: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateTable --start-time <start> --end-time <end>`

MAY:
- Analyze historical traffic patterns to predict capacity needs
- Check if the workload is suitable for on-demand vs provisioned mode
- Review per-partition metrics with Contributor Insights

## Phase 2 — Remediate

MUST:
- If traffic spike is expected, pre-warm the table by temporarily switching to provisioned mode with high capacity, then switch back to on-demand
- Ensure application implements retry with exponential backoff
- Gradually ramp up traffic to allow DynamoDB to scale (double traffic every 30 minutes)

SHOULD:
- For predictable high-traffic events, pre-warm at least 30 minutes before the event
- Pre-warm procedure:
  1. Switch to provisioned mode: `aws dynamodb update-table --table-name <table> --billing-mode PROVISIONED --provisioned-throughput ReadCapacityUnits=<target-rcu>,WriteCapacityUnits=<target-wcu>`
  2. Wait for table to become ACTIVE
  3. Switch back to on-demand: `aws dynamodb update-table --table-name <table> --billing-mode PAY_PER_REQUEST`
- Monitor consumed capacity after pre-warming to confirm scaling

MAY:
- Consider staying on provisioned mode with auto-scaling for predictable workloads
- Use reserved capacity for cost savings on steady-state workloads
- Contact AWS Support for pre-warming assistance on very large tables

## Common Issues

- symptoms: "Throttling on a brand new on-demand table"
  diagnosis: "New on-demand tables start with capacity for 4,000 WCU / 12,000 RCU. Traffic exceeds these initial limits."
  resolution: "Pre-warm by temporarily setting provisioned mode with desired capacity, then switch back to on-demand."

- symptoms: "Throttling after a sudden traffic spike"
  diagnosis: "Traffic exceeded 2x the previous peak. DynamoDB needs time to scale beyond this threshold."
  resolution: "Gradually ramp up traffic. For planned events, pre-warm the table in advance."

- symptoms: "Throttling after switching from provisioned to on-demand"
  diagnosis: "On-demand mode inherits the previous provisioned capacity as its starting point. If traffic immediately exceeds 2x that level, throttling occurs."
  resolution: "Before switching, increase provisioned capacity to at least half the expected peak, then switch to on-demand."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Gradually ramp up traffic | GREEN | No infrastructure change; application-level pacing |
| Implement retry with exponential backoff | GREEN | Application-level resilience; no infrastructure risk |
| Pre-warm via provisioned mode switch | YELLOW | Billing mode change; 24-hour cooldown applies; brief capacity transition |
| Switch to provisioned mode permanently | YELLOW | Billing mode change; requires capacity planning; 24-hour cooldown |

## Escalation Conditions

- On-demand throttling on a newly created production table serving live traffic
- Traffic spike exceeds 4x the previous peak (beyond DynamoDB's instant scaling)
- Pre-warming required for a planned event with >10x normal traffic
- Throttling persists after pre-warming procedure
- Contact AWS Support needed for pre-warming assistance on very large tables

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (BillingModeSummary) | MEDIUM | Exposes billing mode and capacity configuration |
| `get-metric-statistics` (throttle events) | LOW | Operational metrics only |
| `lookup-events` (CloudTrail) | MEDIUM | Exposes table modification history and IAM principals |

## Prohibited Actions

- NEVER suggest deleting a table to fix on-demand throttling
- NEVER suggest disabling encryption to reduce overhead
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER switch from on-demand to provisioned without calculating required capacity first

## Phase 3 — Rollback

If pre-warming or mode switch causes issues:
1. If switched to provisioned for pre-warming, switch back to on-demand: `aws dynamodb update-table --table-name <table> --billing-mode PAY_PER_REQUEST` (note: 24-hour cooldown between switches)
2. If provisioned capacity was set too low during pre-warm, increase immediately before switching back
3. If traffic ramp-up caused application issues, reduce traffic rate and allow DynamoDB to stabilize
4. Verify throttle events subside after rollback

## Output Format

```yaml
root_cause: "on_demand_throttling — <specific_cause>"
evidence:
  - type: billing_mode
    content: "<table billing mode and switch history>"
  - type: throttle_events
    content: "<throttle event count and pattern>"
  - type: traffic_pattern
    content: "<consumed capacity trend>"
severity: HIGH
on_demand_analysis:
  table_age: "<when created or switched>"
  previous_peak_rcu: "<estimated>"
  previous_peak_wcu: "<estimated>"
  current_traffic_rcu: "<current>"
  current_traffic_wcu: "<current>"
  exceeds_2x_peak: "yes | no"
mitigation:
  immediate: "Pre-warm table or gradually ramp up traffic"
  long_term: "Implement pre-warming strategy for planned traffic events"
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
