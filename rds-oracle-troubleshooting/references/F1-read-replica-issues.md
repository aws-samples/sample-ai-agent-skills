---
title: "F1 — Read Replica Issues"
description: "Diagnose RDS Oracle read replica creation, lag, and replication failures"
status: active
severity: HIGH
triggers:
  - "read replica"
  - "replica lag"
  - "Active Data Guard"
  - "ReplicaLag"
  - "replica creation failed"
  - "replica sync"
owner: devops-agent
objective: "Resolve read replica issues for RDS Oracle Enterprise Edition"
context: "RDS Oracle read replicas use Active Data Guard and require Enterprise Edition with Active Data Guard license (BYOL). Read replicas are available in same region or cross-region. SE2 does NOT support read replicas. Replica lag is measured via CloudWatch ReplicaLag metric."
---

## Phase 1 — Triage

MUST:
- Verify Oracle edition is EE: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].Engine'`
- Check replica status: `aws rds describe-db-instances --db-instance-identifier <replica-id> --query 'DBInstances[0].{Status:DBInstanceStatus,ReplicaSource:ReadReplicaSourceDBInstanceIdentifier}'`
- Check replica lag: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReplicaLag --dimensions Name=DBInstanceIdentifier,Value=<replica-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check RDS events: `aws rds describe-events --source-identifier <replica-id> --source-type db-instance --duration 1440`

SHOULD:
- Check primary instance status and load
- Verify network connectivity between primary and replica (cross-region)
- Check replica I/O and apply performance:
  ```sql
  -- On replica
  SELECT process, status, thread#, sequence# FROM v$managed_standby;
  ```
- Check for long-running transactions on primary blocking log archival

MAY:
- Check cross-region network latency
- Verify KMS key access for encrypted cross-region replicas
- Check Enhanced Monitoring on both primary and replica

## Phase 2 — Remediate

MUST:
- For creation failures on SE2: read replicas require EE — cannot be created on SE2
- For high replica lag: check primary write load and replica instance class
  - Scale replica to match or exceed primary instance class
  - Reduce write-heavy operations on primary during catch-up
- For replication errors: check events and alert log on replica

SHOULD:
- Create read replica:
  ```
  aws rds create-db-instance-read-replica \
    --db-instance-identifier <replica-id> \
    --source-db-instance-identifier <primary-id> \
    --db-instance-class <instance-class>
  ```
- For cross-region replica:
  ```
  aws rds create-db-instance-read-replica \
    --db-instance-identifier <replica-id> \
    --source-db-instance-identifier arn:aws:rds:<source-region>:<account>:db:<primary-id> \
    --db-instance-class <instance-class> \
    --region <target-region>
  ```
- Monitor ReplicaLag with CloudWatch alarms

MAY:
- Promote replica to standalone for DR: `aws rds promote-read-replica --db-instance-identifier <replica-id>`
- Note: promotion is irreversible — replica becomes independent

## Common Issues

- symptoms: "Cannot create read replica — engine oracle-se2 does not support read replicas"
  diagnosis: "SE2 does not support Active Data Guard / read replicas."
  resolution: "Migrate to Enterprise Edition (BYOL) to use read replicas."

- symptoms: "ReplicaLag increasing steadily"
  diagnosis: "Replica cannot keep up with primary write load. Undersized replica or high write volume."
  resolution: "Scale replica instance class. Reduce primary write load. Check replica I/O metrics."

- symptoms: "Replica in 'error' state"
  diagnosis: "Replication broken. Check events and alert log for specific error."
  resolution: "Delete and recreate the replica. If persistent, check primary for issues."

## Safety Ratings
- GREEN: describe-db-instances, CloudWatch ReplicaLag metrics, describe-events, SELECT from v$managed_standby — read-only inspection
- YELLOW: create-db-instance-read-replica, modify-db-instance — recoverable operations
- RED: promote-read-replica (irreversible), delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Replica lag increasing and affecting read workloads"
- "Fix requires promoting replica (irreversible)"
- "Replica in error state requiring recreation"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, Active Data Guard configuration
- HIGH: v$managed_standby output (reveals replication topology)
- MEDIUM: CloudWatch replica lag metrics, instance class details

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix replica issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest promoting a replica without understanding it is irreversible"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"

## Phase 3 — Rollback
- "If replica promotion was premature, create a new replica from the primary"
- "If replica is in error state, delete and recreate from primary"
- "Restore from snapshot if replica configuration change causes issues"

## Output Format

```yaml
root_cause: "read_replica — <specific_cause>"
evidence:
  - type: replica_status
    content: "<replica status and lag>"
  - type: cloudwatch
    content: "<ReplicaLag metric>"
severity: HIGH
mitigation:
  immediate: "Scale replica or recreate if in error state"
  long_term: "Implement replica lag monitoring and alerting"
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
  - "NEVER suggest making databases publicly accessible"
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting automated backups"
