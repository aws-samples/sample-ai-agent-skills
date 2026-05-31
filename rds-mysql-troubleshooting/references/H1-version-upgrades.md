---
title: "H1 — MySQL Version Upgrades"
description: "Diagnose version upgrade failures and compatibility issues"
status: active
severity: HIGH
triggers:
  - "upgrade"
  - "version"
  - "5.7 to 8.0"
  - "deprecated"
  - "pre-upgrade check"
---

## Phase 1 — Triage

MUST:
- Check valid upgrade targets: `aws rds describe-db-engine-versions --engine mysql --engine-version <current> --query 'DBEngineVersions[0].ValidUpgradeTarget[*].[EngineVersion,IsMajorVersionUpgrade]'`
- Check current version: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].EngineVersion'`
- For 5.7→8.0: run pre-upgrade checks:
  ```sql
  -- Check for deprecated features
  SELECT TABLE_SCHEMA, TABLE_NAME, ENGINE FROM information_schema.TABLES WHERE ENGINE = 'MyISAM' AND TABLE_SCHEMA NOT IN ('mysql','sys','performance_schema');
  -- Check for utf8mb3 usage
  SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, CHARACTER_SET_NAME FROM information_schema.COLUMNS WHERE CHARACTER_SET_NAME = 'utf8' AND TABLE_SCHEMA NOT IN ('mysql','sys','performance_schema','information_schema');
  ```
- Check for pending modifications: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].PendingModifiedValues'`

SHOULD:
- Test upgrade on a snapshot restore first
- Check parameter group compatibility with new version
- Review MySQL 8.0 breaking changes (default auth plugin, GROUP BY behavior)

## Phase 2 — Remediate

- Minor upgrade: `aws rds modify-db-instance --db-instance-identifier <id> --engine-version <target> --apply-immediately`
- Major upgrade: `aws rds modify-db-instance --db-instance-identifier <id> --engine-version 8.0.35 --allow-major-version-upgrade --apply-immediately`
- Create new parameter group for new version family if needed
- Update read replicas after source upgrade
- Downtime expected during upgrade — plan maintenance window

## Safety Ratings
- GREEN: describe-db-instances, describe-db-engine-versions, describe-pending-maintenance-actions — read-only inspection
- YELLOW: create-db-snapshot (pre-upgrade) — recoverable operations
- RED: modify-db-instance --engine-version (upgrade), delete-db-instance — high-impact or destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Major version upgrade (5.7→8.0) requires extended downtime and testing"
- "Fix requires parameter group family change"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, parameter group configuration
- MEDIUM: engine version details, upgrade targets, compatibility assessment

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix upgrade issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest major version upgrade without creating a pre-upgrade snapshot"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "Restore from pre-upgrade snapshot if upgrade causes application issues"
- "Revert parameter group to previous family if new parameter group causes issues"

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
