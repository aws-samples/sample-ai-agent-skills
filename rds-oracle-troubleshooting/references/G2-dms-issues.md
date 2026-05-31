---
title: "G2 — DMS Issues"
description: "Diagnose AWS Database Migration Service issues with RDS Oracle"
status: active
severity: MEDIUM
triggers:
  - "DMS"
  - "Database Migration Service"
  - "migration task"
  - "CDC"
  - "change data capture"
  - "replication instance"
  - "LogMiner"
  - "Binary Reader"
owner: devops-agent
objective: "Resolve DMS migration and replication issues involving RDS Oracle"
context: "DMS supports Oracle as both source and target. For Oracle source CDC, DMS uses LogMiner (default) or Binary Reader. Oracle supplemental logging must be enabled. RDS Oracle as source requires specific parameter settings. As target, DMS handles schema conversion with SCT."
---

## Phase 1 — Triage

MUST:
- Check DMS task status: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id>`
- Check DMS task statistics: `aws dms describe-table-statistics --replication-task-arn <task-arn>`
- Check replication instance status: `aws dms describe-replication-instances --filters Name=replication-instance-id,Values=<instance-id>`
- Check DMS task logs in CloudWatch: `/aws/dms/tasks/<task-id>`

SHOULD:
- For Oracle source — verify supplemental logging:
  ```sql
  SELECT supplemental_log_data_min, supplemental_log_data_pk, supplemental_log_data_all
  FROM v$database;
  -- Enable if not set
  EXEC rdsadmin.rdsadmin_util.alter_supplemental_logging('ADD');
  EXEC rdsadmin.rdsadmin_util.alter_supplemental_logging('ADD','PRIMARY KEY');
  ```
- Check Oracle archive log mode (required for CDC):
  ```sql
  SELECT log_mode FROM v$database;  -- Should be ARCHIVELOG
  ```
- Verify archive log retention:
  ```sql
  EXEC rdsadmin.rdsadmin_util.set_configuration('archivelog retention hours', 24);
  ```
- Check DMS endpoint connectivity: `aws dms test-connection --replication-instance-arn <ri-arn> --endpoint-arn <endpoint-arn>`

MAY:
- Check for LOB column handling issues (LOB migration is slower)
- Verify character set compatibility between source and target
- Check DMS replication instance sizing

## Phase 2 — Remediate

MUST:
- For supplemental logging not enabled:
  ```sql
  EXEC rdsadmin.rdsadmin_util.alter_supplemental_logging('ADD');
  EXEC rdsadmin.rdsadmin_util.alter_supplemental_logging('ADD','PRIMARY KEY');
  ```
- For archive log retention too short:
  ```sql
  EXEC rdsadmin.rdsadmin_util.set_configuration('archivelog retention hours', 24);
  ```
- For endpoint connectivity: verify security groups allow DMS replication instance to reach RDS Oracle on port 1521

SHOULD:
- For CDC lag: increase replication instance size or optimize source queries
- For LOB issues: use limited LOB mode for better performance
- For character set mismatch: configure DMS task settings for character set mapping

MAY:
- Use Binary Reader instead of LogMiner for better CDC performance (requires additional setup)
- Implement DMS validation to verify data integrity
- Use SCT (Schema Conversion Tool) for heterogeneous migrations

## Common Issues

- symptoms: "CDC not capturing changes"
  diagnosis: "Supplemental logging not enabled or archive logs purged too quickly."
  resolution: "Enable supplemental logging. Set archive log retention to at least 24 hours."

- symptoms: "DMS task failing with 'ORA-01291: missing logfile'"
  diagnosis: "Archive logs deleted before DMS could read them."
  resolution: "Increase archivelog retention hours. Restart task with full load + CDC."

- symptoms: "DMS task slow on LOB columns"
  diagnosis: "Full LOB mode reads each LOB individually. Very slow for large LOBs."
  resolution: "Use limited LOB mode with appropriate max LOB size. Or exclude LOB columns and migrate separately."

- symptoms: "Endpoint test connection fails"
  diagnosis: "Security group, network routing, or credentials issue."
  resolution: "Verify security group allows DMS replication instance IP. Check credentials."

## Safety Ratings
- GREEN: describe-replication-tasks, describe-table-statistics, describe-replication-instances, test-connection, SELECT from v$database — read-only inspection
- YELLOW: EXEC rdsadmin.rdsadmin_util.alter_supplemental_logging, EXEC rdsadmin.rdsadmin_util.set_configuration — recoverable changes
- RED: delete-replication-task, delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "DMS CDC not capturing changes — data consistency at risk"
- "Archive logs being purged before DMS can read them"
- "Fix requires enabling supplemental logging (may increase I/O)"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials in DMS endpoints, connection strings, supplemental logging configuration
- HIGH: DMS task logs (may contain SQL statements and data samples)
- MEDIUM: table migration statistics, replication instance metrics

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix DMS issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest cutting over to target without validating data consistency"

## Phase 3 — Rollback
- "If DMS task fails, do NOT delete the task — stop and investigate first"
- "Revert supplemental logging changes if they cause excessive I/O"
- "Restore from snapshot if DMS migration causes data issues"

## Output Format

```yaml
root_cause: "dms — <specific_cause>"
evidence:
  - type: task_status
    content: "<DMS task status and statistics>"
  - type: oracle_config
    content: "<supplemental logging and archive log settings>"
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
