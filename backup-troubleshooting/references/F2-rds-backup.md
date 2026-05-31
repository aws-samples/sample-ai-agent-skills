---
title: "F2 — RDS Backup Issues"
description: "Diagnose RDS and Aurora database backup failures"
status: active
severity: HIGH
triggers:
  - "RDS backup failed"
  - "RDS snapshot failed"
  - "Aurora backup"
  - "database backup"
  - "RDS restore"
owner: devops-agent
objective: "Identify and resolve RDS/Aurora-specific backup and restore issues"
context: "AWS Backup creates RDS snapshots for backup. RDS has its own automated backup system (retention up to 35 days) that coexists with AWS Backup. AWS Backup provides longer retention and cross-account/cross-region copy. RDS instances must be in 'available' state for backup. Aurora cluster backups include all instances."
---

## Phase 1 — Triage

MUST:
- Describe the failed backup job: `aws backup describe-backup-job --backup-job-id <job-id>`
- Check the RDS instance state: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].DBInstanceStatus'`
- Verify the backup role has RDS permissions
- Check if the RDS instance is encrypted and verify KMS key access
- Check for concurrent snapshot operations: `aws rds describe-db-snapshots --db-instance-identifier <id> --query 'DBSnapshots[?Status==`creating`]'`

SHOULD:
- Check RDS automated backup configuration: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].BackupRetentionPeriod'`
- Verify the RDS instance is not in a maintenance window
- Check if the instance is a read replica (some backup operations differ)
- For Aurora, check the cluster status: `aws rds describe-db-clusters --db-cluster-identifier <id>`

MAY:
- Check RDS event logs for backup-related events: `aws rds describe-events --source-identifier <id> --source-type db-instance --duration 1440`
- Verify storage space is sufficient for snapshot creation
- Check if the instance uses a custom parameter group that affects backups

## Phase 2 — Remediate

MUST:
- Ensure the RDS instance is in 'available' state before backup
- Grant the backup role rds:CreateDBSnapshot, rds:DescribeDBInstances, rds:ListTagsForResource permissions
- Resolve KMS key permission issues for encrypted instances

SHOULD:
- Schedule AWS Backup outside of RDS maintenance windows
- Ensure RDS automated backups are enabled (required for PITR)
- Avoid concurrent manual snapshots during AWS Backup windows

MAY:
- Use continuous backup for RDS/Aurora PITR through AWS Backup
- Configure backup plan to handle Aurora clusters and individual instances separately
- Monitor RDS backup duration for capacity planning

## Common Issues

- symptoms: "RDS backup fails — instance in modifying state"
  diagnosis: "The RDS instance is being modified (scaling, patching, parameter change)."
  resolution: "Wait for the instance to return to 'available' state. Schedule backups outside maintenance windows."

- symptoms: "Cannot create snapshot — maximum snapshot limit reached"
  diagnosis: "The account has reached the RDS manual snapshot quota (default 100 per region)."
  resolution: "Delete old manual snapshots or request a quota increase via Service Quotas."

- symptoms: "RDS backup succeeds but PITR not available through AWS Backup"
  diagnosis: "Continuous backup is not enabled in the backup plan for RDS."
  resolution: "Enable continuous backup in the backup plan rule for RDS resources."

- symptoms: "Aurora cluster backup fails"
  diagnosis: "The backup targets an Aurora instance ARN instead of the cluster ARN."
  resolution: "Use the Aurora cluster ARN for backup selection, not individual instance ARNs."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Grant backup role RDS permissions | GREEN | Standard permission grant |
| Resolve KMS key permissions for encrypted instances | YELLOW | Expands key access — review scope |
| Schedule backups outside maintenance windows | GREEN | Configuration change — non-destructive |
| Enable continuous backup for PITR | GREEN | Adds protection — non-destructive |
| Delete old manual snapshots for quota | YELLOW | Permanent deletion — verify snapshots are no longer needed |

## Escalation Conditions

- Vault contains compliance-critical backups
- Fix requires vault lock modification
- RDS instance is a production database with strict RPO/RTO requirements
- Backup failure affects compliance reporting or audit trail
- Aurora cluster backup impacts multiple dependent services

## Data Sensitivity

- **Classification: HIGH**
- RDS snapshots contain full database contents including sensitive application data
- Backup configurations reveal database architecture and retention policies
- KMS key usage reveals encryption strategy for databases
- RDS event logs may contain connection details and operational patterns

## Prohibited Actions

- **NEVER** suggest deleting a vault with compliance lock
- **NEVER** suggest reducing retention below compliance requirements
- **NEVER** suggest deleting RDS snapshots without confirming they are not needed for compliance or recovery
- **NEVER** suggest disabling RDS automated backups on a production instance

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Granted backup role RDS permissions | Remove the added RDS permissions from the role |
| Updated KMS key policy | Revert KMS key policy to previous version |
| Enabled continuous backup in plan | Disable continuous backup rule if not needed |
| Deleted old manual snapshots | Cannot be undone — snapshots are permanently deleted |

## Output Format

```yaml
root_cause: "rds_backup — <specific_cause>"
evidence:
  - type: backup_job
    content: "<job details and error>"
  - type: rds_state
    content: "<RDS instance/cluster status>"
  - type: permissions
    content: "<IAM role RDS permissions>"
severity: HIGH
mitigation:
  immediate: "Fix RDS state/permissions and retry backup"
  long_term: "Schedule backups outside maintenance windows and enable continuous backup"
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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
