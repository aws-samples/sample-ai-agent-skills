---
title: "F2 — Serverless Capacity"
description: "Diagnose Neptune Serverless capacity configuration issues"
status: active
severity: MEDIUM
triggers:
  - "serverless capacity"
  - "ncu configuration"
  - "serverless limits"
  - "capacity units"
owner: devops-agent
objective: "Identify and resolve Neptune Serverless capacity configuration issues"
context: "Neptune Serverless capacity is measured in NCUs (Neptune Capacity Units). Proper min/max NCU configuration is critical for performance and cost optimization."
---

## Phase 1 — Triage

MUST:
- Check current configuration: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].ServerlessV2ScalingConfiguration'`
- Monitor capacity usage: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name ServerlessDatabaseCapacity --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Check memory utilization: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name FreeableMemory --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`
- Verify cluster is using serverless instances

SHOULD:
- Analyze NCU consumption patterns over 24-48 hours
- Check for capacity-related errors in cluster events
- Compare NCU usage with query throughput metrics
- Verify engine version supports all needed features in serverless mode

MAY:
- Calculate cost comparison between serverless and provisioned
- Test different min/max NCU configurations
- Review feature limitations of serverless mode

## Phase 2 — Remediate

MUST:
- Configure min/max NCU based on workload analysis
- Verify capacity changes take effect

SHOULD:
- Set min NCU based on minimum acceptable performance
- Set max NCU based on peak workload requirements
- Document capacity decisions and rationale

MAY:
- Implement cost monitoring for NCU consumption
- Consider mixed provisioned/serverless for different workloads

## Common Issues

- symptoms: "Cannot modify serverless configuration"
  diagnosis: "Engine version does not support serverless or modification in progress."
  resolution: "Verify engine version supports serverless. Wait for pending modifications."

- symptoms: "Unexpected high NCU consumption"
  diagnosis: "Inefficient queries consuming more resources than expected."
  resolution: "Profile queries and optimize. Check for full graph scans."

- symptoms: "Feature not available in serverless mode"
  diagnosis: "Some Neptune features are not supported in serverless mode."
  resolution: "Check Neptune documentation for serverless feature support. Use provisioned if needed."

## Output Format

```yaml
root_cause: "serverless_capacity — <specific_cause>"
evidence:
  - type: configuration
    content: "<serverless NCU configuration>"
  - type: capacity_metrics
    content: "<NCU consumption patterns>"
severity: MEDIUM
mitigation:
  immediate: "Adjust capacity configuration"
  long_term: "Optimize queries and monitor NCU patterns"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — capacity management is operational
- Service Disruption Risk: MEDIUM — capacity issues affect query performance
- Compliance Impact: LOW — capacity management is an operational concern

## Escalation Conditions
- Capacity consistently at maximum NCU limit
- Capacity scaling too slow for workload demands
- Unexpected capacity fluctuations causing performance instability
- Cost anomalies related to capacity consumption
- Capacity insufficient for critical batch processing jobs

## Data Sensitivity
- Capacity utilization metrics — INTERNAL
- Cost allocation data — INTERNAL
- Workload patterns — INTERNAL
- Capacity planning projections — INTERNAL
- Never expose capacity details that reveal business volume

## Prohibited Actions
- Do NOT ignore capacity limit warnings on production clusters
- Do NOT reduce maximum capacity during peak workload periods
- Do NOT change capacity settings without cost impact analysis
- Do NOT run capacity-intensive operations without monitoring
- Do NOT bypass capacity alerts
- Do NOT assume serverless capacity is unlimited

## Phase 3 — Rollback
- Restore previous capacity configuration settings
- Revert maximum NCU changes if they caused throttling
- Re-enable capacity monitoring if disabled
- Verify workload performance after capacity changes
- Document capacity utilization patterns
- Update capacity planning based on observed behavior

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
