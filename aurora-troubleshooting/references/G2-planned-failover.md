---
title: "G2 — Global Database Planned Failover"
description: "Diagnose Aurora Global Database managed planned failover issues"
status: active
severity: CRITICAL
triggers:
  - "planned failover"
  - "managed failover"
  - "switchover"
  - "Global Database failover"
  - "promote secondary"
owner: devops-agent
objective: "Identify and resolve Aurora Global Database planned failover issues"
context: "Aurora Global Database managed planned failover (switchover) promotes a secondary region to primary with RPO=0 (no data loss). It waits for replication to catch up before switching. The process maintains the global cluster topology. Typical RTO < 1 minute. Different from unplanned failover (detach and promote)."
---

## Phase 1 — Triage

MUST:
- Check global cluster status:
  ```
  aws rds describe-global-clusters --global-cluster-identifier <global-cluster-id>
  ```
- Check replication lag before failover:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name AuroraGlobalDBReplicationLag \
    --dimensions Name=DBClusterIdentifier,Value=<secondary-cluster-id> \
    --start-time <start> --end-time <end> --period 60 --statistics Average Maximum
  ```
- Check failover events:
  ```
  aws rds describe-events --source-identifier <global-cluster-id> --source-type db-cluster --duration 1440
  ```
- Check cluster status in both regions:
  ```
  # Primary region
  aws rds describe-db-clusters --db-cluster-identifier <primary-cluster-id>
  # Secondary region
  aws rds describe-db-clusters --db-cluster-identifier <secondary-cluster-id> --region <secondary-region>
  ```

SHOULD:
- Verify secondary cluster has reader instances that can be promoted
- Check that secondary cluster engine version matches primary
- Verify no pending maintenance on either cluster

MAY:
- Check CloudTrail for the failover API call:
  ```
  aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=FailoverGlobalCluster
  ```
- Review application DNS configuration for cross-region endpoint switching

## Phase 2 — Remediate

MUST:
- For planned failover: use managed planned failover (not detach and promote):
  ```
  aws rds failover-global-cluster --global-cluster-identifier <global-cluster-id> \
    --target-db-cluster-identifier <secondary-cluster-arn>
  ```
- Verify replication lag is low before initiating failover
- Update application endpoints to point to the new primary region

SHOULD:
- Test planned failover in a non-production environment first
- Ensure secondary region has sufficient capacity (instance classes, reader count)
- Implement Route 53 health checks for automatic DNS failover

MAY:
- Use Route 53 Application Recovery Controller for orchestrated failover
- Implement application-level region switching logic

## Common Issues

- symptoms: "Planned failover taking longer than expected"
  diagnosis: "High replication lag. Failover waits for replication to catch up (RPO=0)."
  resolution: "Reduce write throughput on primary. Wait for lag to decrease. Then retry failover."

- symptoms: "Planned failover failed"
  diagnosis: "Secondary cluster not healthy, engine version mismatch, or insufficient capacity."
  resolution: "Check secondary cluster health. Verify engine versions match. Ensure capacity."

- symptoms: "Application not connecting to new primary after failover"
  diagnosis: "Application using hardcoded regional endpoints instead of global endpoint or Route 53."
  resolution: "Update application endpoints. Use Route 53 for automatic DNS failover."

## Safety Ratings
- GREEN: describe-global-clusters, describe-db-clusters, CloudWatch AuroraGlobalDBReplicationLag metrics, describe-events, CloudTrail lookup — read-only inspection
- YELLOW: modify-db-cluster, modify-db-instance — recoverable configuration changes
- RED: failover-global-cluster — high-impact operation that changes primary region, requires careful coordination

## Escalation Conditions
- "Database serves production traffic"
- "Planned failover will change primary region for all applications"
- "Fix requires coordinating application endpoint changes across regions"
- "Replication lag is high before planned failover (increases failover time)"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, global cluster topology, regional endpoint addresses
- HIGH: CloudTrail events for FailoverGlobalCluster (contain API caller identity and parameters)
- MEDIUM: replication lag metrics, cluster status, failover events

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix failover issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest using detach-and-promote (unplanned failover) when planned failover is possible"
- "NEVER suggest failover without verifying replication lag is low"

## Phase 3 — Rollback
- "If planned failover fails, do NOT retry immediately — check cluster status first"
- "If application endpoints are not updated, revert DNS changes"
- "If failover causes issues, wait for stabilization before considering failback"
- "To fail back, perform another planned failover to the original primary region"

## Output Format

```yaml
root_cause: "planned_failover — <specific_cause>"
evidence:
  - type: global_cluster
    content: "<global cluster status>"
  - type: replication_lag
    content: "<AuroraGlobalDBReplicationLag before failover>"
  - type: events
    content: "<failover events>"
severity: CRITICAL
mitigation:
  immediate: "Complete failover and update application endpoints"
  long_term: "Implement automated DNS failover and regular failover testing"
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
