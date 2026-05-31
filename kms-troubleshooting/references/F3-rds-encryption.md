---
title: "F3 — RDS Encryption Issues"
description: "Diagnose RDS instance and snapshot encryption with KMS problems"
status: active
severity: HIGH
triggers:
  - "RDS encryption"
  - "RDS KMS key"
  - "encrypted database"
  - "RDS snapshot encryption"
owner: devops-agent
objective: "Resolve RDS encryption failures and key access issues"
context: "RDS encryption uses KMS keys to encrypt the database, snapshots, read replicas, and automated backups. Encryption is set at creation and cannot be changed. RDS creates grants on the KMS key. Cross-region read replicas and snapshot copies require a KMS key in the destination region."
---

## Phase 1 — Triage

MUST:
- Check RDS instance encryption: `aws rds describe-db-instances --db-instance-identifier <db-id> --query 'DBInstances[0].{Encrypted:StorageEncrypted,KmsKeyId:KmsKeyId}'`
- Describe the KMS key: `aws kms describe-key --key-id <key-id>`
- Check key state: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.KeyState'`
- List grants for RDS: `aws kms list-grants --key-id <key-id>`

SHOULD:
- Check if the issue is with a snapshot or read replica
- Verify cross-region KMS key availability for replicas
- Check CloudTrail for KMS errors related to RDS

MAY:
- Check for cross-account snapshot sharing requirements
- Verify the RDS service role has KMS permissions

## Phase 2 — Remediate

MUST:
- Ensure the KMS key is enabled and accessible
- For cross-region operations, specify a KMS key in the destination region
- Verify grants exist for the RDS service

SHOULD:
- Use customer-managed keys for cross-account snapshot sharing
- Monitor KMS key state for keys used by RDS
- Plan for key rotation (RDS handles rotated keys transparently)

MAY:
- Encrypt an unencrypted database by creating an encrypted snapshot copy and restoring
- Use different KMS keys for different database environments

## Common Issues

- symptoms: "Cannot create encrypted read replica in another region"
  diagnosis: "No KMS key specified in the destination region."
  resolution: "Specify a KMS key ARN in the destination region when creating the cross-region replica."

- symptoms: "Cannot share encrypted snapshot cross-account"
  diagnosis: "Snapshot encrypted with AWS-managed key (aws/rds) cannot be shared."
  resolution: "Copy the snapshot with a customer-managed key, then share the copy."

- symptoms: "RDS instance start fails — KMS key unavailable"
  diagnosis: "The KMS key used for encryption is disabled or pending deletion."
  resolution: "Re-enable the key or cancel deletion. RDS cannot start without the encryption key."

## Output Format

```yaml
root_cause: "rds_encryption_issue — <specific_cause>"
evidence:
  - type: rds_instance
    content: "<RDS encryption configuration>"
  - type: key_state
    content: "<KMS key state>"
  - type: grants
    content: "<RDS-related grants>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key access for RDS operations"
  long_term: "Use customer-managed keys for cross-account/cross-region scenarios"
```


## Safety Ratings

- `describe-db-instances`: GREEN — Read-only retrieval of RDS encryption configuration
- `describe-key`: GREEN — Read-only key metadata retrieval
- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `list-grants`: GREEN — Read-only enumeration of RDS-related grants
- `put-key-policy` (RDS access): YELLOW — Affects RDS instance encryption/decryption operations
- `create-grant` (RDS service): YELLOW — Grants RDS service access to the key
- `enable-key-rotation`: YELLOW — Rotation is additive; RDS handles rotated keys transparently

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves granting cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change affects multiple services
- RDS instance serves production databases
- Cross-account RDS snapshot sharing with encrypted snapshots is involved

## Rollback

- Pre-change: "Save current key policy before modification"
- Verification: "Test encrypt/decrypt with the specific service after policy change"
- Revert: "Restore previous key policy if change breaks dependent services"

## Data Sensitivity

- HIGH: "Key policies reveal all authorized principals and services"
- HIGH: "Grant details expose cross-account relationships"
- MEDIUM: "Key metadata reveals encryption strategy"
- MEDIUM: "RDS instance details reveal KMS key ARNs used for database encryption"

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
