---
title: "E3 — Point-in-Time Recovery"
description: "Diagnose point-in-time recovery (PITR) issues for RDS Oracle"
status: active
severity: HIGH
triggers:
  - "point-in-time"
  - "PITR"
  - "restore to specific time"
  - "LatestRestorableTime"
  - "EarliestRestorableTime"
owner: devops-agent
objective: "Resolve point-in-time recovery issues for RDS Oracle instances"
context: "PITR creates a NEW instance restored to a specific second within the backup retention period. RDS uses automated backups + transaction logs (archived redo logs). PITR is not available if backup retention is 0. The restore time must be between EarliestRestorableTime and LatestRestorableTime."
---

## Phase 1 — Triage

MUST:
- Check restorable time range: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].{Earliest:EarliestRestorableTime,Latest:LatestRestorableTime}'`
- Check backup retention: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].BackupRetentionPeriod'`
- Verify the target restore time is within the valid range
- Check RDS events for backup issues: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --event-categories backup --duration 4320`

SHOULD:
- Check if automated backups are enabled (retention > 0)
- Verify the instance is not in a state that prevents PITR (e.g., being modified)
- Check if the instance is encrypted and KMS key is accessible

MAY:
- Check CloudTrail for recent destructive operations to determine the exact restore point
- Verify available storage for the restored instance

## Phase 2 — Remediate

MUST:
- Perform PITR:
  ```
  aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier <source-instance-id> \
    --target-db-instance-identifier <new-instance-id> \
    --restore-time "2024-01-15T10:30:00Z" \
    --db-instance-class <instance-class> \
    --db-subnet-group-name <subnet-group> \
    --vpc-security-group-ids <sg-id>
  ```
- Or restore to latest restorable time:
  ```
  aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier <source-instance-id> \
    --target-db-instance-identifier <new-instance-id> \
    --use-latest-restorable-time \
    --db-instance-class <instance-class>
  ```
- After restore: associate correct parameter group and option group, then reboot

SHOULD:
- Verify data integrity on restored instance:
  ```sql
  SELECT tablespace_name, status FROM dba_tablespaces;
  SELECT COUNT(*) FROM <critical_table>;
  ```
- Compare row counts between original and restored instance for key tables
- Update DNS/application to point to restored instance when ready

MAY:
- Use the restored instance for forensic analysis before switching production
- Document the incident timeline and restore point rationale

## Common Issues

- symptoms: "PITR not available — LatestRestorableTime is null"
  diagnosis: "Automated backups disabled (retention = 0) or instance just created."
  resolution: "Enable automated backups. Wait for first backup to complete."

- symptoms: "Restore time outside valid range"
  diagnosis: "Requested time is before EarliestRestorableTime or after LatestRestorableTime."
  resolution: "Choose a time within the valid range. Earliest is limited by retention period."

- symptoms: "LatestRestorableTime is significantly behind current time"
  diagnosis: "Transaction log archiving delayed. Possible I/O or storage issue."
  resolution: "Check storage space and I/O metrics. Resolve any underlying storage issues."

## Safety Ratings
- GREEN: describe-db-instances (EarliestRestorableTime, LatestRestorableTime), describe-events — read-only inspection
- YELLOW: restore-db-instance-to-point-in-time — creates new instance, recoverable
- RED: delete-db-instance (original), force-failover — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "PITR needed for disaster recovery or data corruption"
- "Fix requires switching application to restored instance"
- "LatestRestorableTime is significantly behind current time"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials (restored instance inherits source credentials), KMS key IDs
- HIGH: PITR restore point (reveals incident timeline)
- MEDIUM: backup retention configuration, restorable time range

## Prohibited Actions
- "NEVER suggest deleting the original database instance before confirming PITR restore is successful"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest PITR without verifying the target time is within the valid range"

## Phase 3 — Rollback
- "If restored instance has issues, delete it and retry PITR with a different restore point"
- "Keep original instance running until restored instance is fully validated"
- "Revert application connection strings if restored instance has problems"

## Output Format

```yaml
root_cause: "pitr — <specific_cause>"
evidence:
  - type: restorable_range
    content: "<EarliestRestorableTime to LatestRestorableTime>"
  - type: backup_config
    content: "<retention period and backup status>"
severity: HIGH
mitigation:
  immediate: "Perform PITR to the appropriate restore point"
  long_term: "Ensure adequate backup retention and monitor LatestRestorableTime"
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
