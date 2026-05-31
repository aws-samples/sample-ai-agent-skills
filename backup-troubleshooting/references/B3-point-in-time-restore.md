---
title: "B3 — Point-in-Time Restore"
description: "Diagnose issues with point-in-time recovery (PITR) restores"
status: active
severity: HIGH
triggers:
  - "point-in-time restore"
  - "PITR"
  - "restore to specific time"
  - "continuous backup restore"
  - "restore to timestamp"
owner: devops-agent
objective: "Identify and resolve issues with point-in-time recovery restores for supported services"
context: "Point-in-time recovery (PITR) is only supported for S3, RDS, Aurora, and SAP HANA. It requires continuous backup to be enabled. PITR allows restoring to any second within the retention period. Common issues include unsupported services, time outside retention window, and continuous backup not enabled."
---

## Phase 1 — Triage

MUST:
- Verify the resource type supports continuous backup (S3, RDS, Aurora, SAP HANA only)
- Check if continuous backup is enabled for the resource in the backup plan: `aws backup get-backup-plan --backup-plan-id <plan-id>`
- Verify the requested restore time is within the retention period
- List continuous recovery points: `aws backup list-recovery-points-by-backup-vault --backup-vault-name <vault> --by-resource-type <type>`
- Check the recovery point's RestorePointCreationDate and CompletionDate

SHOULD:
- For RDS/Aurora, verify PITR is also enabled at the database level: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].LatestRestorableTime'`
- Check if the continuous backup has any gaps in the recovery timeline
- Verify the restore metadata includes the correct RestoreTime parameter
- Check IAM role permissions for the restore operation

MAY:
- Check CloudTrail for any events that may have disrupted continuous backup
- Verify the S3 bucket versioning is enabled (required for S3 PITR)
- Check for service-specific PITR limitations

## Phase 2 — Remediate

MUST:
- Ensure continuous backup is enabled in the backup plan for the resource
- Specify a valid RestoreTime within the retention window
- Use the correct restore metadata for the service type

SHOULD:
- For RDS, ensure automated backups are enabled (required for PITR)
- Verify the retention period covers the desired restore point
- Test PITR restores periodically using restore testing plans

MAY:
- Extend the continuous backup retention period if needed (up to 35 days for RDS/Aurora)
- Document PITR procedures for each supported service type

## Common Issues

- symptoms: "Continuous backup not available for this resource type"
  diagnosis: "PITR is only supported for S3, RDS, Aurora, and SAP HANA."
  resolution: "Use snapshot-based restore for unsupported services. Enable continuous backup for supported ones."

- symptoms: "Requested restore time is outside the recovery window"
  diagnosis: "The specified timestamp is before the earliest or after the latest restorable time."
  resolution: "Check LatestRestorableTime and EarliestRestorableTime. Choose a time within the window."

- symptoms: "RDS PITR fails — automated backups not enabled"
  diagnosis: "RDS PITR requires automated backups to be enabled with a retention period > 0."
  resolution: "Enable automated backups on the RDS instance with appropriate retention."

- symptoms: "S3 PITR fails — versioning not enabled"
  diagnosis: "S3 continuous backup requires bucket versioning to be enabled."
  resolution: "Enable versioning on the S3 bucket. PITR only covers changes after versioning is enabled."

## Output Format

```yaml
root_cause: "pitr_restore_failure — <specific_cause>"
evidence:
  - type: recovery_point
    content: "<continuous recovery point details>"
  - type: restore_time
    content: "<requested vs available restore window>"
  - type: service_config
    content: "<service-specific PITR configuration>"
severity: HIGH
mitigation:
  immediate: "Fix PITR configuration and retry with valid restore time"
  long_term: "Enable continuous backup and monitor retention windows"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure continuous backup is enabled | YELLOW | Changes backup plan — enables continuous data capture |
| Specify valid RestoreTime | GREEN | Parameter specification |
| Use correct restore metadata | GREEN | Parameter specification |
| Enable RDS automated backups | YELLOW | Changes RDS configuration |
| Extend continuous backup retention | YELLOW | Changes retention — may increase costs |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- PITR restore creates new resources — verify target environment
- Continuous backup changes affect data capture and storage costs

## Data Sensitivity

- PITR restores can recover data to any point in time — powerful data access capability
- Continuous backup captures all changes — contains complete data history
- Restore time selection may be driven by data breach or compliance investigation

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER disable continuous backup on resources with PITR compliance requirements
- NEVER restore to a time outside the valid recovery window

## Phase 3 — Rollback

- If continuous backup was enabled incorrectly: disable in backup plan (stops continuous capture)
- If PITR restore created incorrect resource: delete the restored resource
- If RDS automated backups were changed: restore original retention period
- Document all PITR operations for compliance audit trail

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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
