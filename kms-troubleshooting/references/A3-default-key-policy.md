---
title: "A3 — Default Key Policy Issues"
description: "Diagnose issues with the default key policy and root account statement"
status: active
severity: HIGH
triggers:
  - "default key policy"
  - "root account access"
  - "locked out of key"
  - "cannot manage key"
  - "key policy reset"
owner: devops-agent
objective: "Restore proper default key policy configuration and prevent lockout"
context: "The default key policy includes a root account statement that enables IAM policies to work. Removing this statement locks out all IAM-based access. Only principals explicitly named in the key policy can access the key. If all named principals are deleted, the key becomes unmanageable — AWS Support must be contacted."
---

## Phase 1 — Triage

MUST:
- Get the current key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Check if the root account statement exists: look for `"arn:aws:iam::<account-id>:root"` with `"kms:*"`
- Describe the key to check key manager: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.KeyManager'`
- Verify the key is customer-managed (not AWS-managed)

SHOULD:
- List all principals in the key policy
- Check if any named principals still exist in IAM
- Review CloudTrail for recent key policy changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=PutKeyPolicy --max-results 10`

MAY:
- Check if the key was created via CloudFormation or Terraform (policy may be in IaC)

## Phase 2 — Remediate

MUST:
- If root statement is missing, add it back using a principal that still has access
- If completely locked out, contact AWS Support to reset the key policy
- Verify IAM policies work after restoring the root statement

SHOULD:
- Always include the root account statement in key policies
- Add at least two key administrators to prevent single-point-of-failure lockout
- Use IaC (CloudFormation/Terraform) to manage key policies for version control

MAY:
- Create a new key with proper policy and re-encrypt data if the old key is unrecoverable
- Set up CloudWatch Events to alert on key policy changes

## Common Issues

- symptoms: "All IAM users/roles lost access to the key"
  diagnosis: "The root account statement was removed from the key policy."
  resolution: "Use a principal still named in the key policy to add back the root statement."

- symptoms: "Cannot modify the key policy — access denied"
  diagnosis: "The caller is not a key administrator in the key policy and IAM policies are not enabled."
  resolution: "Use another key administrator or contact AWS Support if completely locked out."

- symptoms: "AWS-managed key policy cannot be viewed or modified"
  diagnosis: "The key is AWS-managed (aws/* alias). These policies cannot be modified."
  resolution: "Create a customer-managed key with the desired policy instead."

## Output Format

```yaml
root_cause: "default_key_policy_issue — <specific_cause>"
evidence:
  - type: key_policy
    content: "<current key policy>"
  - type: key_metadata
    content: "<key manager and state>"
severity: HIGH
mitigation:
  immediate: "Restore the root account statement or contact AWS Support"
  long_term: "Always maintain the root statement and multiple key administrators"
```


## Safety Ratings

- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `describe-key`: GREEN — Read-only key metadata retrieval
- `lookup-events` (CloudTrail): GREEN — Read-only audit log analysis
- `put-key-policy`: RED — Default key policy controls all access; incorrect changes can permanently lock out all principals

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
