---
title: "H3 — Oracle Audit"
description: "Diagnose Oracle audit configuration and log access issues on RDS Oracle"
status: active
severity: MEDIUM
triggers:
  - "audit"
  - "unified audit"
  - "audit trail"
  - "ORA-01031"
  - "audit log"
  - "compliance"
  - "AUDIT_TRAIL"
owner: devops-agent
objective: "Resolve Oracle audit configuration and log access issues on RDS Oracle"
context: "RDS Oracle supports standard audit (AUDIT_TRAIL parameter) and unified audit (Oracle 12c+). Audit logs are stored in the database (SYS.AUD$ or UNIFIED_AUDIT_TRAIL). Audit files are accessible via the RDS logs API. OS-level audit files are not directly accessible."
---

## Phase 1 — Triage

MUST:
- Check audit configuration:
  ```sql
  SELECT name, value FROM v$parameter WHERE name LIKE 'audit%';
  ```
- Check if unified audit is enabled:
  ```sql
  SELECT value FROM v$option WHERE parameter = 'Unified Auditing';
  ```
- Check audit policies (unified audit):
  ```sql
  SELECT policy_name, enabled_option, entity_name, entity_type
  FROM audit_unified_enabled_policies;
  ```
- Check standard audit settings:
  ```sql
  SELECT user_name, audit_option, success, failure FROM dba_stmt_audit_opts;
  SELECT user_name, object_name, audit_option FROM dba_obj_audit_opts;
  ```

SHOULD:
- Check audit trail size:
  ```sql
  -- Standard audit
  SELECT COUNT(*) FROM sys.aud$;
  -- Unified audit
  SELECT COUNT(*) FROM unified_audit_trail;
  ```
- Check audit log files via RDS API: `aws rds describe-db-log-files --db-instance-identifier <instance-id> --filename-contains audit`
- Download audit logs: `aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name <audit-log-file>`

MAY:
- Check for audit trail purge schedule
- Verify audit configuration meets compliance requirements

## Phase 2 — Remediate

MUST:
- Enable standard audit via parameter group:
  ```
  aws rds modify-db-parameter-group \
    --db-parameter-group-name <param-group> \
    --parameters "ParameterName=audit_trail,ParameterValue=DB,ApplyMethod=pending-reboot"
  ```
- Create unified audit policies:
  ```sql
  CREATE AUDIT POLICY critical_actions
    ACTIONS DELETE ON <owner>.<table>,
            ALTER TABLE,
            DROP TABLE;
  AUDIT POLICY critical_actions;
  ```

SHOULD:
- Purge old audit records to manage space:
  ```sql
  -- Standard audit
  EXEC rdsadmin.rdsadmin_util.clean_aud_table(90);  -- Keep 90 days
  -- Unified audit
  BEGIN
    DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL(
      audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
      use_last_arch_timestamp => FALSE);
  END;
  /
  ```
- Set up audit trail archival to S3 before purging
- Monitor audit trail table size

MAY:
- Implement fine-grained auditing (FGA) for column-level audit:
  ```sql
  BEGIN
    DBMS_FGA.ADD_POLICY(
      object_schema => '<owner>',
      object_name => '<table>',
      policy_name => 'sensitive_data_access',
      audit_column => '<column>',
      enable => TRUE);
  END;
  /
  ```
- Stream audit logs to CloudWatch Logs for centralized monitoring

## Common Issues

- symptoms: "Audit trail consuming excessive tablespace"
  diagnosis: "SYS.AUD$ or unified audit trail not being purged."
  resolution: "Use rdsadmin.rdsadmin_util.clean_aud_table or DBMS_AUDIT_MGMT to purge old records."

- symptoms: "Cannot access OS audit files"
  diagnosis: "No OS access on RDS. OS audit files not directly accessible."
  resolution: "Use DB audit trail (AUDIT_TRAIL=DB) or unified audit. Access logs via RDS logs API."

- symptoms: "Audit not capturing expected events"
  diagnosis: "Audit policy not enabled or not covering the specific action."
  resolution: "Review and update audit policies. Ensure policies are enabled."

## Safety Ratings
- GREEN: SELECT from v$parameter/v$option/audit_unified_enabled_policies/dba_stmt_audit_opts/unified_audit_trail, describe-db-log-files — read-only inspection
- YELLOW: modify-db-parameter-group (audit_trail), CREATE AUDIT POLICY, rdsadmin.rdsadmin_util.clean_aud_table — recoverable changes
- RED: delete-db-instance, DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL (purges audit records) — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Audit trail consuming excessive tablespace"
- "Fix requires parameter group change (audit_trail) that needs reboot"
- "Audit configuration change may affect compliance requirements"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: audit trail contents (contain user actions, SQL text, and access patterns)
- HIGH: database credentials, audit policy configuration (reveal security posture)
- MEDIUM: audit trail size, parameter settings

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix audit issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest purging audit records without confirming compliance retention requirements"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest disabling audit on production databases without compliance review"

## Phase 3 — Rollback
- "Revert parameter group changes (audit_trail) and reboot if needed"
- "If audit policy changes break application behavior, disable the specific policy"
- "Restore from snapshot if audit configuration change causes issues"

## Output Format

```yaml
root_cause: "audit — <specific_cause>"
evidence:
  - type: audit_config
    content: "<audit parameter settings>"
  - type: audit_policies
    content: "<enabled audit policies>"
severity: MEDIUM
mitigation:
  immediate: "Fix audit configuration"
  long_term: "Implement audit trail management and monitoring"
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
