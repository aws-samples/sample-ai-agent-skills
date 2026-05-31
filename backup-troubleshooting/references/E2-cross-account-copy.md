---
title: "E2 — Cross-Account Copy Failures"
description: "Diagnose failures when copying backup recovery points to another AWS account"
status: active
severity: HIGH
triggers:
  - "cross-account copy"
  - "copy to another account"
  - "account copy failed"
  - "cross-account backup copy"
owner: devops-agent
objective: "Identify and resolve issues with cross-account backup copy operations"
context: "Cross-account copy requires AWS Organizations, enabled cross-account backup settings, a destination vault with an access policy allowing the source account, and KMS key sharing. The destination vault's KMS key must grant the source account's backup role decrypt and create-grant permissions."
---

## Phase 1 — Triage

MUST:
- Verify both accounts are in the same Organization: `aws organizations describe-organization`
- Check cross-account backup is enabled: `aws backup describe-region-settings`
- List failed copy jobs: `aws backup list-copy-jobs --by-state FAILED`
- Describe the failed copy job: `aws backup describe-copy-job --copy-job-id <copy-job-id>`
- Check the destination vault access policy (in destination account): `aws backup get-backup-vault-access-policy --backup-vault-name <dest-vault>`

SHOULD:
- Verify the destination vault's KMS key allows the source account
- Check the source account's backup IAM role permissions
- Verify the copy action configuration in the backup plan
- Check CloudTrail in both accounts for copy-related errors

MAY:
- Check if SCP policies block cross-account operations
- Verify the recovery point is in a copyable state (COMPLETED)
- Check for service-specific cross-account copy limitations

## Phase 2 — Remediate

MUST:
- Enable cross-account backup in the management account
- Configure the destination vault access policy to allow `backup:CopyIntoBackupVault` from the source account
- Grant the source account's backup role permissions on the destination vault's KMS key
- Retry the copy operation

SHOULD:
- Use `aws:PrincipalOrgID` condition in vault policies for organization-wide access
- Configure automated copy actions in backup plan rules
- Test cross-account copy before relying on it for DR

MAY:
- Set up centralized monitoring for cross-account copy jobs
- Implement standardized vault policies across the organization
- Create runbooks for cross-account restore procedures

## Common Issues

- symptoms: "AccessDeniedException on CopyIntoBackupVault"
  diagnosis: "The destination vault access policy does not allow the source account."
  resolution: "Add a vault policy statement allowing backup:CopyIntoBackupVault from the source account principal."

- symptoms: "KMS error during cross-account copy"
  diagnosis: "The destination vault's KMS key does not grant the source account's backup role access."
  resolution: "Update the KMS key policy to grant kms:Decrypt, kms:CreateGrant, kms:GenerateDataKey to the source backup role."

- symptoms: "Cross-account copy not available"
  diagnosis: "Accounts are not in the same Organization or cross-account backup is not enabled."
  resolution: "Verify Organization membership and enable cross-account backup in management account settings."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Update destination vault access policy | YELLOW | Changes vault access — review policy carefully |
| Update KMS key policy for cross-account | YELLOW | Grants external account access to encryption key |
| Enable cross-account backup in management account | YELLOW | Organization-wide setting change |
| Retry copy operation | GREEN | Non-destructive — copies data to destination |
| Use aws:PrincipalOrgID condition in vault policies | GREEN | Security best practice — restricts to org |

## Escalation Conditions

- Vault contains compliance-critical backups
- Fix requires vault lock modification
- Task replicates production database backups cross-account
- KMS key policy change affects multiple accounts in the organization
- Cross-account copy is part of a disaster recovery strategy with strict RPO

## Data Sensitivity

- **Classification: HIGH**
- Cross-account vault policies reveal organizational trust relationships
- KMS key sharing exposes encryption trust boundaries between accounts
- Copy job details reveal DR strategy and data replication patterns
- Source/destination account IDs and vault ARNs are sensitive infrastructure details

## Prohibited Actions

- **NEVER** suggest deleting a vault with compliance lock
- **NEVER** suggest reducing retention below compliance requirements
- **NEVER** remove cross-account vault policies without confirming destination account no longer needs access
- **NEVER** delete or disable KMS keys used by destination vaults with active recovery points

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Updated destination vault access policy | Revert vault policy to previous version using saved policy JSON |
| Updated KMS key policy for cross-account | Remove the cross-account grant from the KMS key policy |
| Enabled cross-account backup in management account | Disable cross-account backup in management account settings if not needed |

## Output Format

```yaml
root_cause: "cross_account_copy — <specific_cause>"
evidence:
  - type: copy_job
    content: "<copy job details and error>"
  - type: vault_policy
    content: "<destination vault access policy>"
  - type: kms_policy
    content: "<destination KMS key policy>"
severity: HIGH
mitigation:
  immediate: "Fix vault and KMS policies, retry copy"
  long_term: "Implement standardized cross-account backup policies"
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
