---
title: "G2 — Snapshot Restore"
description: "Diagnose Amazon Keyspaces backup and restore operations"
status: active
severity: MEDIUM
triggers:
  - "snapshot"
  - "restore failed"
  - "backup restore"
  - "data recovery"
  - "table restore"
owner: devops-agent
objective: "Resolve backup and restore issues in Amazon Keyspaces"
context: "Keyspaces does not support Cassandra-style snapshots (nodetool snapshot). Backup is provided through PITR (continuous) and data export (CQL COPY, AWS Glue). Restores from PITR create a new table. For data export/import, use CQL COPY TO/FROM or AWS Glue ETL jobs. Cross-region backup requires export and re-import."
---

## Phase 1 — Triage

MUST:
- Determine the backup/restore method needed (PITR vs data export)
- Check PITR status if using PITR: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table> --query 'pointInTimeRecovery'`
- Verify IAM permissions for the restore operation
- Check if a restore is already in progress

SHOULD:
- Estimate data size for export/import operations
- Verify target keyspace and table name availability
- Check available storage and compute for Glue-based export

MAY:
- Compare PITR restore vs data export for the specific use case
- Check cross-region requirements

## Phase 2 — Remediate

MUST:
- For PITR restore: `aws keyspaces restore-table --source-keyspace-name <keyspace> --source-table-name <table> --target-keyspace-name <keyspace> --target-table-name <table-restored> --restore-timestamp <timestamp>`
- For data export via CQL: `cqlsh -e "COPY keyspace.table TO '/path/export.csv'"`
- For data import via CQL: `cqlsh -e "COPY keyspace.table FROM '/path/export.csv'"`
- Verify restored data integrity

SHOULD:
- Use AWS Glue for large-scale data export/import
- Test restore procedures before they are needed in production
- Document the restore process and expected timelines

MAY:
- Implement automated backup to S3 using AWS Glue scheduled jobs
- Set up cross-region data replication for disaster recovery

## Common Issues

- symptoms: "Trying to use nodetool snapshot"
  diagnosis: "Keyspaces does not support Cassandra nodetool commands."
  resolution: "Use PITR for backup/restore. Use CQL COPY or AWS Glue for data export."

- symptoms: "PITR restore taking too long"
  diagnosis: "Restore time depends on table size. Large tables take longer."
  resolution: "Wait for restore to complete. Monitor table status. No way to speed up PITR restore."

- symptoms: "Need to restore to a different region"
  diagnosis: "PITR restores to the same region only."
  resolution: "Export data using CQL COPY or AWS Glue, then import to the target region."

## Output Format

```yaml
root_cause: "backup_restore — <specific_cause>"
evidence:
  - type: backup_method
    content: "<PITR or data export method>"
  - type: restore_status
    content: "<restore progress and status>"
  - type: data_integrity
    content: "<verification of restored data>"
severity: MEDIUM
mitigation:
  immediate: "Complete restore using appropriate method"
  long_term: "Implement regular backup testing and cross-region strategy"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never drop keyspaces or tables containing production data
- Never modify capacity mode on tables during peak traffic without load testing
- Never alter TTL settings without understanding data retention requirements
- Never change encryption settings on tables with active workloads
- Never disable point-in-time recovery on production tables

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore tables from point-in-time recovery if data is accidentally modified
- Revert capacity mode changes if throttling increases after modification
- Re-enable previous auto-scaling configurations if performance degrades

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
