---
title: "H1 — Global Table Replication Lag"
description: "Diagnose global table replication metrics and ReplicationLatency issues"
status: active
severity: HIGH
triggers:
  - "replication lag"
  - "ReplicationLatency"
  - "global table sync"
  - "cross-region delay"
  - "global table behind"
owner: devops-agent
objective: "Identify and resolve global table replication lag issues"
context: "DynamoDB Global Tables replicate data across AWS regions with typical latency under 1 second. ReplicationLatency metric measures the time for changes to propagate. High replication lag can cause stale reads in replica regions. Lag increases when write rate exceeds replication throughput, during region-level issues, or when replica capacity is insufficient. Global Tables Version 2019.11.21 uses on-demand replication capacity."
---

## Phase 1 — Triage

MUST:
- Check ReplicationLatency metric: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ReplicationLatency --dimensions Name=TableName,Value=<table> Name=ReceivingRegion,Value=<region> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Check PendingReplicationCount: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name PendingReplicationCount --dimensions Name=TableName,Value=<table> Name=ReceivingRegion,Value=<region> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Describe global table: `aws dynamodb describe-table --table-name <table> --query 'Table.Replicas'`
- Check write throttling in the source region

SHOULD:
- Check write throttling in the replica region: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> --region <replica-region> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check if the replica table has sufficient capacity (for provisioned mode)
- Verify all replicas are in ACTIVE status
- Check for AWS service health issues in the receiving region

MAY:
- Compare replication latency across different replica regions
- Check if the source region write rate has increased significantly
- Review if the table uses Global Tables Version 2019.11.21 (recommended)

## Phase 2 — Remediate

MUST:
- If replica is under-provisioned, increase capacity in the replica region
- If using provisioned mode, ensure replica capacity matches or exceeds source write rate
- Verify replica status is ACTIVE and not in an error state

SHOULD:
- Use on-demand mode for global tables to avoid capacity management across regions
- Set up CloudWatch alarms on ReplicationLatency with appropriate thresholds
- Monitor PendingReplicationCount to detect replication backlogs early

MAY:
- Consider reducing write rate to the source table if replication cannot keep up
- Review if all replicas are necessary — remove unused replicas
- Contact AWS Support if replication lag persists despite sufficient capacity

## Common Issues

- symptoms: "ReplicationLatency spikes during peak hours"
  diagnosis: "Replica region capacity is insufficient to handle peak write replication."
  resolution: "Increase replica capacity or switch to on-demand mode. Ensure auto-scaling is configured in all regions."

- symptoms: "Replication lag in one region but not others"
  diagnosis: "Region-specific issue — capacity, network, or service health in that region."
  resolution: "Check replica capacity and AWS Health Dashboard for the affected region."

- symptoms: "Stale reads in replica region"
  diagnosis: "Replication lag means recent writes haven't propagated yet. This is expected with eventual consistency."
  resolution: "For critical reads, read from the source region. Accept eventual consistency for non-critical reads. Monitor ReplicationLatency."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Increase replica capacity | GREEN | Non-destructive capacity increase in replica region |
| Switch to on-demand mode | YELLOW | Billing mode change; affects all regions; 24-hour cooldown |
| Set up CloudWatch alarms | GREEN | Monitoring only; no operational impact |
| Remove unused replicas | RED | Permanently removes cross-region replication; data loss in that region |

## Escalation Conditions

- Replication lag exceeds 5 seconds on a production global table
- PendingReplicationCount growing continuously (replication falling behind)
- Replica in non-ACTIVE status
- Replication lag causing stale reads impacting customer experience
- Cross-region replication failure during a regional incident

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (Replicas) | MEDIUM | Exposes multi-region architecture and configuration |
| `get-metric-statistics` (ReplicationLatency) | LOW | Operational metrics only |
| `get-metric-statistics` (PendingReplicationCount) | LOW | Operational metrics only |

## Prohibited Actions

- NEVER suggest deleting a table to fix replication lag
- NEVER suggest disabling encryption on any replica
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER remove a replica region without confirming no applications depend on it

## Phase 3 — Rollback

If replication changes cause issues:
1. Revert replica capacity to previous values in the affected region
2. If billing mode was switched, note: 24-hour cooldown before switching back
3. If a replica was removed, re-add it: `aws dynamodb update-table --table-name <table> --replica-updates '[{"Create":{"RegionName":"<region>"}}]'` (note: full data sync required)
4. Monitor ReplicationLatency and PendingReplicationCount after rollback

## Output Format

```yaml
root_cause: "replication_lag — <specific_cause>"
evidence:
  - type: replication_latency
    content: "<ReplicationLatency metric>"
  - type: pending_replication
    content: "<PendingReplicationCount>"
  - type: replica_status
    content: "<replica configuration and status>"
severity: HIGH
replication_analysis:
  source_region: "<region>"
  replica_regions: "<list>"
  replication_latency_ms: "<latency per region>"
  pending_replication_count: "<count>"
  replica_capacity_sufficient: "yes | no"
mitigation:
  immediate: "Increase replica capacity or switch to on-demand"
  long_term: "Set up replication monitoring alarms and use on-demand mode"
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
