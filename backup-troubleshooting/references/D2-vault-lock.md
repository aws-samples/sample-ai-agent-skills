---
title: "D2 — Vault Lock Issues"
description: "Diagnose backup vault lock configuration problems and compliance mode implications"
status: active
severity: CRITICAL
triggers:
  - "vault lock"
  - "compliance mode"
  - "governance mode"
  - "cannot delete recovery point"
  - "vault immutable"
  - "retention lock"
owner: devops-agent
objective: "Understand vault lock state, resolve lock-related issues, and prevent irreversible misconfigurations"
context: "Vault lock enforces retention policies. Governance mode can be removed by authorized users. Compliance mode is IRREVERSIBLE after the cooling-off period — it cannot be removed by anyone, including root and AWS Support. The cooling-off period is 3-36 days. During cooling-off, the lock can still be removed."
---

## Phase 1 — Triage

MUST:
- Describe the vault to check lock status: `aws backup describe-backup-vault --backup-vault-name <vault-name>`
- Check the Locked, LockDate, MinRetentionDays, MaxRetentionDays fields
- Determine if the lock is in governance or compliance mode
- Check if the vault is still in the cooling-off period (compliance mode only)
- Verify the specific error when operations fail against the locked vault

SHOULD:
- Check CloudTrail for the PutBackupVaultLockConfiguration event to see who applied the lock
- Verify the MinRetentionDays and MaxRetentionDays values
- Check if backup plan lifecycle rules comply with the vault lock retention
- Identify all backup plans targeting this vault

MAY:
- List all recovery points in the vault: `aws backup list-recovery-points-by-backup-vault --backup-vault-name <vault-name>`
- Check if any recovery points have lifecycle conflicts with the lock
- Review organizational policies for vault lock requirements

## Phase 2 — Remediate

MUST:
- If governance mode: remove with `aws backup delete-backup-vault-lock-configuration --backup-vault-name <vault-name>` (requires backup:DeleteBackupVaultLockConfiguration permission)
- If compliance mode in cooling-off period: remove before the cooling-off expires
- If compliance mode after cooling-off: the lock is PERMANENT — adjust backup plans to comply
- Ensure all backup plan lifecycle rules meet the vault lock minimum retention

SHOULD:
- Update backup plans to set DeleteAfterDays >= MinRetentionDays
- Document the vault lock configuration and its implications
- Warn users before applying compliance mode locks

MAY:
- Create separate vaults for different retention requirements
- Use governance mode for testing before applying compliance mode
- Implement approval workflows for compliance mode vault lock changes

## Common Issues

- symptoms: "Cannot delete recovery points — retention period not met"
  diagnosis: "Vault lock MinRetentionDays prevents deletion before the minimum retention."
  resolution: "Wait until the recovery point age exceeds MinRetentionDays. This cannot be overridden."

- symptoms: "Cannot remove vault lock"
  diagnosis: "The vault lock is in compliance mode and the cooling-off period has expired."
  resolution: "Compliance mode locks are PERMANENT. Adjust operations to work within the lock constraints."

- symptoms: "Backup job fails with lifecycle conflict"
  diagnosis: "The backup plan's DeleteAfterDays is less than the vault lock's MinRetentionDays."
  resolution: "Update the backup plan lifecycle to set DeleteAfterDays >= MinRetentionDays."

- symptoms: "Vault lock applied accidentally in compliance mode"
  diagnosis: "If still in cooling-off period, the lock can be removed. If expired, it is permanent."
  resolution: "Check LockDate and cooling-off period. Remove immediately if still in cooling-off."

## Output Format

```yaml
root_cause: "vault_lock — <specific_cause>"
evidence:
  - type: vault_lock
    content: "<lock mode, retention settings, lock date>"
  - type: cooling_off
    content: "<cooling-off period status>"
  - type: lifecycle_conflict
    content: "<conflicting backup plan lifecycle rules>"
severity: CRITICAL
mitigation:
  immediate: "Remove lock if governance mode or in cooling-off. Adjust plans if permanent."
  long_term: "Implement approval workflows for vault lock changes"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Remove governance mode lock | YELLOW | Removes retention enforcement — review implications |
| Remove compliance mode lock during cooling-off | RED | Time-sensitive — irreversible after cooling-off expires |
| Adjust backup plans to comply with permanent lock | YELLOW | Changes backup behavior to meet lock requirements |
| Ensure lifecycle rules meet MinRetentionDays | GREEN | Compliance verification |
| Create separate vaults for different retention | YELLOW | New infrastructure — review configuration |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- Compliance mode vault lock is PERMANENT after cooling-off — cannot be undone by anyone including root and AWS Support
- Vault lock changes affect retention enforcement for all recovery points

## Data Sensitivity

- Vault lock configuration reveals compliance and retention requirements
- Lock mode reveals regulatory obligations (compliance mode = regulatory requirement)
- Recovery point retention reveals data lifecycle policies

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible) — after cooling-off, this is impossible
- NEVER suggest reducing retention below compliance requirements
- NEVER apply compliance mode vault lock without explicit approval and understanding of permanence
- NEVER let compliance mode cooling-off expire without verification

## Phase 3 — Rollback

- If governance mode lock was removed: reapply if needed using `aws backup put-backup-vault-lock-configuration`
- If compliance mode lock is in cooling-off: remove immediately if applied in error — this is the ONLY window
- If compliance mode lock is permanent: NO ROLLBACK POSSIBLE — adjust all backup plans to comply
- If backup plans were updated: restore previous plan configuration if changes were incorrect
- Document all vault lock operations with timestamps for compliance audit trail

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
