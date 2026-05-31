---
title: "D3 — Key Rotation Issues"
description: "Diagnose automatic and manual key rotation configuration problems"
status: active
severity: MEDIUM
triggers:
  - "key rotation"
  - "rotate key"
  - "rotation not working"
  - "rotation period"
owner: devops-agent
objective: "Configure key rotation correctly for compliance and security"
context: "Automatic rotation is only supported for symmetric encryption keys with AWS_KMS origin. It creates new key material while keeping the old material for decryption. Asymmetric keys, imported keys, and custom key store keys must be manually rotated by creating a new key."
---

## Phase 1 — Triage

MUST:
- Check rotation status: `aws kms get-key-rotation-status --key-id <key-id>`
- Describe the key to check spec and origin: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.{Spec:KeySpec,Origin:Origin,Usage:KeyUsage}'`
- Verify the key supports automatic rotation (symmetric, AWS_KMS origin)

SHOULD:
- Check rotation history: `aws kms list-key-rotations --key-id <key-id>`
- Review CloudTrail for rotation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RotateKey --max-results 10`

MAY:
- Check compliance requirements for rotation frequency

## Phase 2 — Remediate

MUST:
- Enable automatic rotation for eligible keys: `aws kms enable-key-rotation --key-id <key-id>`
- For ineligible keys (asymmetric, imported), plan manual rotation
- Verify rotation is enabled after configuration

SHOULD:
- Set rotation period if non-default is needed: `aws kms enable-key-rotation --key-id <key-id> --rotation-period-in-days <days>`
- Document the rotation strategy for each key type
- Test that applications handle rotated keys correctly (old ciphertext still decryptable)

MAY:
- Use aliases to simplify manual key rotation (point alias to new key)
- Implement automated manual rotation for imported key material

## Common Issues

- symptoms: "Cannot enable rotation — key type not supported"
  diagnosis: "Automatic rotation only works for symmetric encryption keys with AWS_KMS origin."
  resolution: "For asymmetric or imported keys, manually rotate by creating a new key and updating references."

- symptoms: "Rotation enabled but key material hasn't changed"
  diagnosis: "Rotation happens on the anniversary of enablement (or configured period)."
  resolution: "Use `rotate-key-on-demand` for immediate rotation, or wait for the scheduled rotation."

- symptoms: "Old ciphertext cannot be decrypted after rotation"
  diagnosis: "This should not happen — KMS retains old key material. Check if the key was replaced instead of rotated."
  resolution: "Verify the key ID is the same. Rotation preserves old material. Key replacement does not."

## Output Format

```yaml
root_cause: "key_rotation_issue — <specific_cause>"
evidence:
  - type: rotation_status
    content: "<rotation enabled/disabled and period>"
  - type: key_metadata
    content: "<key spec, origin, and usage>"
severity: MEDIUM
mitigation:
  immediate: "Enable rotation or plan manual rotation"
  long_term: "Document rotation strategy per key type and automate where possible"
```


## Safety Ratings

- `get-key-rotation-status`: GREEN — Read-only rotation status check
- `describe-key`: GREEN — Read-only key metadata retrieval
- `list-key-rotations`: GREEN — Read-only rotation history
- `enable-key-rotation`: YELLOW — Rotation is additive; old key material is retained for decryption
- `rotate-key-on-demand`: YELLOW — Creates new key material but retains old material

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
