---
title: "E2 — Snapshot Restore"
description: "Diagnose issues restoring RDS SQL Server instances from snapshots"
status: active
severity: HIGH
triggers:
  - "restore snapshot"
  - "restore from backup"
  - "restore failed"
  - "snapshot restore"
  - "new instance from snapshot"
owner: devops-agent
objective: "Resolve snapshot restore failures and configuration issues for RDS SQL Server"
context: "Restoring an RDS snapshot creates a NEW instance — it does not overwrite the existing one. The restored instance uses the default parameter group and security group unless specified. SQL Server-specific considerations include option group compatibility, collation preservation, and TDE key management."
---

## Phase 1 — Triage

MUST:
- Check snapshot status: `aws rds describe-db-snapshots --db-snapshot-identifier <snapshot-id>`
- Check snapshot details: `aws rds describe-db-snapshots --db-snapshot-identifier <snapshot-id> --query 'DBSnapshots[0].{Engine:Engine,Version:EngineVersion,Storage:AllocatedStorage,Encrypted:Encrypted}'`
- Check for restore errors in events: `aws rds describe-events --source-type db-instance --duration 1440`
- Verify target instance class availability: `aws rds describe-orderable-db-instance-options --engine <engine> --engine-version <version>`

SHOULD:
- Check if snapshot is encrypted and KMS key accessible: `aws kms describe-key --key-id <key-id>`
- Verify option group compatibility for the target engine version
- Check if cross-region restore (requires snapshot copy first)
- Verify VPC and subnet group for the restored instance

MAY:
- Check snapshot sharing permissions for cross-account restore
- Verify storage type compatibility

## Phase 2 — Remediate

MUST:
- Restore to a new instance:
  ```
  aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier <new-instance-id> \
    --db-snapshot-identifier <snapshot-id> \
    --db-instance-class <instance-class> \
    --db-subnet-group-name <subnet-group> \
    --vpc-security-group-ids <sg-id>
  ```
- After restore: associate correct parameter group and option group
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <new-instance-id> \
    --db-parameter-group-name <param-group> \
    --option-group-name <option-group> \
    --apply-immediately
  ```
- Reboot after parameter/option group change

SHOULD:
- Verify SQL Server data integrity after restore:
  ```sql
  SELECT name, state_desc, recovery_model_desc FROM sys.databases;
  DBCC CHECKDB('MyDatabase') WITH NO_INFOMSGS;
  ```
- Update application connection strings to point to new instance
- For TDE-encrypted databases: verify TDE option group is associated

MAY:
- Rename restored instance to match original (after deleting/renaming original)
- Test application connectivity before switching production traffic

## Common Issues

- symptoms: "Restore fails with 'incompatible option group'"
  diagnosis: "Snapshot's option group not compatible with target engine version."
  resolution: "Create a compatible option group for the target version and specify during restore."

- symptoms: "Restored instance has default parameter group"
  diagnosis: "Restore always uses default parameter group unless specified."
  resolution: "Associate the correct custom parameter group after restore and reboot."

- symptoms: "Cannot restore encrypted snapshot in different region"
  diagnosis: "Encrypted snapshots must be copied to target region first."
  resolution: "Copy snapshot to target region with a KMS key in that region, then restore."

## Safety Ratings
- GREEN: describe-db-snapshots, describe-orderable-db-instance-options, describe-events — read-only inspection
- YELLOW: restore-db-instance-from-db-snapshot, modify-db-instance — creates new instance, recoverable
- RED: delete-db-instance (original), delete-db-snapshot — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Snapshot restore needed for disaster recovery"
- "Fix requires switching application to restored instance"
- "Encrypted snapshot requires KMS key access"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials (restored instance inherits source credentials), KMS key IDs
- HIGH: snapshot contents (contain full database data)
- MEDIUM: snapshot metadata, option group configuration

## Prohibited Actions
- "NEVER suggest deleting the original database instance before confirming restore is successful"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"

## Phase 3 — Rollback
- "If restored instance has issues, delete it and retry restore with different parameters"
- "Keep original instance running until restored instance is fully validated"

## Output Format

```yaml
root_cause: "snapshot_restore — <specific_cause>"
evidence:
  - type: snapshot_details
    content: "<snapshot configuration>"
  - type: restore_error
    content: "<error message>"
severity: HIGH
mitigation:
  immediate: "Fix restore parameters and retry"
  long_term: "Document restore procedures and test regularly"
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
