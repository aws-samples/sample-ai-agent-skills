---
title: "G2 — DMS Issues"
description: "Diagnose AWS Database Migration Service issues with RDS SQL Server"
status: active
severity: MEDIUM
triggers:
  - "DMS"
  - "Database Migration Service"
  - "migration task"
  - "CDC"
  - "change data capture"
  - "replication instance"
  - "MS-CDC"
  - "MS-Replication"
owner: devops-agent
objective: "Resolve DMS migration and replication issues involving RDS SQL Server"
context: "DMS supports SQL Server as both source and target. For SQL Server source CDC, DMS uses MS-Replication (on-premises) or MS-CDC (RDS). CDC requires databases in FULL recovery model. RDS SQL Server as source requires specific configuration. As target, DMS handles schema conversion with SCT."
---

## Phase 1 — Triage

MUST:
- Check DMS task status: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id>`
- Check DMS task statistics: `aws dms describe-table-statistics --replication-task-arn <task-arn>`
- Check replication instance status: `aws dms describe-replication-instances --filters Name=replication-instance-id,Values=<instance-id>`
- Check DMS task logs in CloudWatch: `/aws/dms/tasks/<task-id>`

SHOULD:
- For SQL Server source — verify CDC is enabled:
  ```sql
  -- Check database-level CDC
  SELECT name, is_cdc_enabled FROM sys.databases WHERE name = '<database>';
  -- Enable CDC
  EXEC msdb.dbo.rds_cdc_enable_db '<database>';
  ```
- Check CDC is enabled on source tables:
  ```sql
  USE [<database>];
  SELECT s.name AS schema_name, t.name AS table_name, t.is_tracked_by_cdc
  FROM sys.tables t JOIN sys.schemas s ON t.schema_id = s.schema_id
  WHERE t.is_tracked_by_cdc = 1;
  -- Enable CDC on a table
  EXEC sys.sp_cdc_enable_table
    @source_schema = N'dbo',
    @source_name = N'MyTable',
    @role_name = NULL;
  ```
- Verify recovery model is FULL (required for CDC):
  ```sql
  SELECT name, recovery_model_desc FROM sys.databases WHERE name = '<database>';
  ALTER DATABASE [<database>] SET RECOVERY FULL;
  ```
- Check DMS endpoint connectivity: `aws dms test-connection --replication-instance-arn <ri-arn> --endpoint-arn <endpoint-arn>`

MAY:
- Check for LOB column handling issues
- Verify collation compatibility between source and target
- Check DMS replication instance sizing

## Phase 2 — Remediate

MUST:
- For CDC not enabled:
  ```sql
  EXEC msdb.dbo.rds_cdc_enable_db '<database>';
  ```
- For tables not tracked:
  ```sql
  EXEC sys.sp_cdc_enable_table
    @source_schema = N'dbo',
    @source_name = N'MyTable',
    @role_name = NULL;
  ```
- For endpoint connectivity: verify security groups allow DMS replication instance to reach RDS SQL Server on port 1433

SHOULD:
- For CDC lag: increase replication instance size or optimize source queries
- For LOB issues: use limited LOB mode for better performance
- Set CDC retention period:
  ```sql
  EXEC sys.sp_cdc_change_job @job_type = 'cleanup', @retention = 4320;  -- minutes
  ```

MAY:
- Use SCT (Schema Conversion Tool) for heterogeneous migrations
- Implement DMS validation to verify data integrity
- Use DMS premigration assessment for compatibility checks

## Common Issues

- symptoms: "CDC not capturing changes"
  diagnosis: "CDC not enabled on database or tables. Or recovery model is SIMPLE."
  resolution: "Enable CDC on database and tables. Set recovery model to FULL."

- symptoms: "DMS task failing with transaction log errors"
  diagnosis: "Transaction log backups clearing logs before DMS reads them."
  resolution: "Increase CDC retention. Ensure DMS can keep up with log generation."

- symptoms: "DMS task slow on LOB columns"
  diagnosis: "Full LOB mode reads each LOB individually."
  resolution: "Use limited LOB mode with appropriate max LOB size."

- symptoms: "Endpoint test connection fails"
  diagnosis: "Security group, network routing, or credentials issue."
  resolution: "Verify security group allows DMS replication instance. Check credentials."

## Safety Ratings
- GREEN: describe-replication-tasks, describe-table-statistics, describe-replication-instances, test-connection — read-only inspection
- YELLOW: modify-replication-task, modify-replication-instance — recoverable configuration changes
- RED: delete-replication-task, delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "DMS CDC not capturing changes — data consistency at risk"
- "Fix requires modifying SQL Server source configuration"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials in DMS endpoints, connection strings
- HIGH: DMS task logs (may contain SQL statements and data samples)
- MEDIUM: table migration statistics, replication instance metrics

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix DMS issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest cutting over to target without validating data consistency"

## Phase 3 — Rollback
- "If DMS task fails, do NOT delete the task — stop and investigate first"
- "Restore from snapshot if DMS migration causes data issues"

## Output Format

```yaml
root_cause: "dms — <specific_cause>"
evidence:
  - type: task_status
    content: "<DMS task status and statistics>"
  - type: cdc_config
    content: "<CDC enabled status and table tracking>"
severity: MEDIUM
mitigation:
  immediate: "Fix the specific DMS configuration issue"
  long_term: "Implement DMS monitoring and validation"
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
