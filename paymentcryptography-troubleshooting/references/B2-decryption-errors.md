---
title: "B2 — Decryption Errors"
description: "Diagnose payment data decryption failures"
status: active
severity: HIGH
triggers: ["decryption failed", "decrypt-data error", "cannot decrypt", "decryption operation"]
owner: devops-agent
objective: "Resolve payment data decryption operation failures"
context: "Decryption requires the same key and parameters used for encryption. Key must have decrypt usage attribute."
---

## Phase 1 — Triage

MUST:
- Verify key exists and is enabled: `aws payment-cryptography get-key --key-identifier <key-arn>`
- Check key usage attributes include decryption capability
- Verify the same key used for encryption is being used for decryption

SHOULD:
- Check decryption parameters match encryption parameters (mode, IV, padding)
- Verify cipher text format (hex-encoded)

MAY:
- Review CloudTrail for DecryptData errors
- Check if key was rotated between encryption and decryption

## Phase 2 — Remediate

MUST:
- Use the correct key that was used for encryption
- Fix decryption parameters to match encryption parameters
- Fix IAM permissions for payment-cryptography-data:DecryptData

SHOULD:
- Implement key tracking to associate encrypted data with keys
- Document encryption/decryption parameter standards

MAY:
- Implement key version tracking for rotated keys
- Set up decryption error monitoring

## Safety Ratings

```
safety_ratings:
  - "get-key: GREEN — read-only key metadata"
  - "decrypt-data: YELLOW — performs cryptographic operation, reveals plain text"
  - "IAM policy modification: YELLOW — changes access permissions"
  - "CloudTrail lookup: GREEN — read-only audit log review"
```

## Escalation Conditions

- Decryption failures affect production payment processing
- Encryption key has been deleted or disabled
- Decryption produces incorrect plain text (data corruption)
- Key rotation caused decryption failures for existing data

## Data Sensitivity

- Cipher text: HIGH — contains encrypted cardholder data
- Decrypted plain text: CRITICAL — contains cardholder data (PCI DSS scope)
- Key metadata: LOW — describes key capabilities
- Decryption parameters: LOW — algorithm configuration

## Prohibited Actions

- NEVER log decrypted plain text payment data
- NEVER attempt decryption with multiple keys as a brute-force approach
- NEVER store decrypted cardholder data outside PCI DSS compliant systems
- NEVER share decrypted data for troubleshooting purposes

## Phase 3 — Rollback

1. If IAM permissions were broadened: revert to least-privilege
2. If wrong key was identified: correct key reference in application
3. If decryption parameters were changed: revert to standard parameters
4. Securely delete any decrypted data generated during troubleshooting
5. Document all changes for PCI DSS audit trail

## Common Issues

- symptoms: "DecryptData returns cryptographic error"
  diagnosis: "Wrong key or wrong decryption parameters (mode, IV)."
  resolution: "Verify the exact key ARN used for encryption. Match decryption mode, IV, and padding to encryption parameters."

- symptoms: "DecryptData returns key not found"
  diagnosis: "Key has been deleted or is in a different region."
  resolution: "Check key status. If deleted, check if within waiting period and restore. If wrong region, use correct regional endpoint."

## Output Format

```yaml
root_cause: "decryption — <specific_cause>"
evidence:
  - type: decryption_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix decryption issue"
  long_term: "Implement key-data association tracking"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
