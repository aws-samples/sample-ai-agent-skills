---
title: "F1 — Backup and Point-in-Time Recovery"
description: "Diagnose backup failures, snapshot issues, and PITR problems"
status: active
severity: HIGH
triggers:
  - "backup failed"
  - "snapshot"
  - "point-in-time"
  - "PITR"
  - "restore"
---

## Phase 1 — Triage

MUST:
- Check automated backup status: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].[BackupRetentionPeriod,LatestRestorableTime,PreferredBackupWindow]'`
- List snapshots: `aws rds describe-db-snapshots --db-instance-identifier <id> --query 'DBSnapshots[*].[DBSnapshotIdentifier,Status,SnapshotCreateTime]'`
- Check for backup events: `aws rds describe-events --source-identifier <id> --source-type db-instance --event-categories backup --duration 1440`
- Verify PITR window: `LatestRestorableTime` shows the most recent restorable point

SHOULD:
- Check if backup window overlaps with maintenance window
- Verify storage space — backups require sufficient I/O capacity
- Check if instance was in a state that prevents backup (e.g., storage-full)

## Phase 2 — Remediate

- PITR restore: `aws rds restore-db-instance-to-point-in-time --source-db-instance-identifier <id> --target-db-instance-identifier <new-id> --restore-time <ISO-8601>`
- PITR creates a NEW instance — update application connection strings
- Snapshot restore: `aws rds restore-db-instance-from-db-snapshot --db-instance-identifier <new-id> --db-snapshot-identifier <snapshot-id>`
- Backup retention: set 1-35 days via `--backup-retention-period`
- If backups failing: check storage space, I/O capacity, and instance health

## Safety Ratings
- GREEN: describe-db-instance-automated-backups, describe-db-snapshots, describe-events, CloudWatch FreeStorageSpace — read-only inspection
- YELLOW: create-db-snapshot, restore-db-instance-to-point-in-time, modify-db-instance (backup retention) — recoverable operations
- RED: delete-db-snapshot, delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Backup failures affecting recovery point objectives (RPO)"
- "PITR needed for disaster recovery"
- "Fix involves modifying encryption settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, KMS key IDs, snapshot contents (contain full database data)
- MEDIUM: backup configuration, retention periods, restorable time range

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix backup issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest setting backup retention to 0 on production databases"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "If restored instance has issues, delete it and retry with different parameters"
- "Keep original instance running until restored instance is fully validated"
- "Revert backup configuration changes if they impact production performance"

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
