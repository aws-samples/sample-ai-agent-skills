---
title: "D3 — Burst Capacity"
description: "Understand burst credits, 300-second reserve, and depletion scenarios"
status: active
severity: MEDIUM
triggers:
  - "burst capacity"
  - "burst credits"
  - "300 seconds"
  - "temporary spike handling"
  - "burst depleted"
owner: devops-agent
objective: "Understand and manage DynamoDB burst capacity for handling traffic spikes"
context: "DynamoDB reserves a portion of unused partition throughput for burst capacity. Each partition retains up to 300 seconds of unused read and write capacity. When traffic spikes, burst capacity allows a partition to temporarily exceed its per-second allocation. Once burst credits are depleted, requests are throttled. Burst capacity is not guaranteed and should not be relied upon for sustained traffic."
---

## Phase 1 — Triage

MUST:
- Check for throttle events that indicate burst depletion: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check consumed capacity patterns for spikes: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check provisioned capacity: `aws dynamodb describe-table --table-name <table> --query 'Table.ProvisionedThroughput'`
- Determine if throttling follows a pattern (e.g., after sustained high usage)

SHOULD:
- Analyze traffic patterns to identify burst vs sustained spikes
- Check if auto-scaling is configured to handle sustained increases
- Compare burst duration with the 300-second reserve window

MAY:
- Calculate burst credit consumption rate based on traffic patterns
- Check if the workload would benefit from higher provisioned capacity
- Review if on-demand mode would better handle the traffic pattern

## Phase 2 — Remediate

MUST:
- Do not rely on burst capacity for sustained traffic — increase provisioned capacity
- Configure auto-scaling to handle sustained load increases
- Ensure application implements retry with exponential backoff for throttled requests

SHOULD:
- Increase provisioned capacity to cover expected peak traffic without relying on burst
- Set auto-scaling target utilization lower (e.g., 50-60%) to maintain burst credit reserves
- Monitor consumed capacity to detect when burst credits are being used

MAY:
- Consider on-demand mode for workloads with frequent unpredictable spikes
- Implement application-level rate limiting to smooth out traffic spikes
- Use SQS as a buffer to control write rate during spikes

## Common Issues

- symptoms: "Throttling starts exactly 5 minutes after a traffic spike begins"
  diagnosis: "Burst credits (300 seconds) have been depleted. The spike has lasted longer than the burst reserve."
  resolution: "This is sustained traffic, not a burst. Increase provisioned capacity or enable auto-scaling."

- symptoms: "Intermittent throttling during the day but not at night"
  diagnosis: "Daytime traffic exceeds provisioned capacity. Burst credits are consumed during the day and replenish at night."
  resolution: "Increase provisioned capacity to match daytime traffic. Use auto-scaling or scheduled scaling."

- symptoms: "No throttling on first spike but throttling on second spike shortly after"
  diagnosis: "First spike consumed burst credits. Credits haven't fully replenished before the second spike."
  resolution: "Increase provisioned capacity to handle spikes without relying on burst. Credits replenish at the provisioned rate."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Increase provisioned capacity | GREEN | Non-destructive capacity increase; immediate effect |
| Configure auto-scaling | GREEN | Adds automatic capacity management; no data impact |
| Lower auto-scaling target utilization | GREEN | Increases headroom; preserves burst credits |
| Switch to on-demand mode | YELLOW | Billing mode change; 24-hour cooldown applies |
| Implement application-level rate limiting | GREEN | Application-level change; no infrastructure risk |

## Escalation Conditions

- Burst credits consistently depleted during business hours
- Throttling pattern indicates sustained traffic exceeding provisioned capacity
- Auto-scaling reaction time insufficient for workload spike patterns
- Multiple tables simultaneously exhausting burst credits
- Burst depletion causing cascading failures in dependent services

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (ProvisionedThroughput) | MEDIUM | Exposes capacity configuration |
| `get-metric-statistics` (throttle events) | LOW | Operational metrics only |
| `get-metric-statistics` (consumed capacity) | LOW | Operational metrics only |

## Prohibited Actions

- NEVER suggest deleting a table to fix burst capacity issues
- NEVER suggest disabling encryption to reduce capacity overhead
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER rely on burst capacity for sustained production traffic

## Phase 3 — Rollback

If capacity changes cause issues:
1. Revert provisioned capacity to previous values: `aws dynamodb update-table --table-name <table> --provisioned-throughput ReadCapacityUnits=<previous-rcu>,WriteCapacityUnits=<previous-wcu>`
2. If auto-scaling target utilization was changed, revert to previous value
3. If billing mode was switched, note: 24-hour cooldown before switching back
4. If application-level rate limiting was added, adjust or remove if causing throughput issues
5. Monitor burst credit consumption pattern after rollback

## Output Format

```yaml
root_cause: "burst_capacity — <specific_cause>"
evidence:
  - type: throttle_pattern
    content: "<throttle events timeline>"
  - type: capacity_usage
    content: "<consumed vs provisioned over time>"
  - type: spike_analysis
    content: "<spike duration and magnitude>"
severity: MEDIUM
burst_analysis:
  provisioned_capacity: "<RCU/WCU>"
  peak_consumed: "<RCU/WCU>"
  spike_duration_seconds: "<duration>"
  burst_credits_sufficient: "yes | no"
  auto_scaling_configured: "yes | no"
mitigation:
  immediate: "Increase provisioned capacity to cover peak traffic"
  long_term: "Configure auto-scaling and reduce reliance on burst credits"
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
