---
title: "B2 — Cross-Account Restore"
description: "Diagnose failures when restoring backups in a different AWS account"
status: active
severity: HIGH
triggers:
  - "cross-account restore"
  - "restore in another account"
  - "restore from shared vault"
  - "cross-account recovery"
owner: devops-agent
objective: "Identify and resolve issues with restoring backup recovery points across AWS accounts"
context: "Cross-account restore requires AWS Organizations, proper vault access policies, KMS key sharing, and IAM roles in both accounts. The recovery point must be copied to the destination account's vault before restore, or the destination account must have access to the source vault."
---

## Phase 1 — Triage

MUST:
- Verify both accounts are in the same AWS Organization: `aws organizations describe-organization`
- Check if cross-account backup is enabled: `aws backup describe-region-settings`
- Verify the destination vault access policy: `aws backup get-backup-vault-access-policy --backup-vault-name <dest-vault>`
- Check if the recovery point exists in the destination account's vault
- Verify the restore IAM role in the destination account

SHOULD:
- Check the KMS key policy allows the destination account: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Verify the copy job completed successfully if the recovery point was copied: `aws backup list-copy-jobs --by-destination-vault-arn <dest-vault-arn>`
- Check CloudTrail in both accounts for error events
- Verify the destination account has the required service permissions

MAY:
- Check if SCP policies are blocking cross-account operations
- Verify network configuration (VPC, subnets) in the destination account
- Check service quotas in the destination account

## Phase 2 — Remediate

MUST:
- Ensure cross-account backup is enabled in the Organization management account
- Configure the destination vault access policy to allow the source account
- Ensure KMS key policies allow cross-account access
- Create or update the IAM restore role in the destination account

SHOULD:
- Copy the recovery point to the destination account's vault before restoring
- Use customer-managed KMS keys with explicit cross-account grants
- Document the cross-account restore procedure

MAY:
- Set up automated cross-account copy rules in backup plans
- Create standardized IAM roles for cross-account restore operations

## Common Issues

- symptoms: "AccessDeniedException when restoring from another account's vault"
  diagnosis: "The destination account lacks access to the source vault or KMS key."
  resolution: "Update vault access policy and KMS key policy to grant the destination account permissions."

- symptoms: "Cross-account backup features not available"
  diagnosis: "Cross-account backup is not enabled in the Organization settings."
  resolution: "Enable cross-account backup in the management account's AWS Backup settings."

- symptoms: "KMS decrypt error during cross-account restore"
  diagnosis: "The destination account's restore role cannot decrypt with the source account's KMS key."
  resolution: "Add kms:Decrypt and kms:CreateGrant permissions for the destination role in the KMS key policy."

## Output Format

```yaml
root_cause: "cross_account_restore_failure — <specific_cause>"
evidence:
  - type: organization
    content: "<organization and cross-account settings>"
  - type: vault_policy
    content: "<destination vault access policy>"
  - type: kms_policy
    content: "<KMS key policy for cross-account access>"
severity: HIGH
mitigation:
  immediate: "Fix cross-account permissions and retry restore"
  long_term: "Establish standardized cross-account backup and restore procedures"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable cross-account backup in Organization | RED | Organization-wide setting — affects all accounts |
| Configure destination vault access policy | YELLOW | Cross-account access change |
| Ensure KMS key policies allow cross-account | YELLOW | Encryption access change |
| Create/update IAM restore role in destination | YELLOW | IAM role change in another account |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- Organization-level settings affect all member accounts
- KMS key policy changes affect encryption access across accounts

## Data Sensitivity

- Cross-account restore moves data between AWS accounts — verify data classification allows this
- Vault access policies control who can access backup data across accounts
- KMS key sharing reveals encryption trust relationships between accounts

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER grant cross-account vault access without security review
- NEVER share KMS keys with accounts outside the organization

## Phase 3 — Rollback

- If cross-account backup was enabled incorrectly: disable in management account settings
- If vault access policy was changed: restore original policy
- If KMS key policy was modified: restore original key policy
- If IAM role was created incorrectly: delete the role in the destination account
- Document all cross-account changes for compliance audit trail

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
