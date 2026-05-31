---
title: "D2 — Provisioned vs On-Demand Mode"
description: "Compare billing modes, switching implications, and cost optimization"
status: active
severity: MEDIUM
triggers:
  - "provisioned vs on-demand"
  - "billing mode"
  - "switch to on-demand"
  - "switch to provisioned"
  - "cost optimization"
owner: devops-agent
objective: "Evaluate and optimize DynamoDB billing mode selection"
context: "Provisioned mode requires specifying RCU/WCU and charges per hour. On-demand mode charges per request with no capacity planning. You can switch between modes once every 24 hours. Provisioned mode with auto-scaling is typically cheaper for predictable workloads. On-demand is better for unpredictable or spiky traffic. Reserved capacity (1-year or 3-year) offers significant discounts on provisioned mode."
---

## Phase 1 — Triage

MUST:
- Check current billing mode: `aws dynamodb describe-table --table-name <table> --query 'Table.BillingModeSummary'`
- Check consumed capacity patterns: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Check write capacity patterns: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedWriteCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Check throttle events: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ThrottledRequests --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 3600 --statistics Sum`

SHOULD:
- Analyze traffic variability — high variance favors on-demand, steady traffic favors provisioned
- Check auto-scaling configuration if in provisioned mode
- Calculate cost comparison based on actual usage patterns
- Check when the last mode switch occurred (24-hour limit): `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateTable --start-time <start> --end-time <end>`

MAY:
- Check reserved capacity purchases: `aws dynamodb describe-reserved-capacity`
- Analyze cost using AWS Cost Explorer for DynamoDB
- Review if different tables should use different billing modes

## Phase 2 — Remediate

MUST:
- Choose the appropriate mode based on workload analysis:
  - Provisioned: predictable traffic, cost-sensitive, steady-state workloads
  - On-demand: unpredictable traffic, new applications, spiky workloads
- Switch billing mode if needed: `aws dynamodb update-table --table-name <table> --billing-mode PAY_PER_REQUEST` or `aws dynamodb update-table --table-name <table> --billing-mode PROVISIONED --provisioned-throughput ReadCapacityUnits=<rcu>,WriteCapacityUnits=<wcu>`
- Note: mode can only be switched once every 24 hours

SHOULD:
- For provisioned mode, always configure auto-scaling
- Monitor costs after switching modes for at least one billing cycle
- Consider reserved capacity for long-term provisioned workloads

MAY:
- Use different billing modes for different tables based on their access patterns
- Implement a cost monitoring dashboard for DynamoDB spending
- Evaluate reserved capacity for 1-year or 3-year commitments

## Common Issues

- symptoms: "On-demand costs are much higher than expected"
  diagnosis: "On-demand charges per request. High-throughput steady workloads are cheaper on provisioned mode with auto-scaling."
  resolution: "Switch to provisioned mode with auto-scaling. On-demand is ~6.5x the per-unit cost of provisioned."

- symptoms: "Cannot switch billing mode"
  diagnosis: "Billing mode can only be changed once every 24 hours."
  resolution: "Wait 24 hours since the last mode change. Plan mode switches carefully."

- symptoms: "Provisioned mode with auto-scaling still causes throttling"
  diagnosis: "Auto-scaling has a 1-2 minute reaction time. Sudden spikes are not handled."
  resolution: "Consider on-demand for spiky workloads, or use scheduled scaling for predictable peaks."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Switch to on-demand mode | YELLOW | Billing mode change; 24-hour cooldown; no data impact |
| Switch to provisioned mode | YELLOW | Billing mode change; requires capacity planning; 24-hour cooldown |
| Configure auto-scaling on provisioned | GREEN | Adds automatic management; no data impact |
| Purchase reserved capacity | GREEN | Cost optimization; no operational impact |

## Escalation Conditions

- Billing mode switch required during active production incident
- Cost increase exceeds 2x after mode switch (unexpected traffic pattern)
- Capacity mode change needed but 24-hour cooldown has not elapsed
- Reserved capacity purchase decision for 1-year or 3-year commitment
- Multiple tables require coordinated billing mode changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (BillingModeSummary) | MEDIUM | Exposes billing mode and capacity configuration |
| `get-metric-statistics` (consumed capacity) | LOW | Operational metrics only |
| `lookup-events` (CloudTrail) | MEDIUM | Exposes modification history and IAM principals |
| `describe-reserved-capacity` | LOW | Cost and reservation details only |

## Prohibited Actions

- NEVER suggest deleting a table to fix billing mode issues
- NEVER suggest disabling encryption to reduce costs
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER switch to provisioned mode without calculating required RCU/WCU first

## Phase 3 — Rollback

If billing mode change causes issues:
1. Switch back to previous mode (note: 24-hour cooldown between switches):
   - To on-demand: `aws dynamodb update-table --table-name <table> --billing-mode PAY_PER_REQUEST`
   - To provisioned: `aws dynamodb update-table --table-name <table> --billing-mode PROVISIONED --provisioned-throughput ReadCapacityUnits=<rcu>,WriteCapacityUnits=<wcu>`
2. If switched to provisioned and throttling occurs, increase capacity immediately before waiting for cooldown
3. If auto-scaling was removed during mode switch, re-enable it
4. Monitor costs and throttle events for at least one billing cycle after rollback

## Output Format

```yaml
root_cause: "billing_mode — <specific_cause>"
evidence:
  - type: current_mode
    content: "<billing mode and configuration>"
  - type: usage_pattern
    content: "<traffic variability and trends>"
  - type: cost_analysis
    content: "<estimated cost comparison>"
severity: MEDIUM
billing_analysis:
  current_mode: "PROVISIONED | PAY_PER_REQUEST"
  recommended_mode: "PROVISIONED | PAY_PER_REQUEST"
  estimated_monthly_cost_provisioned: "<amount>"
  estimated_monthly_cost_ondemand: "<amount>"
  auto_scaling_configured: "yes | no"
mitigation:
  immediate: "Switch billing mode if cost savings are significant"
  long_term: "Implement cost monitoring and periodic billing mode review"
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
