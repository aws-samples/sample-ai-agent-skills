---
title: "A1 — KMS Access Denied"
description: "Diagnose AccessDeniedException on KMS operations"
status: active
severity: HIGH
triggers:
  - "AccessDeniedException"
  - "access denied kms"
  - "not authorized to perform kms"
  - "kms permission denied"
owner: devops-agent
objective: "Identify why a principal is denied access to a KMS key and restore access"
context: "KMS access denied errors can stem from key policy, IAM policy, grants, SCPs, VPC endpoint policies, or key state. The key policy is the primary access control — IAM policies only work if the key policy allows it. Always check the key policy first."
---

## Phase 1 — Triage

MUST:
- Describe the key: `aws kms describe-key --key-id <key-id>`
- Get the key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Check if the key policy allows the account root (enables IAM policies): look for `"Principal": {"AWS": "arn:aws:iam::<account-id>:root"}`
- Check the caller's IAM policies: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names <kms-action> --resource-arns <key-arn>`
- Check key state (must be Enabled): `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.KeyState'`

SHOULD:
- List grants on the key: `aws kms list-grants --key-id <key-id>`
- Check for SCPs that may deny KMS actions: review Organizations SCPs
- Check CloudTrail for the specific denied event: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<kms-action> --max-results 10`
- Check for VPC endpoint policy restrictions if using a VPC endpoint

MAY:
- Check for condition keys in the key policy (kms:ViaService, kms:CallerAccount, aws:SourceVpc)
- Verify the principal ARN matches what's in the key policy (exact match required)

## Phase 2 — Remediate

MUST:
- If key policy lacks root account statement, add it to enable IAM policies
- If principal needs direct access, add them to the key policy
- If key is disabled, enable it: `aws kms enable-key --key-id <key-id>`

SHOULD:
- Use grants for temporary or service-based access instead of modifying key policy
- Follow least-privilege principle — grant only the specific KMS actions needed
- Document the key policy and access requirements

MAY:
- Create a grant for the specific principal and operations needed
- Use encryption context conditions for fine-grained access control

## Common Issues

- symptoms: "AccessDeniedException despite IAM policy allowing kms:Decrypt"
  diagnosis: "Key policy does not include the account root statement, so IAM policies are ignored."
  resolution: "Add the account root principal statement to the key policy."

- symptoms: "Access denied for cross-account principal"
  diagnosis: "Key policy does not grant access to the external account, or the external account's IAM policy is missing."
  resolution: "Both the key policy AND the external account's IAM policy must grant access."

- symptoms: "Access denied with kms:ViaService condition"
  diagnosis: "The key policy restricts access to requests made through a specific AWS service."
  resolution: "Ensure the request comes through the specified service, or modify the condition."

## Output Format

```yaml
root_cause: "kms_access_denied — <specific_cause>"
evidence:
  - type: key_policy
    content: "<key policy document>"
  - type: iam_simulation
    content: "<simulate-principal-policy result>"
  - type: key_state
    content: "<key state and metadata>"
severity: HIGH
mitigation:
  immediate: "Update key policy or IAM policy to grant required access"
  long_term: "Document key access requirements and implement policy reviews"
```


## Safety Ratings

- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `describe-key`: GREEN — Read-only key metadata retrieval
- `list-grants`: GREEN — Read-only enumeration of grants
- `simulate-principal-policy`: GREEN — Read-only analysis of existing permissions
- `put-key-policy`: RED — Key policy is the primary access control; incorrect changes can lock out all principals
- `enable-key`: YELLOW — Re-enables encryption/decryption for all principals with access

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
