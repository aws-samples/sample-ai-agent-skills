---
title: "G3 — Global Database Unplanned Failover"
description: "Diagnose Aurora Global Database unplanned failover (detach and promote)"
status: active
severity: CRITICAL
triggers:
  - "unplanned failover"
  - "detach and promote"
  - "region failure"
  - "disaster recovery"
  - "remove from global"
owner: devops-agent
objective: "Execute and troubleshoot Aurora Global Database unplanned failover"
context: "Unplanned failover (detach and promote) is used when the primary region is unavailable. It detaches the secondary cluster from the global cluster and promotes it to a standalone cluster. RPO is typically < 1 second but data loss is possible because cross-region replication is asynchronous. The global cluster topology is broken and must be manually rebuilt."
---

## Phase 1 — Triage

MUST:
- Verify primary region is truly unavailable (not a transient issue)
- Check global cluster status:
  ```
  aws rds describe-global-clusters --global-cluster-identifier <global-cluster-id>
  ```
- Check secondary cluster status:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <secondary-cluster-id> --region <secondary-region>
  ```
- Check last known replication lag (indicates potential data loss):
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name AuroraGlobalDBReplicationLag \
    --dimensions Name=DBClusterIdentifier,Value=<secondary-cluster-id> \
    --start-time <1h-ago> --end-time <now> --period 60 --statistics Maximum \
    --region <secondary-region>
  ```

SHOULD:
- Check AuroraGlobalDBRPOLag for estimated data loss:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name AuroraGlobalDBRPOLag \
    --dimensions Name=DBClusterIdentifier,Value=<secondary-cluster-id> ...
  ```
- Verify secondary cluster has sufficient instances and capacity
- Document the decision to perform unplanned failover (for audit)

MAY:
- Check AWS Health Dashboard for primary region status
- Review Recovery Point Objective (RPO) requirements with stakeholders

## Phase 2 — Remediate

MUST:
- Detach secondary cluster from global cluster:
  ```
  aws rds remove-from-global-cluster --global-cluster-identifier <global-cluster-id> \
    --db-cluster-identifier <secondary-cluster-arn> --region <secondary-region>
  ```
- The detached cluster becomes a standalone regional cluster with read/write capability
- Update application endpoints to point to the promoted cluster
- Verify the promoted cluster is accepting writes

SHOULD:
- After primary region recovers: do NOT re-add the old primary directly
- Create a new global cluster from the promoted cluster and add the old primary region as a new secondary
- Reconcile any data differences between old primary and promoted cluster

MAY:
- Implement Route 53 health checks for automatic DNS failover
- Set up CloudWatch alarms for AuroraGlobalDBReplicationLag and AuroraGlobalDBRPOLag

## Common Issues

- symptoms: "Cannot detach secondary — global cluster in modifying state"
  diagnosis: "Global cluster is in a transitional state."
  resolution: "Wait for the state to stabilize. If primary is truly down, contact AWS Support."

- symptoms: "Data loss after unplanned failover"
  diagnosis: "Asynchronous replication had lag. Transactions committed on primary not yet replicated."
  resolution: "Expected behavior. RPO < 1s typical but not guaranteed. Reconcile data from primary when recovered."

- symptoms: "Cannot rebuild global cluster after failover"
  diagnosis: "Old primary cluster still exists and conflicts with new global cluster."
  resolution: "Delete old primary cluster. Create new global cluster from promoted cluster. Add new secondary."

## Safety Ratings
- GREEN: describe-global-clusters, describe-db-clusters, CloudWatch AuroraGlobalDBReplicationLag/AuroraGlobalDBRPOLag metrics, AWS Health Dashboard — read-only inspection
- YELLOW: verify secondary cluster capacity, document failover decision — preparatory actions
- RED: remove-from-global-cluster (detach and promote) — irreversible operation that breaks global cluster topology, potential data loss

## Escalation Conditions
- "Database serves production traffic"
- "Unplanned failover will result in potential data loss (RPO > 0)"
- "Primary region is unavailable — confirm with AWS Health Dashboard"
- "Fix requires rebuilding global cluster topology after failover"
- "Data loss risk identified — asynchronous replication lag determines RPO"

## Data Sensitivity
- HIGH: database credentials, connection strings, global cluster topology, regional endpoint addresses
- HIGH: replication lag metrics (indicate actual data loss window)
- MEDIUM: cluster status, AWS Health Dashboard information

## Prohibited Actions
- "NEVER suggest unplanned failover without confirming primary region is truly unavailable"
- "NEVER suggest re-adding old primary directly to global cluster after recovery — rebuild instead"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest deleting the old primary cluster before confirming data reconciliation"

## Phase 3 — Rollback
- "Unplanned failover (detach and promote) is NOT reversible — the global cluster topology is broken"
- "After primary region recovers, create a NEW global cluster from the promoted cluster"
- "Reconcile data differences between old primary and promoted cluster before rebuilding"
- "Do NOT attempt to re-add the old primary cluster directly — create a new secondary instead"

## Output Format

```yaml
root_cause: "unplanned_failover — <specific_cause>"
evidence:
  - type: global_cluster
    content: "<global cluster status>"
  - type: replication_lag
    content: "<last known AuroraGlobalDBReplicationLag>"
  - type: rpo_lag
    content: "<AuroraGlobalDBRPOLag indicating potential data loss>"
severity: CRITICAL
mitigation:
  immediate: "Detach and promote secondary. Update application endpoints."
  long_term: "Rebuild global cluster. Implement automated failover with Route 53."
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
