---
title: "E2 — Asymmetric Key Issues"
description: "Diagnose asymmetric KMS key operation and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "asymmetric key"
  - "RSA key"
  - "ECC key"
  - "sign verify"
  - "public key"
owner: devops-agent
objective: "Resolve asymmetric key operation failures and configuration issues"
context: "Asymmetric KMS keys have a public/private key pair. The private key never leaves KMS. The public key can be downloaded. Key usage is either ENCRYPT_DECRYPT or SIGN_VERIFY (not both). Asymmetric keys have lower request rate limits and do not support automatic rotation."
---

## Phase 1 — Triage

MUST:
- Describe the key: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.{Spec:KeySpec,Usage:KeyUsage,State:KeyState}'`
- Verify the key usage matches the operation (ENCRYPT_DECRYPT vs SIGN_VERIFY)
- Check the key spec is compatible with the algorithm being used
- Download the public key if needed: `aws kms get-public-key --key-id <key-id>`

SHOULD:
- Check for algorithm compatibility (RSA_OAEP for encryption, RSASSA_PKCS1_V1_5 or ECDSA for signing)
- Verify the caller has the correct KMS action (kms:Encrypt vs kms:Sign)

MAY:
- Test with a simple sign/verify or encrypt/decrypt operation
- Check request rate limits for asymmetric operations

## Phase 2 — Remediate

MUST:
- Use the correct algorithm for the key spec and usage
- Ensure IAM/key policy grants the correct action (kms:Sign, kms:Verify, kms:Encrypt, kms:Decrypt)
- Use the public key for external encryption/verification operations

SHOULD:
- Document the key spec, usage, and supported algorithms
- Plan manual rotation since automatic rotation is not supported

MAY:
- Use envelope encryption with symmetric keys for bulk data instead of asymmetric encryption
- Cache the public key to reduce KMS API calls

## Common Issues

- symptoms: "InvalidKeyUsageException — key usage is SIGN_VERIFY"
  diagnosis: "Attempting to encrypt/decrypt with a key created for signing."
  resolution: "Use a key with ENCRYPT_DECRYPT usage for encryption operations."

- symptoms: "Cannot enable automatic rotation on asymmetric key"
  diagnosis: "Automatic rotation is only supported for symmetric encryption keys."
  resolution: "Manually rotate by creating a new asymmetric key and updating references."

## Output Format

```yaml
root_cause: "asymmetric_key_issue — <specific_cause>"
evidence:
  - type: key_metadata
    content: "<key spec, usage, and state>"
  - type: operation
    content: "<attempted operation and algorithm>"
severity: MEDIUM
mitigation:
  immediate: "Use correct algorithm and key usage combination"
  long_term: "Document key capabilities and plan manual rotation"
```


## Safety Ratings

- `describe-key`: GREEN — Read-only key metadata retrieval (key spec, usage, state)
- `get-public-key`: GREEN — Public key is non-sensitive by design
- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `list-grants`: GREEN — Read-only enumeration of grants
- `put-key-policy` (asymmetric key): YELLOW — Policy changes affect signing or encryption operations
- `create-grant` (asymmetric key): YELLOW — Grants provide access to sign/verify or encrypt/decrypt operations

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves granting cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change affects multiple services
- Key is used for digital signing (code signing, document signing)
- Key spec change is needed (requires creating a new key)

## Rollback

- Pre-change: "Save current key policy before modification"
- Verification: "Test encrypt/decrypt with the specific service after policy change"
- Revert: "Restore previous key policy if change breaks dependent services"

## Data Sensitivity

- HIGH: "Key policies reveal all authorized principals and services"
- HIGH: "Grant details expose cross-account relationships"
- MEDIUM: "Key metadata reveals encryption strategy"
- LOW: "Public key is non-sensitive by design"

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
