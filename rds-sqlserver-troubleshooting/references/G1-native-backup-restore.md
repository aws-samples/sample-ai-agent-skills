---
title: "G1 — Native Backup Restore Migration"
description: "Diagnose native backup restore migration issues on RDS SQL Server"
status: active
severity: MEDIUM
triggers:
  - "migration"
  - "native restore"
  - "backup restore migration"
  - "on-premises to RDS"
  - "rds_restore_database"
  - "compatibility level"
owner: devops-agent
objective: "Resolve native backup restore migration issues when migrating to RDS SQL Server"
context: "Native backup restore is the primary migration method for SQL Server to RDS. Upload .bak file to S3, then use msdb.dbo.rds_restore_database. The source SQL Server version must be equal to or lower than the target RDS version. Backup must be FULL (not differential or log). Database compatibility level may need adjustment."
---

## Phase 1 — Triage

MUST:
- Check restore task status:
  ```sql
  EXEC msdb.dbo.rds_task_status @db_name='<database>';
  ```
- Verify source backup version compatibility:
  ```sql
  -- On RDS, check engine version
  SELECT @@VERSION;
  SELECT SERVERPROPERTY('ProductVersion') AS version, SERVERPROPERTY('Edition') AS edition;
  ```
- Check option group has SQLSERVER_BACKUP_RESTORE:
  ```
  aws rds describe-option-groups --option-group-name <option-group> \
    --query "OptionGroupsList[0].Options[?OptionName=='SQLSERVER_BACKUP_RESTORE']"
  ```
- Verify S3 bucket and IAM role access

SHOULD:
- Check backup file details before restore:
  ```sql
  EXEC msdb.dbo.rds_restore_database
    @restore_db_name='MyDatabase',
    @s3_arn_to_restore_from='arn:aws:s3:::mybucket/MyDatabase.bak',
    @with_norecovery=0;
  ```
- Verify the backup is a FULL backup (not differential or log-only)
- Check for features not supported on RDS in the source database
- Verify sufficient storage on the RDS instance for the restored database

MAY:
- Check for orphaned users after restore:
  ```sql
  USE [MyDatabase];
  EXEC sp_change_users_login 'Report';
  ```
- Verify database compatibility level after restore

## Phase 2 — Remediate

MUST:
- For version mismatch: ensure RDS SQL Server version >= source version
- For restore failures: check rds_task_status for specific error message
- Fix orphaned users after migration:
  ```sql
  USE [MyDatabase];
  -- Map orphaned user to existing login
  ALTER USER [username] WITH LOGIN = [username];
  -- Or create new login and map
  CREATE LOGIN [username] WITH PASSWORD = 'StrongPassword123!';
  ALTER USER [username] WITH LOGIN = [username];
  ```

SHOULD:
- Update database compatibility level to match target version:
  ```sql
  ALTER DATABASE [MyDatabase] SET COMPATIBILITY_LEVEL = 150;  -- SQL Server 2019
  ```
- Rebuild indexes after migration for optimal performance:
  ```sql
  USE [MyDatabase];
  EXEC sp_MSforeachtable 'ALTER INDEX ALL ON ? REBUILD';
  ```
- Update statistics:
  ```sql
  USE [MyDatabase];
  EXEC sp_updatestats;
  ```

MAY:
- Use multi-file backup for faster transfer:
  ```sql
  -- On source (on-premises)
  BACKUP DATABASE [MyDatabase] TO
    DISK = 'C:\backup\MyDB_1.bak',
    DISK = 'C:\backup\MyDB_2.bak',
    DISK = 'C:\backup\MyDB_3.bak',
    DISK = 'C:\backup\MyDB_4.bak'
  WITH COMPRESSION;
  ```
- Validate data integrity after restore: `DBCC CHECKDB('MyDatabase') WITH NO_INFOMSGS;`

## Common Issues

- symptoms: "Restore fails — backup from newer SQL Server version"
  diagnosis: "Cannot restore a backup from a newer version to an older version."
  resolution: "Upgrade the RDS instance to match or exceed the source SQL Server version."

- symptoms: "Restore fails — differential or log backup"
  diagnosis: "RDS native restore only supports FULL backups for initial restore."
  resolution: "Create a FULL backup on the source and upload to S3."

- symptoms: "Orphaned users after restore"
  diagnosis: "Logins on source server don't exist on RDS instance."
  resolution: "Create matching logins on RDS and map users with ALTER USER ... WITH LOGIN."

- symptoms: "Features not available after restore"
  diagnosis: "Source database uses features not supported on RDS (e.g., FILESTREAM, Service Broker cross-instance)."
  resolution: "Remove unsupported features before backup or redesign for RDS compatibility."

## Safety Ratings
- GREEN: SELECT from msdb.dbo.rds_task_status, describe-option-groups — read-only inspection
- YELLOW: msdb.dbo.rds_restore_database, msdb.dbo.rds_backup_database — recoverable operations
- RED: delete-db-instance, dropping databases — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Native restore failing — migration timeline at risk"
- "Fix requires IAM role or S3 bucket policy changes"
- "Restore affecting production database performance"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, S3 bucket ARNs, IAM role ARNs, backup files (contain full database data)
- HIGH: restore task logs (may contain database names and configuration)
- MEDIUM: task status, S3 integration configuration

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix restore issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest restoring over existing production databases without backup"

## Phase 3 — Rollback
- "Restore from snapshot if native restore causes data issues"
- "If restore task fails, do NOT delete the task — check status and logs first"

## Output Format

```yaml
root_cause: "native_restore_migration — <specific_cause>"
evidence:
  - type: task_status
    content: "<rds_task_status output>"
  - type: version_info
    content: "<source and target SQL Server versions>"
severity: MEDIUM
mitigation:
  immediate: "Fix the specific restore error"
  long_term: "Document migration runbook and validate with test restores"
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
