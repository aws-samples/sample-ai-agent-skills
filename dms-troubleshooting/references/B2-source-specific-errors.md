---
title: "B2 — Source-Specific Errors"
description: "Diagnose source engine-specific errors in DMS migrations"
status: active
severity: HIGH
triggers:
  - "Oracle source error"
  - "MySQL binlog error"
  - "PostgreSQL replication slot"
  - "SQL Server CDC error"
  - "source engine error"
owner: devops-agent
objective: "Resolve source database engine-specific errors that prevent DMS from reading data"
context: "Each source engine has unique requirements and failure modes. Oracle requires ARCHIVELOG mode and supplemental logging. MySQL requires binary logging with ROW format. PostgreSQL requires logical replication. SQL Server requires MS-CDC. Engine-specific errors often appear in task logs."
---

## Phase 1 — Triage

MUST:
- Check task last failure message: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].LastFailureMessage'`
- Check endpoint engine and version: `aws dms describe-endpoints --filters Name=endpoint-id,Values=<endpoint-id> --query 'Endpoints[*].{Engine:EngineName,Version:EngineDisplayName}'`
- Check endpoint extra connection attributes: `aws dms describe-endpoints --filters Name=endpoint-id,Values=<endpoint-id> --query 'Endpoints[*].ExtraConnectionAttributes'`
- Verify DMS supports the source engine version: `aws dms describe-orderable-replication-instances --query 'OrderableReplicationInstances[0].EngineVersion'`

SHOULD:
- Review DMS task CloudWatch Logs for engine-specific error details
- Check if the source database user has required privileges
- Verify source database parameter settings match DMS requirements

MAY:
- Check AWS DMS release notes for known issues with the source engine version
- Test with a simpler task (single table) to isolate the issue

## Phase 2 — Remediate

MUST:
- For Oracle: ensure ARCHIVELOG mode, supplemental logging, and LogMiner or Binary Reader access
- For MySQL: set binlog_format=ROW, binlog_row_image=FULL, enable binary logging
- For PostgreSQL: set wal_level=logical, create logical replication slot
- For SQL Server: enable MS-CDC on database and required tables

SHOULD:
- Grant the DMS user all required engine-specific privileges
- Set appropriate extra connection attributes for the engine
- Test CDC separately after fixing source configuration

MAY:
- Consider using Binary Reader instead of LogMiner for Oracle (better performance)
- Adjust wal_sender_timeout for PostgreSQL to prevent slot disconnection

## Common Issues

- symptoms: "Oracle: ORA-01291 missing logfile"
  diagnosis: "Archive logs have been deleted before DMS could read them."
  resolution: "Increase archive log retention. Use Binary Reader for faster log processing."

- symptoms: "MySQL: binlog_format is not ROW"
  diagnosis: "MySQL binary logging not configured for row-based replication."
  resolution: "Set binlog_format=ROW in MySQL configuration. Restart MySQL if needed."

- symptoms: "PostgreSQL: replication slot does not exist"
  diagnosis: "Logical replication slot was not created or was dropped."
  resolution: "Create slot: SELECT pg_create_logical_replication_slot('dms_slot', 'test_decoding');"

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix source engine configuration (ARCHIVELOG, binlog, wal_level) | RED | Source database change — may require restart |
| Grant DMS user engine-specific privileges | YELLOW | Permission change on source database |
| Set extra connection attributes | GREEN | Endpoint configuration — non-destructive |
| Use Binary Reader instead of LogMiner (Oracle) | YELLOW | Changes log reading method — test thoroughly |
| Test with single table to isolate issue | GREEN | Testing — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Source database configuration changes require DBA approval
- Engine-specific fixes may require source database restart

## Data Sensitivity

- **Classification: HIGH**
- Source engine details reveal database technology, version, and configuration
- DMS user privileges expose database access level
- Extra connection attributes may contain sensitive tuning parameters
- Task logs may contain source data samples in error messages

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest restarting production source databases without DBA approval
- **NEVER** suggest granting DMS user superuser/DBA privileges on source

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Changed source database configuration | Revert configuration and restart if needed (maintenance window) |
| Granted DMS user additional privileges | Revoke added privileges after migration completes |
| Changed extra connection attributes | Revert endpoint extra connection attributes |
| Switched to Binary Reader (Oracle) | Revert to LogMiner if Binary Reader causes issues |

## Output Format

```yaml
root_cause: "source_engine_error — <engine> — <specific_cause>"
evidence:
  - type: task_error
    content: "<last failure message>"
  - type: endpoint_config
    content: "<engine and extra connection attributes>"
severity: HIGH
mitigation:
  immediate: "Fix the engine-specific configuration"
  long_term: "Document source prerequisites per engine type"
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
  - "NEVER suggest disabling SSL for replication endpoints"
  - "NEVER suggest public replication instances"
  - "NEVER suggest deleting replication tasks without data verification"
