---
title: "H2 — SQL Server Audit"
description: "Diagnose SQL Server audit configuration and log access issues on RDS SQL Server"
status: active
severity: MEDIUM
triggers:
  - "audit"
  - "SQL Server audit"
  - "server audit"
  - "database audit"
  - "audit specification"
  - "compliance"
  - "SQLSERVER_AUDIT"
owner: devops-agent
objective: "Resolve SQL Server audit configuration and log access issues on RDS SQL Server"
context: "RDS SQL Server supports SQL Server Audit via option group (SQLSERVER_AUDIT). Audit files are written to S3. Server-level and database-level audit specifications are supported. The audit feature is configured via option group, not directly via T-SQL CREATE SERVER AUDIT ... TO FILE."
---

## Phase 1 — Triage

MUST:
- Check audit option in option group:
  ```
  aws rds describe-option-groups --option-group-name <option-group> \
    --query "OptionGroupsList[0].Options[?OptionName=='SQLSERVER_AUDIT']"
  ```
- Check server audit configuration:
  ```sql
  SELECT name, status_desc, audit_file_path, max_file_size, max_rollover_files
  FROM sys.server_audits;
  ```
- Check server audit specifications:
  ```sql
  SELECT sa.name AS audit_name, sas.name AS spec_name,
         sas.is_state_enabled, sasd.audit_action_name
  FROM sys.server_audit_specifications sas
  JOIN sys.server_audits sa ON sas.audit_guid = sa.audit_guid
  JOIN sys.server_audit_specification_details sasd ON sas.server_specification_id = sasd.server_specification_id;
  ```
- Check database audit specifications:
  ```sql
  SELECT sa.name AS audit_name, das.name AS spec_name,
         das.is_state_enabled, dasd.audit_action_name, dasd.class_desc
  FROM sys.database_audit_specifications das
  JOIN sys.server_audits sa ON das.audit_guid = sa.audit_guid
  JOIN sys.database_audit_specification_details dasd ON das.database_specification_id = dasd.database_specification_id;
  ```

SHOULD:
- Verify S3 bucket configuration for audit files:
  ```
  aws rds describe-option-groups --option-group-name <option-group> \
    --query "OptionGroupsList[0].Options[?OptionName=='SQLSERVER_AUDIT'].OptionSettings"
  ```
- Check for audit-related errors in error log
- Verify IAM role has S3 write permissions for audit bucket

MAY:
- Query audit log data:
  ```sql
  SELECT event_time, action_id, succeeded, session_id,
         server_principal_name, database_name, statement
  FROM sys.fn_get_audit_file('D:\rdsdbdata\SQLAudit\*.sqlaudit', DEFAULT, DEFAULT)
  ORDER BY event_time DESC;
  ```
- Check audit file sizes in S3

## Phase 2 — Remediate

MUST:
- Enable SQL Server Audit via option group:
  ```
  aws rds add-option-to-option-group \
    --option-group-name <option-group> \
    --options '[{
      "OptionName": "SQLSERVER_AUDIT",
      "OptionSettings": [
        {"Name": "S3_BUCKET_ARN", "Value": "arn:aws:s3:::my-audit-bucket"},
        {"Name": "IAM_ROLE_ARN", "Value": "arn:aws:iam::<account>:role/<role>"}
      ]
    }]'
  ```
- Create server audit and specifications:
  ```sql
  USE [master];
  CREATE SERVER AUDIT [RDS_Audit]
    TO FILE (FILEPATH = 'D:\rdsdbdata\SQLAudit\');
  ALTER SERVER AUDIT [RDS_Audit] WITH (STATE = ON);

  CREATE SERVER AUDIT SPECIFICATION [ServerAuditSpec]
    FOR SERVER AUDIT [RDS_Audit]
    ADD (FAILED_LOGIN_GROUP),
    ADD (DATABASE_CHANGE_GROUP),
    ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP)
    WITH (STATE = ON);
  ```

SHOULD:
- Create database-level audit specification:
  ```sql
  USE [MyDatabase];
  CREATE DATABASE AUDIT SPECIFICATION [DatabaseAuditSpec]
    FOR SERVER AUDIT [RDS_Audit]
    ADD (SELECT, INSERT, UPDATE, DELETE ON SCHEMA::dbo BY public)
    WITH (STATE = ON);
  ```
- Monitor audit file delivery to S3
- Set appropriate max_file_size and max_rollover_files

MAY:
- Implement audit log analysis with Athena or CloudWatch Logs
- Set up alerts for specific audit events

## Common Issues

- symptoms: "Audit files not appearing in S3"
  diagnosis: "IAM role missing S3 write permissions or bucket ARN incorrect."
  resolution: "Verify IAM role policy. Check S3 bucket ARN in option group settings."

- symptoms: "Cannot create server audit — permission denied"
  diagnosis: "Master user may need specific permissions for audit creation."
  resolution: "Use the RDS master user. Ensure SQLSERVER_AUDIT option is active."

- symptoms: "Audit consuming excessive storage"
  diagnosis: "Audit specifications too broad or max_rollover_files too high."
  resolution: "Narrow audit specifications. Reduce max_rollover_files. Archive old audit files."

## Safety Ratings
- GREEN: SELECT from sys.server_audits/sys.server_audit_specifications/sys.database_audit_specifications — read-only inspection
- YELLOW: CREATE SERVER AUDIT, ALTER SERVER AUDIT, modify-db-parameter-group — recoverable changes
- RED: DROP SERVER AUDIT (purges audit records), delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Audit configuration change may affect compliance requirements"
- "Fix requires option group changes"
- "Audit log consuming excessive storage"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: audit trail contents (contain user actions, SQL text, and access patterns)
- HIGH: database credentials, audit configuration (reveal security posture)
- MEDIUM: audit specification details, S3 bucket configuration for audit output

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix audit issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest disabling audit on production databases without compliance review"

## Phase 3 — Rollback
- "Revert audit specification changes if they break application behavior"
- "Restore from snapshot if audit configuration change causes issues"

## Output Format

```yaml
root_cause: "audit — <specific_cause>"
evidence:
  - type: audit_config
    content: "<server audit configuration>"
  - type: audit_specs
    content: "<audit specifications>"
severity: MEDIUM
mitigation:
  immediate: "Fix audit configuration"
  long_term: "Implement audit log management and monitoring"
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
