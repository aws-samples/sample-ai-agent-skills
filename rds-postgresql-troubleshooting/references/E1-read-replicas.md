---
title: "E1 — Read Replica Issues"
description: "Diagnose read replica lag, replication errors, and creation failures"
status: active
severity: HIGH
triggers:
  - "replica lag"
  - "replication"
  - "read replica"
  - "streaming replication"
---

## Phase 1 — Triage

MUST:
- Check ReplicaLag metric: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReplicaLag --dimensions Name=DBInstanceIdentifier,Value=<replica-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check replication status on primary:
  ```sql
  SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
         pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
  FROM pg_stat_replication;
  ```
- Check replica for conflicts:
  ```sql
  SELECT datname, confl_tablespace, confl_lock, confl_snapshot, confl_bufferpin, confl_deadlock
  FROM pg_stat_database_conflicts;
  ```
- Verify replica instance class matches or exceeds primary

SHOULD:
- Check `hot_standby_feedback` setting (prevents query cancellation on replica but may delay vacuum on primary)
- Check `max_standby_streaming_delay` and `max_standby_archive_delay`
- Check TransactionLogsDiskUsage on primary

## Phase 2 — Remediate

- Replica undersized: scale to match primary instance class
- Replication conflicts: increase `max_standby_streaming_delay` or enable `hot_standby_feedback`
- High write volume: replica replay is single-threaded — scale replica I/O
- Broken replication: delete and recreate replica
- Cross-region replica: expect higher lag due to network latency

## Safety Ratings
- GREEN: CloudWatch ReplicaLag metrics, SELECT from pg_stat_replication — read-only inspection
- YELLOW: create-db-instance-read-replica, modify-db-instance — recoverable operations
- RED: promote-read-replica (irreversible), delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Replica lag increasing and affecting read workloads"
- "Fix requires promoting replica (irreversible)"
- "Replica in error state requiring recreation"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, replication configuration
- MEDIUM: CloudWatch replica lag metrics, instance class details

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix replica issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest promoting a replica without understanding it is irreversible"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "If replica promotion was premature, create a new replica from the source"
- "If replica is in error state, delete and recreate from source"
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
