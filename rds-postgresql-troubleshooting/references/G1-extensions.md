---
title: "G1 — Extension Management"
description: "Diagnose extension installation, compatibility, and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "extension"
  - "CREATE EXTENSION"
  - "pg_stat_statements"
  - "PostGIS"
  - "pg_cron"
---

## Phase 1 — Triage

MUST:
- Check installed extensions: `SELECT * FROM pg_extension;`
- Check available extensions: `SELECT * FROM pg_available_extensions WHERE installed_version IS NULL ORDER BY name;`
- Check RDS-allowed extensions: `SHOW rds.allowed_extensions;`
- Check if extension requires shared_preload_libraries: `aws rds describe-db-parameters --db-parameter-group-name <group> --query 'Parameters[?ParameterName==\`shared_preload_libraries\`]'`

SHOULD:
- Verify extension compatibility with PostgreSQL version
- Check if extension requires specific parameter group settings
- For PostGIS: verify `CREATE EXTENSION postgis;` — may need `rds_superuser` role

## Phase 2 — Remediate

- Install extension: `CREATE EXTENSION IF NOT EXISTS <ext_name>;`
- Extensions requiring shared_preload_libraries (pg_stat_statements, pg_cron, pgaudit):
  1. Add to `shared_preload_libraries` in parameter group
  2. Reboot instance
  3. `CREATE EXTENSION <ext_name>;`
- Update extension: `ALTER EXTENSION <ext_name> UPDATE TO '<version>';`
- Extension not available: check if it's on the RDS allowlist for your engine version
- pg_cron: requires `cron.database_name` parameter set to the target database

## Safety Ratings
- GREEN: SELECT from pg_available_extensions/pg_extension — read-only inspection
- YELLOW: CREATE EXTENSION, ALTER EXTENSION UPDATE, modify-db-parameter-group (shared_preload_libraries) — recoverable but may require reboot
- RED: DROP EXTENSION, delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires shared_preload_libraries change (needs reboot)"
- "Extension not available in current PostgreSQL version"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings
- MEDIUM: extension inventory, shared_preload_libraries configuration

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix extension issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest dropping extensions without confirming application dependencies"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "Restore from snapshot if extension change causes issues"
- "Revert shared_preload_libraries parameter and reboot if needed"
- "If DROP EXTENSION was premature, recreate it"

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
