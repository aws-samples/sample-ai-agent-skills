---
title: "E1 — Read Replica Issues"
description: "Diagnose read replica lag, replication errors, and creation failures"
status: active
severity: HIGH
triggers:
  - "replica lag"
  - "replication error"
  - "Seconds_Behind_Master"
  - "read replica"
---

## Phase 1 — Triage

MUST:
- Check ReplicaLag metric: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReplicaLag --dimensions Name=DBInstanceIdentifier,Value=<replica-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check replication status on replica:
  ```sql
  SHOW REPLICA STATUS\G
  -- Key fields: Seconds_Behind_Source, Replica_IO_Running, Replica_SQL_Running, Last_Error
  ```
- Check source write volume: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteIOPS --dimensions Name=DBInstanceIdentifier,Value=<source-id> ...`
- Verify replica instance class matches or exceeds source

SHOULD:
- Check for long-running queries on replica blocking SQL thread
- Verify `binlog_format=ROW` on source
- Check replica's CPU and IOPS — undersized replicas cause lag
- Check for DDL operations on source (ALTER TABLE causes single-threaded replay)

## Phase 2 — Remediate

- Replica undersized: scale replica to match or exceed source instance class
- Replication error: check Last_Error, may need `CALL mysql.rds_skip_repl_error` (use cautiously)
- High write volume: enable parallel replication `replica_parallel_workers > 0`
- DDL lag: schedule DDL during low-traffic periods
- Broken replication: delete and recreate replica if unrecoverable

## Safety Ratings
- GREEN: CloudWatch ReplicaLag/WriteIOPS metrics, SHOW REPLICA STATUS — read-only inspection
- YELLOW: create-db-instance-read-replica, CALL mysql.rds_skip_repl_error — recoverable but may affect data consistency
- RED: promote-read-replica (irreversible), delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Replica lag increasing and affecting read workloads"
- "Fix requires skipping replication errors (potential data inconsistency)"
- "Replica in error state requiring recreation"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, SHOW REPLICA STATUS (contains replication configuration)
- MEDIUM: CloudWatch replica lag metrics, instance class details

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix replica issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"
- "NEVER suggest skipping replication errors without understanding the data impact"

## Phase 3 — Rollback
- "If replica promotion was premature, create a new replica from the source"
- "If replication skip causes data inconsistency, rebuild replica from snapshot"
- "Restore from snapshot if replica configuration change causes issues"

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
