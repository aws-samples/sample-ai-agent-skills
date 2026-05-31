---
title: "F2 — EBS Encryption Issues"
description: "Diagnose EBS volume encryption with KMS problems"
status: active
severity: HIGH
triggers:
  - "EBS encryption"
  - "volume encryption"
  - "EBS KMS key"
  - "encrypted volume"
  - "snapshot encryption"
owner: devops-agent
objective: "Resolve EBS encryption failures and key access issues"
context: "EBS encryption uses KMS keys to encrypt volumes and snapshots. EBS creates grants on the KMS key for each volume attachment. The key must be accessible in the volume's region. Cross-region snapshot copy requires a KMS key in the destination region. EBS volumes cannot be re-encrypted with a different key."
---

## Phase 1 — Triage

MUST:
- Check volume encryption: `aws ec2 describe-volumes --volume-ids <vol-id> --query 'Volumes[0].{Encrypted:Encrypted,KmsKeyId:KmsKeyId}'`
- Describe the KMS key: `aws kms describe-key --key-id <key-id>`
- Check key state: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.KeyState'`
- List grants for EBS: `aws kms list-grants --key-id <key-id> --grantee-principal <instance-role-arn>`

SHOULD:
- Check default EBS encryption settings: `aws ec2 get-ebs-encryption-by-default`
- Check default KMS key for EBS: `aws ec2 get-ebs-default-kms-key-id`
- Verify the instance role has kms:CreateGrant permission

MAY:
- Check for cross-region snapshot copy issues
- Verify the key is not in PendingDeletion state

## Phase 2 — Remediate

MUST:
- Ensure the KMS key is enabled and accessible
- Grant kms:CreateGrant to the principal attaching the volume
- For cross-region copy, specify a KMS key in the destination region

SHOULD:
- Enable default EBS encryption for the account/region
- Use customer-managed keys for cross-account volume sharing
- Monitor KMS key state for keys used by EBS

MAY:
- Create encrypted AMIs for consistent encryption across instances
- Implement key rotation (EBS handles rotated keys transparently)

## Common Issues

- symptoms: "Cannot attach encrypted volume — KMS key error"
  diagnosis: "KMS key is disabled, pending deletion, or the instance role lacks kms:CreateGrant."
  resolution: "Enable the key and grant kms:CreateGrant to the instance role."

- symptoms: "Cross-region snapshot copy fails with KMS error"
  diagnosis: "No KMS key specified in the destination region, or the key is inaccessible."
  resolution: "Specify a KMS key ARN in the destination region when copying the snapshot."

- symptoms: "Cannot share encrypted snapshot cross-account"
  diagnosis: "AWS-managed key (aws/ebs) cannot be shared. Must use customer-managed key."
  resolution: "Re-encrypt the snapshot with a customer-managed key, then share."

## Output Format

```yaml
root_cause: "ebs_encryption_issue — <specific_cause>"
evidence:
  - type: volume
    content: "<volume encryption details>"
  - type: key_state
    content: "<KMS key state>"
  - type: grants
    content: "<EBS-related grants>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key access for EBS operations"
  long_term: "Use customer-managed keys and enable default encryption"
```


## Safety Ratings

- `describe-volumes`: GREEN — Read-only retrieval of EBS volume encryption details
- `describe-key`: GREEN — Read-only key metadata retrieval
- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `list-grants`: GREEN — Read-only enumeration of EBS-related grants
- `get-ebs-encryption-by-default`: GREEN — Read-only check of default encryption settings
- `put-key-policy` (EBS access): YELLOW — Affects EBS volume encryption/decryption operations
- `modify-ebs-default-kms-key-id`: YELLOW — Changes default encryption key for all new EBS volumes in the account/region
- `enable-key-rotation`: YELLOW — Rotation is additive; EBS handles rotated keys transparently

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves granting cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change affects multiple services
- EBS volumes attached to production instances are affected
- Default EBS encryption key change impacts multiple workloads

## Rollback

- Pre-change: "Save current key policy before modification"
- Verification: "Test encrypt/decrypt with the specific service after policy change"
- Revert: "Restore previous key policy if change breaks dependent services"

## Data Sensitivity

- HIGH: "Key policies reveal all authorized principals and services"
- HIGH: "Grant details expose cross-account relationships"
- MEDIUM: "Key metadata reveals encryption strategy"
- MEDIUM: "Volume encryption details reveal KMS key ARNs used for EBS encryption"

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
