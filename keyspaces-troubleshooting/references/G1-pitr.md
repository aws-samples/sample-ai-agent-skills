---
title: "G1 — Point-in-Time Recovery (PITR)"
description: "Diagnose Amazon Keyspaces PITR configuration and restore issues"
status: active
severity: HIGH
triggers:
  - "PITR"
  - "point-in-time recovery"
  - "backup"
  - "restore table"
  - "continuous backup"
owner: devops-agent
objective: "Resolve PITR configuration and restore issues"
context: "Keyspaces PITR provides continuous backups with 35-day retention. PITR must be enabled per table. Restores create a new table — in-place restore is not supported. Restore time depends on table size. PITR is not enabled by default. There are no manual snapshot commands."
---

## Phase 1 — Triage

MUST:
- Check PITR status: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table> --query 'pointInTimeRecovery'`
- Verify the restore timestamp is within the 35-day retention window
- Check if a restore is already in progress: `aws keyspaces list-tables --keyspace-name <keyspace>`
- Verify IAM permissions for restore: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names cassandra:Restore`

SHOULD:
- Check the earliest restorable timestamp from PITR status
- Verify the target table name does not already exist
- Check CloudTrail for PITR-related events

MAY:
- Estimate restore time based on table size
- Verify encryption settings for the restored table

## Phase 2 — Remediate

MUST:
- Enable PITR if not already enabled: `aws keyspaces update-table --keyspace-name <keyspace> --table-name <table> --point-in-time-recovery status=ENABLED`
- Restore to a new table: `aws keyspaces restore-table --source-keyspace-name <keyspace> --source-table-name <table> --target-keyspace-name <keyspace> --target-table-name <table-restored> --restore-timestamp <timestamp>`
- Wait for restore to complete and verify data

SHOULD:
- Enable PITR on all production tables
- Test restore procedures periodically
- Document the restore process for the operations team

MAY:
- Set up CloudWatch alarms for PITR status changes
- Implement cross-region backup by restoring to another region

## Common Issues

- symptoms: "PITR not enabled — cannot restore"
  diagnosis: "PITR must be explicitly enabled on each table."
  resolution: "Enable PITR: aws keyspaces update-table --point-in-time-recovery status=ENABLED"

- symptoms: "Restore timestamp outside retention window"
  diagnosis: "PITR retention is 35 days. Requested timestamp is older."
  resolution: "Use a timestamp within the 35-day window. Check earliestRestorableTimestamp."

- symptoms: "Restore table already exists"
  diagnosis: "Target table name for restore already exists in the keyspace."
  resolution: "Use a different target table name. Delete the existing table if it's from a previous restore."

## Output Format

```yaml
root_cause: "pitr — <specific_cause>"
evidence:
  - type: pitr_status
    content: "<PITR enabled/disabled and retention window>"
  - type: restore_request
    content: "<restore timestamp and target table>"
  - type: iam_permissions
    content: "<restore permissions check>"
severity: HIGH
mitigation:
  immediate: "Enable PITR and perform restore to new table"
  long_term: "Enable PITR on all production tables and test restores regularly"
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
