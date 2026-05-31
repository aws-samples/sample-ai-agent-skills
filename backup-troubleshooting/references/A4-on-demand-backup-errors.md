---
title: "A4 — On-Demand Backup Errors"
description: "Diagnose failures when creating on-demand (ad hoc) backups"
status: active
severity: MEDIUM
triggers:
  - "on-demand backup failed"
  - "manual backup error"
  - "start-backup-job failed"
  - "ad hoc backup"
  - "one-time backup"
owner: devops-agent
objective: "Identify why on-demand backup requests fail and resolve the issue"
context: "On-demand backups are created via the console or StartBackupJob API. They do NOT inherit lifecycle rules from backup plans — lifecycle must be specified explicitly. Common failures include missing parameters, IAM issues, vault lock conflicts, and unsupported resource types."
---

## Phase 1 — Triage

MUST:
- Check the error message from the StartBackupJob API call or console
- Describe the failed job if a job ID was created: `aws backup describe-backup-job --backup-job-id <job-id>`
- Verify the resource ARN is correct and the resource exists
- Check the IAM role: `aws iam get-role --role-name <role-name>`
- Verify the target vault exists: `aws backup describe-backup-vault --backup-vault-name <vault-name>`

SHOULD:
- Check if the vault has a lock with minimum retention that conflicts with the specified lifecycle
- Verify the resource type is supported by AWS Backup
- Check KMS key permissions if using a custom-encrypted vault
- Verify the backup role has permissions for the specific resource type

MAY:
- Check CloudTrail for the StartBackupJob event and error details
- Verify no SCP or IAM boundary is blocking the backup operation
- Check if the resource is in a state that allows backup (e.g., RDS available, EC2 running)

## Phase 2 — Remediate

MUST:
- Fix the identified issue (permissions, vault, lifecycle, resource state)
- Specify lifecycle parameters explicitly — on-demand backups do NOT inherit plan rules
- Retry the on-demand backup

SHOULD:
- Ensure the lifecycle meets vault lock minimum retention if applicable
- Use the correct IAM role with service-specific permissions
- Verify the vault's KMS key allows the backup role

MAY:
- Consider creating a backup plan for recurring backups instead of repeated on-demand
- Document the correct parameters for on-demand backups in runbooks

## Common Issues

- symptoms: "InvalidParameterValueException when calling StartBackupJob"
  diagnosis: "The lifecycle DeleteAfterDays is less than the vault lock minimum retention."
  resolution: "Set DeleteAfterDays to at least the vault lock minimum retention period."

- symptoms: "AccessDeniedException for StartBackupJob"
  diagnosis: "The caller lacks backup:StartBackupJob permission or the IAM role cannot be assumed."
  resolution: "Grant backup:StartBackupJob to the caller and ensure the role trust policy allows backup.amazonaws.com."

- symptoms: "On-demand backup has no lifecycle — recovery point never expires"
  diagnosis: "Lifecycle was not specified in the StartBackupJob call."
  resolution: "Specify Lifecycle with DeleteAfterDays and optionally MoveToColdStorageAfterDays."

- symptoms: "Unsupported resource type error"
  diagnosis: "The resource type is not supported by AWS Backup."
  resolution: "Verify the service is in the supported services list. Use native service backup if not supported."

## Output Format

```yaml
root_cause: "on_demand_backup_error — <specific_cause>"
evidence:
  - type: api_error
    content: "<error message from StartBackupJob>"
  - type: vault_config
    content: "<vault lock and encryption details>"
  - type: iam_role
    content: "<role permissions and trust policy>"
severity: MEDIUM
mitigation:
  immediate: "Fix parameters and retry the on-demand backup"
  long_term: "Create backup plans for recurring backup needs"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix permissions, vault, lifecycle, resource state | YELLOW | Configuration changes — review each |
| Specify lifecycle parameters explicitly | GREEN | Parameter specification — non-destructive |
| Retry on-demand backup | GREEN | Re-executes backup — non-destructive |
| Ensure lifecycle meets vault lock minimum retention | GREEN | Compliance verification |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- Vault lock conflicts require understanding compliance mode implications
- IAM role changes affect backup operations

## Data Sensitivity

- On-demand backup parameters reveal data protection urgency and scope
- Vault lock configuration reveals compliance requirements
- Resource ARNs reveal infrastructure being protected

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER create on-demand backups without lifecycle (creates recovery points that never expire)
- NEVER set lifecycle below vault lock minimum retention

## Phase 3 — Rollback

- If on-demand backup created with wrong lifecycle: update recovery point lifecycle or delete if within retention rules
- If IAM permissions were changed: revert to previous policy
- Document all on-demand backup operations for compliance audit trail

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
