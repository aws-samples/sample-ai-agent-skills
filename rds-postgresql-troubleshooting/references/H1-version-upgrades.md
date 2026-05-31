---
title: "H1 — PostgreSQL Version Upgrades"
description: "Diagnose version upgrade failures and compatibility issues"
status: active
severity: HIGH
triggers:
  - "upgrade"
  - "version"
  - "pg_upgrade"
  - "major version"
---

## Phase 1 — Triage

MUST:
- Check valid upgrade targets: `aws rds describe-db-engine-versions --engine postgres --engine-version <current> --query 'DBEngineVersions[0].ValidUpgradeTarget[*].[EngineVersion,IsMajorVersionUpgrade]'`
- Check current version: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].EngineVersion'`
- Pre-upgrade checks:
  ```sql
  -- Check for incompatible extensions
  SELECT * FROM pg_extension;
  -- Check for reg* data types (problematic for pg_upgrade)
  SELECT c.relname, a.attname, t.typname FROM pg_class c JOIN pg_attribute a ON c.oid = a.attrelid
  JOIN pg_type t ON a.atttypid = t.oid WHERE t.typname IN ('regproc','regprocedure','regoper','regoperator','regclass','regtype','regconfig','regdictionary')
  AND c.relnamespace = 'public'::regnamespace;
  ```

SHOULD:
- Test upgrade on a snapshot restore first
- Check parameter group compatibility with new version
- Verify all extensions are available on target version
- Check for deprecated features in target version

## Phase 2 — Remediate

- Minor upgrade: `aws rds modify-db-instance --db-instance-identifier <id> --engine-version <target> --apply-immediately`
- Major upgrade: `aws rds modify-db-instance --db-instance-identifier <id> --engine-version 16.4 --allow-major-version-upgrade --apply-immediately`
- Create new parameter group for new version family
- Drop incompatible extensions before upgrade, recreate after
- Update read replicas after source upgrade
- Expect downtime during major upgrades (pg_upgrade runs internally)

## Safety Ratings
- GREEN: describe-db-instances, describe-db-engine-versions, describe-pending-maintenance-actions — read-only inspection
- YELLOW: create-db-snapshot (pre-upgrade) — recoverable operations
- RED: modify-db-instance --engine-version (upgrade), delete-db-instance — high-impact or destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Major version upgrade requires extended downtime and testing"
- "Fix requires parameter group family change"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, parameter group configuration
- MEDIUM: engine version details, upgrade targets, extension compatibility

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
