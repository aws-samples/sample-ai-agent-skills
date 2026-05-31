---
title: "G1 — KMS Encryption Issues"
description: "Diagnose KMS encryption-related failures in AWS Backup operations"
status: active
severity: HIGH
triggers:
  - "KMS error"
  - "encryption failed"
  - "decrypt error"
  - "key access denied"
  - "GenerateDataKey failed"
  - "CreateGrant failed"
owner: devops-agent
objective: "Identify and resolve KMS encryption issues affecting backup and restore operations"
context: "Every backup vault is encrypted with a KMS key (set at creation, immutable). Backup operations require the IAM role to have KMS permissions on both the source resource's key and the vault's key. Cross-account and cross-region operations require KMS key policies to grant access to external principals. Key deletion or disabling causes permanent data loss risk."
---

## Phase 1 — Triage

MUST:
- Check the backup/restore job error message for KMS-specific errors
- Identify the KMS key used by the vault: `aws backup describe-backup-vault --backup-vault-name <vault-name>`
- Check the KMS key status: `aws kms describe-key --key-id <key-id>`
- Check the KMS key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Verify the backup role has KMS permissions: `aws kms list-grants --key-id <key-id>`

SHOULD:
- Check if the source resource uses a different KMS key than the vault
- Verify the key is not disabled, pending deletion, or in an unusable state
- Check for key rotation issues
- Verify cross-account/cross-region KMS key policies if applicable

MAY:
- Check CloudTrail for KMS-related denied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com --max-results 20`
- Verify no SCP blocks KMS operations
- Check if the key has usage conditions that restrict the backup role

## Phase 2 — Remediate

MUST:
- Grant the backup role kms:Decrypt, kms:GenerateDataKey, kms:CreateGrant, kms:DescribeKey on the vault's KMS key
- For encrypted source resources, grant kms:Decrypt on the source resource's KMS key
- Ensure the KMS key is enabled and not pending deletion

SHOULD:
- Use customer-managed KMS keys for backup vaults (more control over key policies)
- For cross-account operations, add explicit grants in the KMS key policy for the external account's backup role
- For cross-region copy, ensure a KMS key exists in the destination region with appropriate policies

MAY:
- Implement KMS key policies using aws:PrincipalOrgID for organization-wide access
- Set up CloudWatch alarms for KMS key state changes
- Document KMS key dependencies for backup infrastructure

## Common Issues

- symptoms: "AccessDeniedException: User is not authorized to perform kms:GenerateDataKey"
  diagnosis: "The backup role lacks kms:GenerateDataKey permission on the vault's KMS key."
  resolution: "Add kms:GenerateDataKey and kms:Decrypt to the KMS key policy for the backup role."

- symptoms: "KMS key is disabled or pending deletion"
  diagnosis: "The KMS key used by the vault has been disabled or scheduled for deletion."
  resolution: "Re-enable the key or cancel deletion. If deleted, recovery points in the vault are unrecoverable."

- symptoms: "Cross-account copy fails with KMS error"
  diagnosis: "The destination vault's KMS key does not allow the source account's backup role."
  resolution: "Update the destination KMS key policy to grant kms:Decrypt, kms:CreateGrant to the source backup role ARN."

- symptoms: "Cannot create grant on AWS-managed key for cross-account"
  diagnosis: "AWS-managed keys (aws/backup) do not support cross-account grants."
  resolution: "Use a customer-managed KMS key for vaults that need cross-account access."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Grant backup role KMS permissions (Decrypt, GenerateDataKey) | YELLOW | Expands key access — review scope carefully |
| Add cross-account grants to KMS key policy | YELLOW | Grants external account access to encryption key |
| Re-enable disabled KMS key | YELLOW | Restores access to encrypted data — verify intent |
| Cancel KMS key deletion | RED | Time-sensitive — key deletion is irreversible after waiting period |
| Implement aws:PrincipalOrgID condition | GREEN | Security best practice — restricts to org |

## Escalation Conditions

- Vault contains compliance-critical backups
- Fix requires vault lock modification
- KMS key is pending deletion and recovery points depend on it
- Key policy change affects multiple vaults or cross-account operations
- KMS key is shared across multiple AWS services beyond backup

## Data Sensitivity

- **Classification: HIGH**
- KMS key policies reveal encryption trust relationships and access boundaries
- Key grants expose which principals can decrypt backup data
- Cross-account key sharing reveals organizational trust topology
- Key state changes can cause permanent data loss for encrypted recovery points

## Prohibited Actions

- **NEVER** suggest deleting a vault with compliance lock
- **NEVER** suggest reducing retention below compliance requirements
- **NEVER** suggest disabling or scheduling deletion of a KMS key that encrypts active backup vaults
- **NEVER** suggest using AWS-managed keys for vaults that require cross-account access

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Added KMS grants for backup role | Remove the specific grants from the KMS key policy |
| Added cross-account grants to KMS key | Remove the cross-account principal from the key policy |
| Re-enabled a disabled KMS key | Disable the key again if re-enabling was not intended |
| Cancelled KMS key deletion | Re-schedule key deletion if cancellation was not intended |

## Output Format

```yaml
root_cause: "kms_encryption — <specific_cause>"
evidence:
  - type: kms_key
    content: "<key status, policy, and grants>"
  - type: backup_role
    content: "<role KMS permissions>"
  - type: error
    content: "<specific KMS error message>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key policy and grants"
  long_term: "Use customer-managed keys with documented policies for backup infrastructure"
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
