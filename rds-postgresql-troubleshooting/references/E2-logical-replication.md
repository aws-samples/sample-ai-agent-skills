---
title: "E2 — Logical Replication"
description: "Diagnose logical replication setup, slot management, and failures"
status: active
severity: HIGH
triggers:
  - "logical replication"
  - "publication"
  - "subscription"
  - "replication slot"
  - "wal_level"
---

## Phase 1 — Triage

MUST:
- Verify logical replication is enabled: check `rds.logical_replication=1` in parameter group
- Check replication slots:
  ```sql
  SELECT slot_name, plugin, slot_type, active, restart_lsn,
         pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
  FROM pg_replication_slots;
  ```
- Check publications: `SELECT * FROM pg_publication;`
- Check subscriptions (on subscriber): `SELECT * FROM pg_subscription;`
- Check WAL disk usage: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name TransactionLogsDiskUsage ...`

SHOULD:
- Check for inactive slots consuming WAL:
  ```sql
  SELECT slot_name, active, pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) / 1024 / 1024 AS lag_mb
  FROM pg_replication_slots WHERE NOT active;
  ```
- Check subscription status: `SELECT * FROM pg_stat_subscription;`
- Verify `max_replication_slots` and `max_wal_senders` are sufficient

## Phase 2 — Remediate

- Enable logical replication: set `rds.logical_replication=1` in parameter group, reboot
- Drop inactive slots: `SELECT pg_drop_replication_slot('<slot_name>');`
- Set safety limit: `max_slot_wal_keep_size` to prevent unbounded WAL growth
- Create publication: `CREATE PUBLICATION my_pub FOR TABLE t1, t2;`
- Create subscription: `CREATE SUBSCRIPTION my_sub CONNECTION 'host=... dbname=... user=...' PUBLICATION my_pub;`
- DDL is NOT replicated — apply schema changes on both sides manually

## Safety Ratings
- GREEN: SELECT from pg_replication_slots/pg_publication/pg_subscription/pg_stat_subscription, SHOW rds.logical_replication — read-only inspection
- YELLOW: modify-db-parameter-group (rds.logical_replication), ALTER SUBSCRIPTION — recoverable but may require reboot
- RED: pg_drop_replication_slot (can cause data loss if slot is needed), delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires enabling rds.logical_replication (needs reboot)"
- "Inactive replication slots causing WAL accumulation and storage growth"
- "Fix requires dropping replication slots"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, subscription connection info (may contain credentials)
- HIGH: replication slot details and WAL positions (reveal replication topology)
- MEDIUM: publication/subscription configuration, WAL retention settings

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix replication issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest dropping active replication slots without confirming downstream impact"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "Restore from snapshot if logical replication parameter change causes issues"
- "Revert parameter group changes (rds.logical_replication) and reboot if needed"
- "If dropping a replication slot was premature, recreate the slot and resync the subscriber"

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
