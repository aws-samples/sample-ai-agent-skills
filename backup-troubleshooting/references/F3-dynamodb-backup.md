---
title: "F3 — DynamoDB Backup Issues"
description: "Diagnose DynamoDB table backup and restore failures"
status: active
severity: HIGH
triggers:
  - "DynamoDB backup failed"
  - "DynamoDB restore"
  - "table backup"
  - "DynamoDB PITR"
  - "DynamoDB recovery"
owner: devops-agent
objective: "Identify and resolve DynamoDB-specific backup and restore issues"
context: "AWS Backup supports DynamoDB table backups including full table data, provisioned capacity settings, LSI/GSI, streams, and TTL configuration. DynamoDB also has native PITR (point-in-time recovery) which is separate from AWS Backup continuous backup. DynamoDB backups support cold storage transition. Restores always create a new table."
---

## Phase 1 — Triage

MUST:
- Describe the failed backup job: `aws backup describe-backup-job --backup-job-id <job-id>`
- Check the DynamoDB table status: `aws dynamodb describe-table --table-name <table-name> --query 'Table.TableStatus'`
- Verify the backup role has DynamoDB permissions
- Check if the table uses customer-managed KMS encryption: `aws dynamodb describe-table --table-name <table-name> --query 'Table.SSEDescription'`
- Verify the table exists and is in ACTIVE state

SHOULD:
- Check DynamoDB native PITR status: `aws dynamodb describe-continuous-backups --table-name <table-name>`
- Verify the table size (large tables take longer to back up)
- Check if global tables are involved (backup behavior differs)
- Verify the backup plan's advanced backup settings for DynamoDB

MAY:
- Check DynamoDB native backup list: `aws dynamodb list-backups --table-name <table-name>`
- Verify DynamoDB Streams configuration if relevant
- Check for table-level encryption key rotation issues

## Phase 2 — Remediate

MUST:
- Ensure the backup role has dynamodb:CreateBackup, dynamodb:DescribeTable, dynamodb:ListTagsOfResource permissions
- Grant KMS permissions if the table uses customer-managed encryption
- Ensure the table is in ACTIVE state before backup

SHOULD:
- Enable DynamoDB advanced backup features in AWS Backup settings for cold storage support
- Configure appropriate lifecycle rules (DynamoDB supports cold storage transition)
- Use AWS Backup for centralized management instead of native DynamoDB backups alone

MAY:
- Enable native PITR as an additional protection layer
- Monitor DynamoDB backup sizes for cost optimization
- Implement table-level backup tagging for compliance tracking

## Common Issues

- symptoms: "DynamoDB backup fails with permission error"
  diagnosis: "The backup role lacks dynamodb:CreateBackup or dynamodb:DescribeTable permissions."
  resolution: "Attach AWSBackupServiceRolePolicyForBackup or add specific DynamoDB permissions."

- symptoms: "DynamoDB restore creates table but GSIs are missing"
  diagnosis: "GSIs are restored but may take time to backfill. Check the table description for GSI status."
  resolution: "Wait for GSI backfill to complete. GSI status will show CREATING then ACTIVE."

- symptoms: "Cannot transition DynamoDB backup to cold storage"
  diagnosis: "Advanced backup features are not enabled for DynamoDB in AWS Backup settings."
  resolution: "Enable DynamoDB advanced features in AWS Backup settings (region-level setting)."

- symptoms: "DynamoDB restore fails — table name already exists"
  diagnosis: "DynamoDB restore always creates a new table. A table with the target name already exists."
  resolution: "Specify a unique table name for the restore. Rename or delete the existing table if needed."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Grant backup role DynamoDB permissions | GREEN | Standard permission grant |
| Grant KMS permissions for encrypted tables | YELLOW | Expands key access — review scope |
| Enable DynamoDB advanced backup features | GREEN | Region-level setting — enables cold storage |
| Enable native PITR as additional protection | GREEN | Adds protection — non-destructive |
| Rename or delete existing table for restore | RED | Destructive — table deletion is permanent |

## Escalation Conditions

- Vault contains compliance-critical backups
- Fix requires vault lock modification
- DynamoDB table serves production traffic with strict availability requirements
- Table is a global table with cross-region replication
- Backup failure affects compliance reporting or data retention obligations

## Data Sensitivity

- **Classification: HIGH**
- DynamoDB backups contain full table data including all items and attributes
- Table configurations reveal application data model and access patterns
- GSI/LSI definitions expose query patterns and data relationships
- KMS key configurations reveal encryption strategy for application data

## Prohibited Actions

- **NEVER** suggest deleting a vault with compliance lock
- **NEVER** suggest reducing retention below compliance requirements
- **NEVER** suggest deleting a production DynamoDB table to resolve a restore name conflict
- **NEVER** suggest disabling PITR on a table that requires continuous backup for compliance

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Granted backup role DynamoDB permissions | Remove the added DynamoDB permissions from the role |
| Updated KMS key policy | Revert KMS key policy to previous version |
| Enabled DynamoDB advanced features | Disable advanced features in AWS Backup settings if not needed |
| Deleted/renamed existing table for restore | Restore the original table from its most recent backup |

## Output Format

```yaml
root_cause: "dynamodb_backup — <specific_cause>"
evidence:
  - type: backup_job
    content: "<job details and error>"
  - type: table_state
    content: "<DynamoDB table status and configuration>"
  - type: permissions
    content: "<IAM role DynamoDB permissions>"
severity: HIGH
mitigation:
  immediate: "Fix permissions/configuration and retry backup"
  long_term: "Enable advanced features and configure lifecycle rules"
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
