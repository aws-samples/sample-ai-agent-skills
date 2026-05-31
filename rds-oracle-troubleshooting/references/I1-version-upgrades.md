---
title: "I1 — Version Upgrades"
description: "Diagnose Oracle major and minor version upgrade issues on RDS Oracle"
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
objective: "Resolve Oracle version upgrade planning and execution issues on RDS Oracle"
context: "RDS Oracle supports major version upgrades (e.g., 12.1 to 19) and minor version upgrades (e.g., 19.0.0.0.ru-2023-01 to 19.0.0.0.ru-2023-04). Not all upgrade paths are direct. Major upgrades may require parameter group family changes. Upgrades cause downtime. AWS deprecates Oracle versions on a schedule."
---

## Phase 1 — Triage

MUST:
- Check current version: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].EngineVersion'`
- Check valid upgrade targets:
  ```
  aws rds describe-db-engine-versions \
    --engine oracle-ee \
    --engine-version <current-version> \
    --query 'DBEngineVersions[0].ValidUpgradeTarget[].{Version:EngineVersion,Auto:AutoUpgrade,Major:IsMajorVersionUpgrade}'
  ```
- Check for pending maintenance: `aws rds describe-pending-maintenance-actions --resource-identifier <instance-arn>`
- Check parameter group family compatibility: `aws rds describe-db-parameter-groups --db-parameter-group-name <param-group>`

SHOULD:
- Check option group compatibility with target version
- Review Oracle upgrade documentation for known issues
- Check for deprecated features in the target version:
  ```sql
  SELECT name, value FROM v$parameter WHERE name IN ('compatible','optimizer_features_enable');
  ```
- Test upgrade in non-production environment first

MAY:
- Check for invalid objects that may need recompilation:
  ```sql
  SELECT owner, object_type, object_name, status FROM dba_objects WHERE status = 'INVALID';
  ```
- Review application SQL for deprecated syntax

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
- Recompile invalid objects after upgrade:
  ```sql
  EXEC UTL_RECOMP.RECOMP_SERIAL();
  ```
- Gather fresh statistics after upgrade:
  ```sql
  EXEC DBMS_STATS.GATHER_DICTIONARY_STATS;
  EXEC DBMS_STATS.GATHER_FIXED_OBJECTS_STATS;
  ```
- Verify application functionality after upgrade

MAY:
- Use blue-green deployment for minimal downtime upgrades
- Schedule upgrade during maintenance window for automatic application

## Common Issues

- symptoms: "No valid upgrade target for current version"
  diagnosis: "Current version may require intermediate upgrade step."
  resolution: "Check describe-db-engine-versions for multi-step upgrade path (e.g., 12.1 → 12.2 → 19)."

- symptoms: "Upgrade failed — parameter group incompatible"
  diagnosis: "Parameter group family doesn't match target version."
  resolution: "Create new parameter group with target family (e.g., oracle-ee-19). Migrate custom settings."

- symptoms: "Application errors after major upgrade"
  diagnosis: "Deprecated features, changed optimizer behavior, or invalid objects."
  resolution: "Recompile invalid objects. Check optimizer_features_enable. Test with target version optimizer."

## Safety Ratings
- GREEN: describe-db-instances, describe-db-engine-versions, describe-pending-maintenance-actions, describe-db-parameter-groups, SELECT from v$parameter/dba_objects — read-only inspection
- YELLOW: create-db-snapshot (pre-upgrade), UTL_RECOMP.RECOMP_SERIAL — recoverable operations
- RED: modify-db-instance --engine-version (upgrade), delete-db-instance — high-impact or destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Major version upgrade requires extended downtime"
- "Fix requires parameter group family change"
- "Upgrade path requires intermediate version steps"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, parameter group configuration
- MEDIUM: engine version details, upgrade targets, invalid object inventory

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix upgrade issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest major version upgrade without creating a pre-upgrade snapshot"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest upgrading production without testing in non-production first"

## Phase 3 — Rollback
- "Restore from pre-upgrade snapshot if upgrade causes application issues"
- "If upgrade fails, instance may be in an inconsistent state — restore from snapshot"
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
