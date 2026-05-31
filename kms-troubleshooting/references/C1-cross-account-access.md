---
title: "C1 — Cross-Account Key Access"
description: "Diagnose cross-account KMS key access failures"
status: active
severity: HIGH
triggers:
  - "cross-account kms"
  - "another account key"
  - "external account access"
  - "cross-account decrypt"
owner: devops-agent
objective: "Configure cross-account KMS key access correctly"
context: "Cross-account KMS access requires BOTH the key policy in the key-owning account AND an IAM policy in the accessing account. The key policy must grant the external account (or specific principal), and the external account must have an IAM policy allowing the KMS action on the key ARN. Grants can also be used for cross-account access."
---

## Phase 1 — Triage

MUST:
- Get the key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Check if the key policy grants access to the external account or principal
- Check the IAM policy in the accessing account: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names <kms-action> --resource-arns <key-arn>`
- Verify the key ARN is correct (includes the key-owning account ID and region)

SHOULD:
- Check for grants that may provide cross-account access: `aws kms list-grants --key-id <key-id>`
- Verify there are no SCPs blocking KMS access in either account
- Check CloudTrail in both accounts for the denied event

MAY:
- Check for VPC endpoint policies that may restrict cross-account access
- Verify the key is not in a custom key store (which may have additional restrictions)

## Phase 2 — Remediate

MUST:
- Add the external account or principal to the key policy
- Add an IAM policy in the accessing account allowing the KMS action on the key ARN
- Both policies must be in place — one alone is insufficient

SHOULD:
- Use specific principal ARNs in the key policy instead of account-wide access
- Add conditions (kms:CallerAccount, kms:ViaService) for additional security
- Document cross-account key access requirements

MAY:
- Use grants for temporary cross-account access
- Create a replica key in the accessing account's region for multi-region keys

## Common Issues

- symptoms: "Key policy grants access to external account but access is still denied"
  diagnosis: "The IAM policy in the accessing account is missing or does not include the key ARN."
  resolution: "Add an IAM policy in the accessing account with the KMS action and key ARN."

- symptoms: "Cross-account access works for some operations but not others"
  diagnosis: "The key policy or IAM policy only grants specific actions, not all needed operations."
  resolution: "Add all required KMS actions (e.g., kms:Decrypt, kms:GenerateDataKey, kms:CreateGrant)."

- symptoms: "Cross-account access denied with kms:ViaService condition"
  diagnosis: "The key policy restricts cross-account access to requests through a specific service."
  resolution: "Ensure the request originates from the specified service, or modify the condition."

## Output Format

```yaml
root_cause: "cross_account_access_issue — <specific_cause>"
evidence:
  - type: key_policy
    content: "<key policy with cross-account statements>"
  - type: iam_policy
    content: "<accessing account IAM policy>"
  - type: grants
    content: "<cross-account grants>"
severity: HIGH
mitigation:
  immediate: "Configure both key policy and IAM policy for cross-account access"
  long_term: "Document cross-account access patterns and implement policy reviews"
```


## Safety Ratings

- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `list-grants`: GREEN — Read-only enumeration of grants
- `describe-key`: GREEN — Read-only key metadata retrieval
- `simulate-principal-policy`: GREEN — Read-only permission evaluation
- `put-key-policy`: RED — Cross-account key sharing exposes encryption keys to external principals
- `create-grant` (cross-account): RED — Grants external account access to encryption operations

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change (disable/enable/schedule deletion) is needed
- Multiple services depend on the affected key

## Rollback

1. Before any key policy change: Save current policy with `aws kms get-key-policy`
2. After change: Verify encryption/decryption operations work
3. If change causes issues: Restore the saved key policy immediately
4. Cleanup: Revoke any temporary grants created during investigation

## Data Sensitivity

| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-key-policy` | HIGH | Contains all authorized principals — redact |
| `list-grants` | HIGH | Shows who has access — summarize |
| `describe-key` | MEDIUM | Key metadata — safe to summarize |
| `list-keys` | LOW | Key IDs only — safe to include |

## Prohibited Actions

- NEVER suggest `"kms:*"` in a key policy
- NEVER suggest making a key policy public with `"Principal": "*"`
- NEVER suggest disabling key rotation as a fix
- NEVER suggest scheduling key deletion without confirming no resources depend on it
- NEVER suggest creating a new key to "work around" a policy issue
- ALWAYS identify the specific KMS action needed (kms:Decrypt, kms:GenerateDataKey, etc.)
- ALWAYS verify the encryption context matches before modifying policies

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
