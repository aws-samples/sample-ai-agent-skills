---
title: "B3 — Supplemental Logging"
description: "Diagnose supplemental logging configuration issues for DMS CDC"
status: active
severity: HIGH
triggers:
  - "supplemental logging"
  - "CDC not capturing changes"
  - "missing changes"
  - "binlog not enabled"
  - "wal_level not logical"
owner: devops-agent
objective: "Ensure supplemental logging is correctly configured on the source for CDC"
context: "CDC requires the source database to log sufficient detail for DMS to reconstruct changes. Without proper supplemental logging, DMS cannot capture inserts, updates, or deletes. Each engine has different logging requirements."
---

## Phase 1 — Triage

MUST:
- Check task migration type includes CDC: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].MigrationType'`
- Check source endpoint engine: `aws dms describe-endpoints --filters Name=endpoint-id,Values=<endpoint-id> --query 'Endpoints[*].EngineName'`
- Check task error messages for logging-related errors: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].LastFailureMessage'`
- Run pre-migration assessment: `aws dms start-replication-task-assessment-run --replication-task-arn <task-arn> --service-access-role-arn <role-arn> --result-location-bucket <bucket> --assessment-run-name logging-check`

SHOULD:
- Check extra connection attributes for logging settings: `aws dms describe-endpoints --filters Name=endpoint-id,Values=<endpoint-id> --query 'Endpoints[*].ExtraConnectionAttributes'`
- Verify the DMS user has privileges to read change logs
- Check if the source database has sufficient log retention

MAY:
- Connect to source database directly to verify logging configuration
- Check if supplemental logging was recently disabled

## Phase 2 — Remediate

MUST:
- Oracle: enable supplemental logging — `ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;` and table-level: `ALTER TABLE schema.table ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;`
- MySQL: set `binlog_format=ROW`, `binlog_row_image=FULL`, ensure `log_bin=ON`
- PostgreSQL: set `wal_level=logical`, `max_replication_slots >= 1`, `max_wal_senders >= 1`
- SQL Server: enable MS-CDC — `EXEC sys.sp_cdc_enable_db;` then per table

SHOULD:
- Set adequate log retention to prevent log loss during migration
- For Oracle, consider using `addSupplementalLogging=Y` in extra connection attributes
- Restart the DMS task after fixing logging configuration

MAY:
- Use DMS pre-migration assessments to validate logging before starting
- Configure log archiving for long-running migrations

## Common Issues

- symptoms: "CDC task starts but no changes captured"
  diagnosis: "Supplemental logging not enabled on source."
  resolution: "Enable engine-specific supplemental logging. Restart the task."

- symptoms: "Oracle: missing supplemental log data"
  diagnosis: "Table-level supplemental logging not enabled for migrated tables."
  resolution: "Add supplemental log data for all columns on each migrated table."

- symptoms: "PostgreSQL: FATAL: number of requested standby connections exceeds max_wal_senders"
  diagnosis: "max_wal_senders too low for the number of replication connections."
  resolution: "Increase max_wal_senders in postgresql.conf. Restart PostgreSQL."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable supplemental logging on source (Oracle) | YELLOW | Source database change — may increase log volume |
| Set binlog_format=ROW on MySQL | RED | Requires MySQL restart — affects all replication |
| Set wal_level=logical on PostgreSQL | RED | Requires PostgreSQL restart — affects all connections |
| Enable MS-CDC on SQL Server | YELLOW | Database-level change — increases log usage |
| Increase log retention on source | GREEN | Configuration change — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Source database changes require DBA approval and maintenance window
- Logging changes affect source database performance

## Data Sensitivity

- **Classification: HIGH**
- Supplemental logging configuration reveals CDC architecture
- Source engine details expose database technology and version
- Extra connection attributes may contain sensitive configuration
- Pre-migration assessment results reveal migration readiness details

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest restarting production databases without DBA approval and maintenance window
- **NEVER** suggest disabling supplemental logging while CDC tasks are running

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Enabled Oracle supplemental logging | Disable supplemental logging if no longer needed (after migration) |
| Changed MySQL binlog_format | Revert binlog_format and restart MySQL (requires maintenance window) |
| Changed PostgreSQL wal_level | Revert wal_level and restart PostgreSQL (requires maintenance window) |
| Enabled SQL Server MS-CDC | Disable MS-CDC on database and tables if no longer needed |

## Output Format

```yaml
root_cause: "supplemental_logging — <engine> — <specific_cause>"
evidence:
  - type: task_type
    content: "<migration type (full-load-and-cdc or cdc-only)>"
  - type: source_engine
    content: "<engine name and version>"
  - type: logging_status
    content: "<current logging configuration>"
severity: HIGH
mitigation:
  immediate: "Enable required supplemental logging on the source"
  long_term: "Include logging verification in migration runbooks"
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
