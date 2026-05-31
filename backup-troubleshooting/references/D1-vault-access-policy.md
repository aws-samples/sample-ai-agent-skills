---
title: "D1 — Vault Access Policy Issues"
description: "Diagnose backup vault access policy denials and misconfigurations"
status: active
severity: HIGH
triggers:
  - "vault access denied"
  - "vault policy"
  - "cannot access vault"
  - "vault permissions"
  - "backup vault access"
owner: devops-agent
objective: "Identify and resolve backup vault access policy issues preventing backup or restore operations"
context: "Backup vault access policies control who can access recovery points in the vault. They use IAM policy syntax with principal, action, resource, and condition elements. Without a policy, only the vault creator account has access. Cross-account access requires explicit vault policies."
---

## Phase 1 — Triage

MUST:
- Get the vault access policy: `aws backup get-backup-vault-access-policy --backup-vault-name <vault-name>`
- If no policy exists, note that only the vault owner account has access
- Check the specific error message from the failed operation
- Verify the principal (IAM role/user) attempting access
- Check the vault exists: `aws backup describe-backup-vault --backup-vault-name <vault-name>`

SHOULD:
- Analyze the vault policy for deny statements that may block access
- Check if the principal's IAM policy allows the backup action
- Verify condition keys in the vault policy (e.g., aws:SourceAccount, aws:PrincipalOrgID)
- Check if the vault is in the expected region

MAY:
- Check CloudTrail for AccessDenied events related to the vault
- Verify no SCP is blocking backup vault operations
- Check if the vault has a lock that restricts certain operations

## Phase 2 — Remediate

MUST:
- Update the vault access policy to grant required permissions: `aws backup put-backup-vault-access-policy --backup-vault-name <vault-name> --policy <policy-json>`
- Ensure both the vault policy AND the IAM principal policy allow the action
- Test access after policy update

SHOULD:
- Use least-privilege principles in vault policies
- Include condition keys for security (aws:PrincipalOrgID for cross-account)
- Document vault access policies and their intended purpose

MAY:
- Use AWS Organizations SCPs to enforce vault policy standards
- Implement vault policy templates for consistent configuration

## Common Issues

- symptoms: "AccessDeniedException when accessing recovery points in another account's vault"
  diagnosis: "The vault has no access policy or the policy does not include the requesting account."
  resolution: "Add a vault access policy granting the source account backup:CopyIntoBackupVault or other required actions."

- symptoms: "Cannot delete recovery points from vault"
  diagnosis: "The vault policy has an explicit deny on backup:DeleteRecoveryPoint, or vault lock prevents deletion."
  resolution: "Check vault policy for deny statements. Check vault lock minimum retention period."

- symptoms: "Vault policy update fails"
  diagnosis: "The policy JSON is malformed or references invalid principals/actions."
  resolution: "Validate the policy JSON. Use valid backup service actions and correct principal ARNs."

## Output Format

```yaml
root_cause: "vault_access_policy — <specific_cause>"
evidence:
  - type: vault_policy
    content: "<current vault access policy>"
  - type: principal
    content: "<requesting principal and their IAM permissions>"
  - type: error
    content: "<specific access denied error>"
severity: HIGH
mitigation:
  immediate: "Update vault access policy to grant required permissions"
  long_term: "Implement standardized vault policies with least-privilege access"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Update vault access policy | YELLOW | Changes who can access recovery points |
| Ensure both vault and IAM policies allow action | YELLOW | Dual permission change |
| Test access after policy update | GREEN | Verification only |
| Use least-privilege in vault policies | GREEN | Security best practice |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- Vault access policy changes affect who can access all recovery points in the vault
- Cross-account vault access requires security review

## Data Sensitivity

- Vault access policies control access to all backup data in the vault
- Policy changes may expose recovery points to new principals
- Cross-account access reveals organizational trust relationships

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER grant wildcard principal access to backup vaults
- NEVER remove vault access policies without understanding who needs access

## Phase 3 — Rollback

- If vault access policy was changed incorrectly: restore original policy using `aws backup put-backup-vault-access-policy`
- If policy was deleted: recreate with original permissions
- Document all vault policy changes for compliance audit trail

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
