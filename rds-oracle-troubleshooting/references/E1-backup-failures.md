---
title: "E1 — Automated Backup Failures"
description: "Diagnose RDS Oracle automated backup and snapshot failures"
status: active
severity: HIGH
triggers:
  - "backup failed"
  - "snapshot failed"
  - "automated backup"
  - "backup window"
  - "backup retention"
  - "backup storage"
owner: devops-agent
objective: "Identify and resolve automated backup failures on RDS Oracle"
context: "RDS Oracle automated backups use EBS snapshots (not RMAN). Backups occur during the backup window. Backup retention is 0-35 days (0 disables). Backups may fail due to storage issues, I/O suspension during snapshot, or KMS key problems for encrypted instances."
---

## Phase 1 — Triage

MUST:
- Check backup status: `aws rds describe-db-instance-automated-backups --db-instance-identifier <instance-id>`
- Check RDS events for backup failures: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --event-categories backup --duration 4320`
- Check backup retention and window: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].{Retention:BackupRetentionPeriod,Window:PreferredBackupWindow}'`
- Check storage space: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeStorageSpace ...`

SHOULD:
- Check if instance is encrypted and KMS key status: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].{Encrypted:StorageEncrypted,KmsKey:KmsKeyId}'`
- If encrypted, verify KMS key: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.KeyState'`
- Check for I/O-intensive operations during backup window
- Check latest successful snapshot: `aws rds describe-db-snapshots --db-instance-identifier <instance-id> --snapshot-type automated --query 'sort_by(DBSnapshots,&SnapshotCreateTime)[-1]'`

MAY:
- Check CloudTrail for backup-related API calls
- Verify IAM service-linked role for RDS backups

## Phase 2 — Remediate

MUST:
- For disabled backups: enable with retention > 0
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --backup-retention-period 7 \
    --preferred-backup-window "03:00-04:00"
  ```
- For KMS key issues: ensure key is enabled and RDS has access
- For storage full: increase storage before backup can succeed
- Create manual snapshot to verify backup capability:
  ```
  aws rds create-db-snapshot \
    --db-instance-identifier <instance-id> \
    --db-snapshot-identifier <snapshot-name>
  ```

SHOULD:
- Schedule backup window during low-activity period
- Monitor backup completion with CloudWatch Events / EventBridge
- For large databases: ensure backup window is long enough

MAY:
- Implement cross-region backup copy for DR
- Use AWS Backup for centralized backup management

## Common Issues

- symptoms: "Automated backup not occurring"
  diagnosis: "Backup retention set to 0 (disabled)."
  resolution: "Set backup retention period to at least 1 day."

- symptoms: "Backup failed — KMS key not accessible"
  diagnosis: "KMS key disabled, pending deletion, or RDS lost access."
  resolution: "Re-enable KMS key. Check key policy grants RDS access."

- symptoms: "I/O suspension during backup causing application timeouts"
  diagnosis: "Single-AZ instances experience brief I/O suspension during backup snapshot."
  resolution: "Use Multi-AZ (backups taken from standby, no I/O suspension on primary)."

## Safety Ratings
- GREEN: describe-db-instance-automated-backups, describe-events, describe-db-instances, describe-db-snapshots, CloudWatch FreeStorageSpace metrics — read-only inspection
- YELLOW: modify-db-instance (backup retention/window), create-db-snapshot — recoverable changes
- RED: delete-db-snapshot, delete-db-instance — destructive operations, potential data loss

## Escalation Conditions
- "Database serves production traffic"
- "Backup failures affecting recovery point objectives (RPO)"
- "KMS key issues preventing backup operations"
- "Single-AZ instance experiencing I/O suspension during backups"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, KMS key IDs and policies, snapshot ARNs
- HIGH: backup configuration (reveals retention and recovery capabilities)
- MEDIUM: CloudWatch storage metrics, backup window timing

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix backup issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest setting backup retention to 0 on production databases"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"

## Phase 3 — Rollback
- "If backup configuration change causes I/O issues, revert backup window timing"
- "If KMS key was disabled, re-enable immediately to restore backup functionality"
- "Restore from snapshot if backup-related changes cause issues"

## Output Format

```yaml
root_cause: "backup_failure — <specific_cause>"
evidence:
  - type: backup_status
    content: "<automated backup status>"
  - type: rds_events
    content: "<backup-related events>"
severity: HIGH
mitigation:
  immediate: "Fix the specific backup blocker"
  long_term: "Enable Multi-AZ and implement backup monitoring"
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
