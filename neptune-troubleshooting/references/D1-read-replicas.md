---
title: "D1 — Read Replicas"
description: "Diagnose Neptune read replica lag and issues"
status: active
severity: MEDIUM
triggers:
  - "replica lag"
  - "read replica"
  - "reader instance"
  - "replication delay"
owner: devops-agent
objective: "Identify and resolve Neptune read replica issues"
context: "Neptune supports up to 15 read replicas within the same region. Replicas share the same storage volume as the writer. Replica lag is typically low but can increase under heavy write load."
---

## Phase 1 — Triage

MUST:
- Check replica lag: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name ClusterReplicaLag --dimensions Name=DBInstanceIdentifier,Value=<replica-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Describe instances: `aws neptune describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id>`
- Check replica status and role
- Verify reader endpoint: `aws neptune describe-db-cluster-endpoints --db-cluster-identifier <cluster-id>`

SHOULD:
- Check writer instance CPU: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name CPUUtilization --dimensions Name=DBInstanceIdentifier,Value=<writer-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Monitor replica CPU and memory
- Check for failover events: `aws neptune describe-events --source-type db-cluster --duration 1440`

MAY:
- Compare query performance across replicas
- Check custom endpoint routing
- Review failover priority settings

## Phase 2 — Remediate

MUST:
- Address high replica lag by reducing write load or scaling writer
- Verify replicas are healthy and available
- Ensure reader endpoint distributes traffic correctly

SHOULD:
- Set failover priority for replicas
- Use custom endpoints for workload isolation
- Monitor replica lag with CloudWatch alarms

MAY:
- Add more replicas for read scaling
- Configure auto-scaling for read replicas

## Common Issues

- symptoms: "High replica lag during bulk writes"
  diagnosis: "Heavy write workload causing replica apply lag."
  resolution: "Reduce write batch size or scale writer instance. Lag should recover after write burst."

- symptoms: "Replica stuck in rebooting state"
  diagnosis: "Replica instance experiencing issues during restart."
  resolution: "Wait for recovery or delete and recreate the replica."

- symptoms: "Stale reads from reader endpoint"
  diagnosis: "Replica lag causing reads to return outdated data."
  resolution: "Use writer endpoint for read-after-write consistency. Monitor ClusterReplicaLag."

## Output Format

```yaml
root_cause: "read_replica — <specific_cause>"
evidence:
  - type: replica_lag
    content: "<ClusterReplicaLag metrics>"
  - type: instances
    content: "<instance status and roles>"
severity: MEDIUM
mitigation:
  immediate: "Address replica lag or health issue"
  long_term: "Set up replica lag monitoring and auto-scaling"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — replica management is operational
- Service Disruption Risk: MEDIUM — replica issues affect read scalability
- Compliance Impact: LOW — replica configuration is an operational concern

## Escalation Conditions
- Read replica falls significantly behind the primary instance
- Replica promotion fails during failover
- All read replicas become unavailable simultaneously
- Replication lag exceeds acceptable thresholds for application consistency
- Replica instance enters error state

## Data Sensitivity
- Replica endpoint addresses — INTERNAL
- Replication lag metrics — INTERNAL
- Failover priority configurations — INTERNAL
- Instance performance metrics — INTERNAL
- Never expose database connection strings externally

## Prohibited Actions
- Do NOT delete all read replicas on a production cluster without approval
- Do NOT promote a replica to primary without understanding the impact
- Do NOT ignore persistent replication lag
- Do NOT route write queries to read replica endpoints
- Do NOT change failover priority during active incidents without coordination
- Do NOT disable replica auto-scaling without a capacity plan

## Phase 3 — Rollback
- Restore deleted read replicas by adding new instances to the cluster
- Revert failover priority changes to previous configuration
- Re-enable replica auto-scaling if disabled
- Verify replication health after any cluster topology changes
- Restore application connection routing to include replicas
- Document replica changes and their impact on read performance

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
