---
title: "H1 — Key Policy vs IAM Policy Conflicts"
description: "Diagnose conflicts between KMS key policies and IAM policies"
status: active
severity: MEDIUM
triggers:
  - "key policy vs IAM"
  - "policy conflict"
  - "which policy applies"
  - "policy evaluation order"
owner: devops-agent
objective: "Understand and resolve conflicts between key policies and IAM policies"
context: "KMS has a unique authorization model. The key policy is always evaluated. IAM policies only apply if the key policy delegates to IAM (via root account statement). Explicit denies in either policy override allows. Understanding this interaction is critical for troubleshooting access issues."
---

## Phase 1 — Triage

MUST:
- Get the key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Check if the root account statement exists (enables IAM policies)
- Get the caller's IAM policies: `aws iam list-attached-role-policies --role-name <role-name>`
- Simulate the combined effect: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names <kms-action> --resource-arns <key-arn>`

SHOULD:
- Check for explicit deny statements in both key policy and IAM policies
- Check for SCPs that may affect KMS access
- Identify which policy is granting or denying access

MAY:
- Check for permission boundaries on the IAM principal
- Review VPC endpoint policies for KMS

## Phase 2 — Remediate

MUST:
- Ensure the key policy and IAM policy work together (not against each other)
- Remove conflicting explicit deny statements
- Verify the root account statement is present if relying on IAM policies

SHOULD:
- Use key policy for key-specific access control
- Use IAM policies for principal-specific access control
- Document the authorization model for each key

MAY:
- Simplify by using only key policy (without IAM delegation) for sensitive keys
- Use grants for temporary access instead of modifying policies

## Common Issues

- symptoms: "IAM policy grants kms:Decrypt but access is denied"
  diagnosis: "Key policy does not include the root account statement, so IAM policies are ignored."
  resolution: "Add the root account statement to the key policy, or add the principal directly to the key policy."

- symptoms: "Key policy allows access but IAM policy has explicit deny"
  diagnosis: "Explicit deny in IAM policy overrides the key policy allow."
  resolution: "Remove the explicit deny from the IAM policy, or use the key policy to grant access directly."

## Output Format

```yaml
root_cause: "policy_conflict — <specific_cause>"
evidence:
  - type: key_policy
    content: "<key policy document>"
  - type: iam_policy
    content: "<IAM policy document>"
  - type: simulation
    content: "<policy simulation result>"
severity: MEDIUM
mitigation:
  immediate: "Resolve the policy conflict"
  long_term: "Document the authorization model and review policies regularly"
```


## Safety Ratings

- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `list-attached-role-policies`: GREEN — Read-only IAM policy enumeration
- `simulate-principal-policy`: GREEN — Read-only permission evaluation
- `describe-key`: GREEN — Read-only key metadata retrieval
- `put-key-policy`: RED — Key policy is the primary access control mechanism; incorrect changes can lock out all principals
- `put-key-policy` (root account statement): RED — Enables or disables IAM policy evaluation for the key

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves granting cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change affects multiple services
- Root account statement is being added or removed
- Key policy change could affect IAM policy evaluation for multiple principals

## Rollback

- Pre-change: "Save current key policy before modification"
- Verification: "Test encrypt/decrypt with the specific service after policy change"
- Revert: "Restore previous key policy if change breaks dependent services"

## Data Sensitivity

- HIGH: "Key policies reveal all authorized principals and services"
- HIGH: "Grant details expose cross-account relationships"
- HIGH: "Policy analysis reveals the complete access model for the key"
- MEDIUM: "Key metadata reveals encryption strategy"

## Prohibited Actions

- NEVER suggest `"kms:*"` in a key policy
- NEVER suggest making a key policy public with `"Principal": "*"`
- NEVER suggest disabling key rotation as a fix
- NEVER suggest scheduling key deletion without confirming no active usage
- NEVER suggest creating a new key to bypass policy issues (data becomes inaccessible)

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
  - command: "get-key-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-key"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-grants"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest kms:* on key policies"
  - "NEVER suggest disabling key rotation"
  - "NEVER suggest making key policies public"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
