---
title: "E1 — Encryption Context Issues"
description: "Diagnose encryption context mismatch and configuration problems"
status: active
severity: HIGH
triggers:
  - "encryption context"
  - "InvalidCiphertextException"
  - "context mismatch"
  - "decrypt failed"
owner: devops-agent
objective: "Resolve encryption context mismatches and configure context correctly"
context: "Encryption context is a set of key-value pairs cryptographically bound to ciphertext. The same context must be provided during decryption. It is logged in CloudTrail and can be used in policy conditions. AWS services like S3 and EBS use encryption context automatically."
---

## Phase 1 — Triage

MUST:
- Check CloudTrail for the encrypt event to find the original encryption context: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=Encrypt --max-results 10`
- Compare the encryption context used during encryption vs decryption
- Check for InvalidCiphertextException in the error response
- Verify the application is passing the correct encryption context

SHOULD:
- Check if AWS services are adding their own encryption context (S3 adds bucket/key, EBS adds volume ID)
- Review key policy for encryption context conditions: `aws kms get-key-policy --key-id <key-id> --policy-name default`

MAY:
- Check grants for encryption context constraints: `aws kms list-grants --key-id <key-id>`

## Phase 2 — Remediate

MUST:
- Ensure the exact same encryption context is used for both encrypt and decrypt
- Fix the application code to pass the correct context
- If the original context is unknown, check CloudTrail for the encrypt event

SHOULD:
- Document the encryption context format for each use case
- Use consistent encryption context key-value pairs across the application
- Leverage encryption context in key policies for access control

MAY:
- Implement encryption context validation in the application layer
- Use encryption context for audit trail purposes

## Common Issues

- symptoms: "InvalidCiphertextException when decrypting"
  diagnosis: "Encryption context provided during decryption does not match what was used during encryption."
  resolution: "Check CloudTrail for the original encryption context. Provide the exact same key-value pairs."

- symptoms: "S3 SSE-KMS decrypt fails with context error"
  diagnosis: "S3 automatically adds encryption context (aws:s3:arn). Custom context may conflict."
  resolution: "S3 manages encryption context automatically. Do not override it for S3 operations."

## Output Format

```yaml
root_cause: "encryption_context_issue — <specific_cause>"
evidence:
  - type: cloudtrail
    content: "<encrypt event with original context>"
  - type: error
    content: "<InvalidCiphertextException details>"
severity: HIGH
mitigation:
  immediate: "Fix encryption context to match the original"
  long_term: "Document and standardize encryption context usage"
```


## Safety Ratings

- `lookup-events` (CloudTrail): GREEN — Read-only audit log analysis of encryption context usage
- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `describe-key`: GREEN — Read-only key metadata retrieval
- `list-grants`: GREEN — Read-only enumeration of grants with encryption context constraints
- `put-key-policy` (encryption context conditions): YELLOW — Adding encryption context conditions to key policy affects access control

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves granting cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change affects multiple services
- Encryption context mismatch is causing production decryption failures

## Rollback

- Pre-change: "Save current key policy before modification"
- Verification: "Test encrypt/decrypt with the specific service after policy change"
- Revert: "Restore previous key policy if change breaks dependent services"

## Data Sensitivity

- HIGH: "Key policies reveal all authorized principals and services"
- HIGH: "Grant details expose cross-account relationships"
- HIGH: "Encryption context values are logged in plaintext in CloudTrail — ensure they do not contain sensitive data"
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
