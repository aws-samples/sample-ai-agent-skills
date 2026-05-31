---
title: "D1 — Parameter Group Issues"
description: "Diagnose RDS MySQL parameter group misconfiguration and apply issues"
status: active
severity: MEDIUM
triggers:
  - "parameter group"
  - "pending-reboot"
  - "parameter not taking effect"
  - "static parameter"
---

## Phase 1 — Triage

MUST:
- Check current parameter group: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].DBParameterGroups'`
- Check pending-reboot status: look for `ParameterApplyStatus: pending-reboot`
- List modified parameters: `aws rds describe-db-parameters --db-parameter-group-name <group> --source user`
- Compare with defaults: `aws rds describe-db-parameters --db-parameter-group-name default.mysql8.0 --query 'Parameters[?ParameterName==\`innodb_buffer_pool_size\`]'`

SHOULD:
- Check if parameter is static or dynamic: `aws rds describe-db-parameters --db-parameter-group-name <group> --query 'Parameters[?ParameterName==\`<param>\`].ApplyType'`
- Verify parameter group family matches engine version
- Check for conflicting parameters (e.g., `innodb_file_per_table` with `innodb_file_format`)

## Phase 2 — Remediate

- Static parameters require reboot: `aws rds reboot-db-instance --db-instance-identifier <id>`
- Dynamic parameters apply immediately with `--apply-method immediate`
- Wrong family: create new parameter group with correct family, associate, and reboot
- Key MySQL parameters to verify: `innodb_buffer_pool_size`, `max_connections`, `slow_query_log`, `long_query_time`, `binlog_format`, `character_set_server`, `innodb_flush_log_at_trx_commit`

## Safety Ratings
- GREEN: describe-db-instances, describe-db-parameters, describe-db-parameter-groups — read-only inspection
- YELLOW: modify-db-parameter-group, reboot-db-instance — recoverable but may require reboot
- RED: reset-db-parameter-group, delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Parameter group family mismatch after engine upgrade"
- "Fix involves modifying encryption settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, parameter group values (may contain security-sensitive settings)
- MEDIUM: parameter group configuration, apply status, engine version details

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix parameter issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "Document previous parameter values before making changes to enable rollback"

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
