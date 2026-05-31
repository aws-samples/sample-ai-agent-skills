---
title: "G1 — Data Pump Issues"
description: "Diagnose Oracle Data Pump import/export issues on RDS Oracle"
status: active
severity: MEDIUM
triggers:
  - "Data Pump"
  - "expdp"
  - "impdp"
  - "DBMS_DATAPUMP"
  - "ORA-39002"
  - "ORA-39070"
  - "ORA-31626"
  - "DATA_PUMP_DIR"
owner: devops-agent
objective: "Resolve Oracle Data Pump import/export issues on RDS Oracle"
context: "On RDS, Data Pump cannot be run from the OS command line (no expdp/impdp). Use the DBMS_DATAPUMP PL/SQL API or the rdsadmin wrapper procedures. The DATA_PUMP_DIR directory object points to an RDS-managed location. Files are transferred via S3 integration or database links."
---

## Phase 1 — Triage

MUST:
- Check Data Pump job status:
  ```sql
  SELECT owner_name, job_name, operation, job_mode, state, degree
  FROM dba_datapump_jobs WHERE state != 'NOT RUNNING';
  ```
- Check Data Pump log:
  ```sql
  SELECT text FROM TABLE(rdsadmin.rds_file_util.read_text_file('DATA_PUMP_DIR','<log_file>'));
  ```
- Check available space in DATA_PUMP_DIR:
  ```sql
  SELECT * FROM TABLE(rdsadmin.rds_file_util.listdir('DATA_PUMP_DIR'));
  ```
- Check RDS Oracle version supports S3 integration (if using S3):
  ```
  aws rds describe-db-instances --db-instance-identifier <instance-id> \
    --query 'DBInstances[0].{Version:EngineVersion,OptionGroups:OptionGroupMemberships}'
  ```

SHOULD:
- Check for stuck Data Pump jobs:
  ```sql
  SELECT owner_name, job_name, state FROM dba_datapump_jobs;
  -- Kill stuck job
  DECLARE
    h NUMBER;
  BEGIN
    h := DBMS_DATAPUMP.ATTACH('<job_name>','<owner>');
    DBMS_DATAPUMP.STOP_JOB(h, immediate=>1);
  END;
  /
  ```
- Verify S3 integration option group (if using S3 for file transfer):
  ```
  aws rds describe-option-groups --option-group-name <option-group> \
    --query "OptionGroupsList[0].Options[?OptionName=='S3_INTEGRATION']"
  ```
- Check tablespace space for import target

MAY:
- Check for object-level errors in the Data Pump log
- Verify database link connectivity (if using DB link for transfer)

## Phase 2 — Remediate

MUST:
- For S3-based Data Pump workflow:
  ```sql
  -- Download dump file from S3
  SELECT rdsadmin.rdsadmin_s3_tasks.download_from_s3(
    p_bucket_name => '<bucket>',
    p_s3_prefix => '<prefix>/export.dmp',
    p_directory_name => 'DATA_PUMP_DIR')
  AS TASK_ID FROM DUAL;

  -- Check task status
  SELECT text FROM TABLE(rdsadmin.rds_file_util.read_text_file('BDUMP','dbtask-<task_id>.log'));

  -- Run import
  DECLARE
    h NUMBER;
  BEGIN
    h := DBMS_DATAPUMP.OPEN('IMPORT','SCHEMA',NULL,'IMPORT_JOB');
    DBMS_DATAPUMP.ADD_FILE(h,'export.dmp','DATA_PUMP_DIR');
    DBMS_DATAPUMP.ADD_FILE(h,'import.log','DATA_PUMP_DIR',NULL,DBMS_DATAPUMP.KU$_FILE_TYPE_LOG_FILE);
    DBMS_DATAPUMP.METADATA_FILTER(h,'SCHEMA_EXPR','IN (''MYSCHEMA'')');
    DBMS_DATAPUMP.START_JOB(h);
  END;
  /
  ```
- Clean up old dump files: `EXEC UTL_FILE.FREMOVE('DATA_PUMP_DIR','<old_file>');`

SHOULD:
- Upload dump files to S3 after export:
  ```sql
  SELECT rdsadmin.rdsadmin_s3_tasks.upload_to_s3(
    p_bucket_name => '<bucket>',
    p_prefix => '<prefix>/',
    p_s3_prefix => '',
    p_directory_name => 'DATA_PUMP_DIR')
  AS TASK_ID FROM DUAL;
  ```
- Monitor Data Pump job progress via DBA_DATAPUMP_JOBS

MAY:
- Use database links for direct Data Pump network mode import
- Implement parallel Data Pump for large datasets

## Common Issues

- symptoms: "ORA-39002: invalid operation / ORA-39070: Unable to open the log file"
  diagnosis: "DATA_PUMP_DIR not accessible or full."
  resolution: "Check directory permissions. Clean up old files. Verify DATA_PUMP_DIR exists."

- symptoms: "Data Pump job stuck in 'NOT RUNNING' but still listed"
  diagnosis: "Orphaned Data Pump master table."
  resolution: "Drop the master table: DROP TABLE <owner>.<job_name>;"

- symptoms: "S3 integration download fails"
  diagnosis: "IAM role for S3 access not configured or bucket policy incorrect."
  resolution: "Verify S3_INTEGRATION option group and associated IAM role."

## Safety Ratings
- GREEN: SELECT from dba_datapump_jobs, rds_file_util.read_text_file, rds_file_util.listdir, describe-option-groups — read-only inspection
- YELLOW: DBMS_DATAPUMP operations, rdsadmin_s3_tasks (download/upload), UTL_FILE.FREMOVE — recoverable data operations
- RED: delete-db-instance, DROP TABLE (master table cleanup) — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Data Pump import affecting production database performance"
- "S3 integration IAM role issues"
- "Data Pump job stuck and cannot be stopped"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, S3 bucket ARNs, IAM role ARNs, Data Pump dump files (contain full schema data)
- HIGH: Data Pump log files (may contain schema structure and object details)
- MEDIUM: job status, S3 integration configuration

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix Data Pump issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest running Data Pump import on production without testing on a non-production instance first"

## Phase 3 — Rollback
- "Restore from snapshot if Data Pump import causes data issues"
- "If Data Pump job is stuck, use DBMS_DATAPUMP.STOP_JOB to terminate"
- "Clean up dump files from DATA_PUMP_DIR after successful operations"

## Output Format

```yaml
root_cause: "data_pump — <specific_cause>"
evidence:
  - type: job_status
    content: "<dba_datapump_jobs output>"
  - type: log_content
    content: "<Data Pump log entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix the specific Data Pump error"
  long_term: "Implement automated Data Pump workflows with S3 integration"
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
