---
title: "A2 — Key Policy Evaluation Issues"
description: "Diagnose complex key policy evaluation and authorization logic"
status: active
severity: MEDIUM
triggers:
  - "key policy not working"
  - "policy evaluation"
  - "policy conflict"
  - "unexpected deny"
  - "condition key"
owner: devops-agent
objective: "Understand and fix key policy evaluation issues"
context: "KMS key policy evaluation follows a unique model: the key policy is always checked first. If the key policy grants access, the request is allowed (no IAM policy needed). If the key policy delegates to IAM (via root account statement), then IAM policies are also evaluated. Conditions, SCPs, and VPC endpoint policies add complexity."
---

## Phase 1 — Triage

MUST:
- Get the full key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Identify all principals in the key policy and their allowed actions
- Check for condition keys (kms:ViaService, kms:EncryptionContext, kms:CallerAccount, aws:SourceVpc)
- Simulate the policy: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names <kms-action> --resource-arns <key-arn>`

SHOULD:
- Check for explicit deny statements in the key policy
- Verify condition values match the actual request context
- Check IAM policies attached to the principal: `aws iam list-attached-role-policies --role-name <role-name>`

MAY:
- Check SCPs in Organizations: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`
- Check VPC endpoint policy if applicable

## Phase 2 — Remediate

MUST:
- Fix condition key values to match the actual request context
- Remove or correct explicit deny statements that block legitimate access
- Ensure the key policy and IAM policy work together correctly

SHOULD:
- Use `aws iam simulate-principal-policy` with context entries to test conditions
- Document the policy evaluation logic for the key
- Test policy changes in a non-production environment first

MAY:
- Simplify the key policy by reducing condition complexity
- Use separate keys for different access patterns instead of complex conditions

## Common Issues

- symptoms: "Key policy grants access but request is still denied"
  diagnosis: "A condition key in the policy does not match the request context."
  resolution: "Check all condition keys. Use CloudTrail to see the actual request context."

- symptoms: "SCP is blocking KMS access"
  diagnosis: "An Organization SCP denies the KMS action for the account."
  resolution: "Review SCPs. The management account is exempt from SCPs."

- symptoms: "Policy works for some operations but not others"
  diagnosis: "The key policy grants specific actions, not kms:*."
  resolution: "Add the missing KMS actions to the key policy statement."

## Output Format

```yaml
root_cause: "policy_evaluation_issue — <specific_cause>"
evidence:
  - type: key_policy
    content: "<full key policy with conditions>"
  - type: simulation
    content: "<policy simulation results>"
  - type: conditions
    content: "<condition key analysis>"
severity: MEDIUM
mitigation:
  immediate: "Fix the policy evaluation issue"
  long_term: "Document and test key policy logic regularly"
```


## Safety Ratings

- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `simulate-principal-policy`: GREEN — Read-only permission evaluation
- `list-attached-role-policies`: GREEN — Read-only IAM policy enumeration
- `lookup-events` (CloudTrail): GREEN — Read-only audit log analysis
- `put-key-policy`: RED — Modifying key policy affects all authorization decisions for the key

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
