---
title: "G1 — Global Database Replication Lag"
description: "Diagnose Aurora Global Database cross-region replication lag"
status: active
severity: HIGH
triggers:
  - "global replication lag"
  - "AuroraGlobalDBReplicationLag"
  - "cross-region lag"
  - "secondary region lag"
  - "global database lag"
owner: devops-agent
objective: "Identify and resolve Aurora Global Database replication lag"
context: "Aurora Global Database replicates data asynchronously across regions. Typical lag is < 1 second. Lag increases with high write throughput on the primary, network latency between regions, or secondary cluster resource constraints. High lag increases RPO for unplanned failover."
---

## Phase 1 — Triage

MUST:
- Check replication lag:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name AuroraGlobalDBReplicationLag \
    --dimensions Name=DBClusterIdentifier,Value=<secondary-cluster-id> \
    --start-time <start> --end-time <end> --period 60 --statistics Average Maximum \
    --region <secondary-region>
  ```
- Check RPO lag:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name AuroraGlobalDBRPOLag \
    --dimensions Name=DBClusterIdentifier,Value=<secondary-cluster-id> ...
  ```
- Check primary cluster write throughput:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeWriteIOPs \
    --dimensions Name=DBClusterIdentifier,Value=<primary-cluster-id> \
    --start-time <start> --end-time <end> --period 300 --statistics Sum
  ```
- Check global cluster status:
  ```
  aws rds describe-global-clusters --global-cluster-identifier <global-cluster-id>
  ```

SHOULD:
- Check secondary cluster instance health:
  ```
  aws rds describe-db-instances --filters Name=db-cluster-id,Values=<secondary-cluster-id> \
    --query 'DBInstances[].{Id:DBInstanceIdentifier,Status:DBInstanceStatus,Class:DBInstanceClass}' \
    --region <secondary-region>
  ```
- Compare primary and secondary instance classes
- Check for network issues between regions

MAY:
- Check AWS Health Dashboard for cross-region connectivity issues
- Review write patterns on primary for optimization opportunities

## Phase 2 — Remediate

MUST:
- If lag is due to high write throughput: optimize write patterns on primary (batch writes, reduce commit frequency)
- If secondary is undersized: scale up secondary cluster instances
- Monitor lag continuously — high lag increases RPO for unplanned failover

SHOULD:
- Set CloudWatch alarms for AuroraGlobalDBReplicationLag (e.g., > 1000ms)
- Set CloudWatch alarms for AuroraGlobalDBRPOLag
- Ensure secondary cluster instance classes match or approach primary

MAY:
- Consider write throttling on primary during critical periods
- Review if Global Database is necessary or if cross-region read replicas suffice

## Common Issues

- symptoms: "AuroraGlobalDBReplicationLag consistently > 1 second"
  diagnosis: "High write throughput on primary or secondary cluster resource constraints."
  resolution: "Optimize writes on primary. Scale up secondary instances."

- symptoms: "Replication lag spikes during batch operations"
  diagnosis: "Large batch writes on primary overwhelm cross-region replication."
  resolution: "Throttle batch writes. Schedule batches during low-traffic periods."

- symptoms: "Replication lag increases over time"
  diagnosis: "Secondary cluster cannot keep up with primary write rate."
  resolution: "Scale up secondary. Reduce primary write rate. Check for network issues."

## Safety Ratings
- GREEN: describe-global-clusters, describe-db-clusters, describe-db-instances, CloudWatch AuroraGlobalDBReplicationLag/AuroraGlobalDBRPOLag/VolumeWriteIOPs metrics — read-only inspection
- YELLOW: modify-db-instance (scale up secondary), modify-db-cluster — recoverable changes
- RED: failover-global-cluster, remove-from-global-cluster, delete-db-cluster — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Global replication lag increasing RPO beyond acceptable threshold"
- "Fix requires scaling secondary cluster instances"
- "Fix requires reducing write throughput on primary"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, global cluster topology (reveals multi-region architecture)
- HIGH: replication lag metrics (indicate RPO and potential data loss window)
- MEDIUM: CloudWatch metrics, instance classes, write throughput patterns

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix replication lag — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest detaching secondary cluster to fix lag — this breaks the global cluster"

## Phase 3 — Rollback
- "Restore from snapshot if configuration change causes issues"
- "If scaling secondary causes issues, scale back down after confirming"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "global_replication_lag — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<AuroraGlobalDBReplicationLag, AuroraGlobalDBRPOLag metrics>"
  - type: write_throughput
    content: "<primary VolumeWriteIOPs>"
  - type: secondary_health
    content: "<secondary cluster instance status>"
severity: HIGH
mitigation:
  immediate: "Reduce primary write throughput or scale up secondary"
  long_term: "Implement lag monitoring and write optimization"
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
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling encryption"
  - "NEVER force failover without understanding impact"
