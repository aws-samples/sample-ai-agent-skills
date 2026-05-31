---
title: "I1 — Version Upgrades"
description: "Diagnose SQL Server major and minor version upgrade issues on RDS SQL Server"
status: active
severity: HIGH
triggers:
  - "upgrade"
  - "major version"
  - "minor version"
  - "engine version"
  - "deprecation"
  - "upgrade path"
  - "upgrade failed"
owner: devops-agent
objective: "Resolve SQL Server version upgrade planning and execution issues on RDS SQL Server"
context: "RDS SQL Server supports major version upgrades (e.g., 2017 to 2019) and minor version upgrades (CU updates). Not all upgrade paths are direct. Major upgrades may require parameter group family changes. Upgrades cause downtime. AWS deprecates SQL Server versions on a schedule."
---

## Phase 1 — Triage

MUST:
- Check current version: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].EngineVersion'`
- Check valid upgrade targets:
  ```
  aws rds describe-db-engine-versions \
    --engine sqlserver-se \
    --engine-version <current-version> \
    --query 'DBEngineVersions[0].ValidUpgradeTarget[].{Version:EngineVersion,Auto:AutoUpgrade,Major:IsMajorVersionUpgrade}'
  ```
- Check for pending maintenance: `aws rds describe-pending-maintenance-actions --resource-identifier <instance-arn>`
- Check parameter group family compatibility: `aws rds describe-db-parameter-groups --db-parameter-group-name <param-group>`

SHOULD:
- Check option group compatibility with target version
- Check database compatibility level:
  ```sql
  SELECT name, compatibility_level FROM sys.databases;
  ```
- Test upgrade in non-production environment first
- Check for deprecated features:
  ```sql
  SELECT feature_name, feature_id FROM sys.dm_db_persisted_sku_features;
  ```

MAY:
- Check for objects using deprecated syntax
- Review application SQL for compatibility issues

## Phase 2 — Remediate

MUST:
- For minor version upgrade:
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --engine-version <target-version> \
    --apply-immediately
  ```
- For major version upgrade:
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --engine-version <target-version> \
    --allow-major-version-upgrade \
    --db-parameter-group-name <new-family-param-group> \
    --option-group-name <compatible-option-group> \
    --apply-immediately
  ```
- Create snapshot before upgrade:
  ```
  aws rds create-db-snapshot \
    --db-instance-identifier <instance-id> \
    --db-snapshot-identifier pre-upgrade-<date>
  ```

SHOULD:
- Plan for downtime (major upgrades can take 30+ minutes)
- Update database compatibility level after upgrade:
  ```sql
  ALTER DATABASE [MyDatabase] SET COMPATIBILITY_LEVEL = 150;  -- SQL Server 2019
  ```
- Update statistics after upgrade:
  ```sql
  EXEC sp_updatestats;
  ```
- Verify application functionality after upgrade

MAY:
- Use blue-green deployment for minimal downtime upgrades
- Schedule upgrade during maintenance window for automatic application

## Common Issues

- symptoms: "No valid upgrade target for current version"
  diagnosis: "Current version may require intermediate upgrade step."
  resolution: "Check describe-db-engine-versions for multi-step upgrade path."

- symptoms: "Upgrade failed — parameter group incompatible"
  diagnosis: "Parameter group family doesn't match target version."
  resolution: "Create new parameter group with target family (e.g., sqlserver-se-15.0). Migrate custom settings."

- symptoms: "Application errors after major upgrade"
  diagnosis: "Changed query optimizer behavior or deprecated features."
  resolution: "Set compatibility level to previous version temporarily. Test with new level. Use Query Store for plan regression."

## Safety Ratings
- GREEN: describe-db-instances, describe-db-engine-versions, describe-pending-maintenance-actions — read-only inspection
- YELLOW: create-db-snapshot (pre-upgrade) — recoverable operations
- RED: modify-db-instance --engine-version (upgrade), delete-db-instance — high-impact or destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Major version upgrade requires extended downtime"
- "Fix requires parameter group family change"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, parameter group configuration
- MEDIUM: engine version details, upgrade targets, compatibility assessment

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix upgrade issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest major version upgrade without creating a pre-upgrade snapshot"

## Phase 3 — Rollback
- "Restore from pre-upgrade snapshot if upgrade causes application issues"
- "Revert parameter group to previous family if new parameter group causes issues"

## Output Format

```yaml
root_cause: "version_upgrade — <specific_cause>"
evidence:
  - type: current_version
    content: "<current engine version>"
  - type: upgrade_targets
    content: "<valid upgrade targets>"
severity: HIGH
mitigation:
  immediate: "Fix upgrade prerequisites and retry"
  long_term: "Implement upgrade testing pipeline and version lifecycle management"
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
