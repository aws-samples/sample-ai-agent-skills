---
title: "E1 — Native Backup to S3"
description: "Diagnose native backup and restore to/from S3 issues on RDS SQL Server"
status: active
severity: HIGH
triggers:
  - "native backup"
  - "backup to S3"
  - "rds_backup_database"
  - "rds_restore_database"
  - "rds_task_status"
  - "SQLSERVER_BACKUP_RESTORE"
  - "backup failed"
owner: devops-agent
objective: "Resolve native backup/restore to S3 issues on RDS SQL Server"
context: "RDS SQL Server native backup/restore uses msdb stored procedures (rds_backup_database, rds_restore_database) to interact with S3. Requires the SQLSERVER_BACKUP_RESTORE option in an option group and an IAM role with S3 access. Direct BACKUP DATABASE TO DISK is not supported."
---

## Phase 1 — Triage

MUST:
- Check backup task status:
  ```sql
  EXEC msdb.dbo.rds_task_status @db_name='<database>';
  ```
- Check option group for SQLSERVER_BACKUP_RESTORE:
  ```
  aws rds describe-option-groups --option-group-name <option-group> \
    --query "OptionGroupsList[0].Options[?OptionName=='SQLSERVER_BACKUP_RESTORE']"
  ```
- Verify option group is associated with instance: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].OptionGroupMemberships'`
- Check IAM role for S3 access:
  ```
  aws rds describe-option-groups --option-group-name <option-group> \
    --query "OptionGroupsList[0].Options[?OptionName=='SQLSERVER_BACKUP_RESTORE'].OptionSettings"
  ```

SHOULD:
- Verify S3 bucket exists and IAM role has access:
  ```
  aws s3 ls s3://<bucket>/<prefix>/
  aws iam get-role --role-name <role-name>
  ```
- Check for failed tasks with error details:
  ```sql
  EXEC msdb.dbo.rds_task_status;
  -- Look for task_type = 'BACKUP_DB' or 'RESTORE_DB' with status = 'ERROR'
  ```
- Check RDS events: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440`

MAY:
- Check S3 bucket policy for cross-account access
- Verify KMS key permissions if using encrypted backups

## Phase 2 — Remediate

MUST:
- Configure SQLSERVER_BACKUP_RESTORE option:
  ```
  aws rds add-option-to-option-group \
    --option-group-name <option-group> \
    --options '[{
      "OptionName": "SQLSERVER_BACKUP_RESTORE",
      "OptionSettings": [{"Name": "IAM_ROLE_ARN", "Value": "arn:aws:iam::<account>:role/<role>"}]
    }]'
  ```
- Run native backup:
  ```sql
  EXEC msdb.dbo.rds_backup_database
    @source_db_name='MyDatabase',
    @s3_arn_to_backup_to='arn:aws:s3:::mybucket/backups/MyDatabase.bak',
    @overwrite_s3_backup_file=1;
  ```
- Run native restore:
  ```sql
  EXEC msdb.dbo.rds_restore_database
    @restore_db_name='MyDatabase',
    @s3_arn_to_restore_from='arn:aws:s3:::mybucket/backups/MyDatabase.bak';
  ```
- Monitor task progress:
  ```sql
  EXEC msdb.dbo.rds_task_status @db_name='MyDatabase';
  ```

SHOULD:
- For compressed backups (faster, smaller):
  ```sql
  EXEC msdb.dbo.rds_backup_database
    @source_db_name='MyDatabase',
    @s3_arn_to_backup_to='arn:aws:s3:::mybucket/backups/MyDatabase.bak',
    @overwrite_s3_backup_file=1,
    @type='FULL',
    @number_of_files=4;  -- Parallel backup files for faster backup
  ```
- Cancel a stuck task:
  ```sql
  EXEC msdb.dbo.rds_cancel_task @task_id=<task_id>;
  ```
- Enable backup compression default in parameter group for faster backups

MAY:
- Use KMS encryption for backup files:
  ```sql
  EXEC msdb.dbo.rds_backup_database
    @source_db_name='MyDatabase',
    @s3_arn_to_backup_to='arn:aws:s3:::mybucket/backups/MyDatabase.bak',
    @kms_master_key_arn='arn:aws:kms:<region>:<account>:key/<key-id>';
  ```
- Implement automated backup schedules via SQL Agent jobs

## Common Issues

- symptoms: "Task fails with 'Access Denied' to S3"
  diagnosis: "IAM role does not have s3:PutObject/s3:GetObject permissions on the bucket."
  resolution: "Update IAM role policy. Verify S3 bucket policy allows the role."

- symptoms: "SQLSERVER_BACKUP_RESTORE option not found"
  diagnosis: "Option not added to the option group or option group not associated."
  resolution: "Add SQLSERVER_BACKUP_RESTORE option with IAM role ARN to the option group."

- symptoms: "Restore fails with 'database already exists'"
  diagnosis: "Target database name already exists on the instance."
  resolution: "Drop the existing database first or use a different restore_db_name."

## Safety Ratings
- GREEN: SELECT from msdb.dbo.rds_task_status, describe-db-instances — read-only inspection
- YELLOW: msdb.dbo.rds_backup_database, msdb.dbo.rds_restore_database — recoverable backup/restore operations
- RED: delete-db-instance, dropping databases — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Native backup to S3 failing — RPO at risk"
- "Fix requires IAM role or S3 bucket policy changes"
- "Backup consuming excessive storage or I/O"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, S3 bucket ARNs, IAM role ARNs, backup files (contain full database data)
- HIGH: backup task logs (may contain database names and configuration)
- MEDIUM: backup status, S3 integration configuration

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix backup issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest deleting S3 backup files without confirming retention requirements"

## Phase 3 — Rollback
- "Restore from snapshot if backup configuration change causes issues"
- "If native backup fails, use RDS automated snapshots as fallback"

## Output Format

```yaml
root_cause: "native_backup — <specific_cause>"
evidence:
  - type: task_status
    content: "<rds_task_status output>"
  - type: option_group
    content: "<SQLSERVER_BACKUP_RESTORE configuration>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role or option group configuration"
  long_term: "Implement automated native backup schedules with monitoring"
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
