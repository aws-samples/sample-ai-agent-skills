---
title: "C1 — PIN Operations"
description: "Diagnose PIN generation, verification, and translation failures"
status: active
severity: CRITICAL
triggers: ["PIN operation failed", "generate-pin-data error", "PIN verification", "PIN block", "translate-pin-data"]
owner: devops-agent
objective: "Resolve PIN operation failures in Payment Cryptography"
context: "PIN operations require specific key types (PEK, PVK). PIN blocks use industry-standard formats (ISO 0-4)."
---

## Phase 1 — Triage

MUST:
- Verify PIN key exists and is enabled: `aws payment-cryptography get-key --key-identifier <pin-key-arn>`
- Check key usage attributes match PIN operation (PEK for encryption, PVK for verification)
- Verify IAM permissions for payment-cryptography-data PIN operations

SHOULD:
- Check PIN block format matches expected standard (ISO 0, ISO 1, ISO 3, ISO 4)
- Verify PAN (Primary Account Number) format is correct

MAY:
- Review CloudTrail for PIN operation errors
- Check PIN verification method parameters (IBM 3624, Visa PVV)

## Phase 2 — Remediate

MUST:
- Use correct key type for PIN operation (PEK for generate/translate, PVK for verify)
- Fix PIN block format parameters
- Fix PAN format (must be valid card number format)

SHOULD:
- Implement PIN operation parameter validation
- Document PIN key assignments per card program

MAY:
- Set up PIN operation monitoring dashboards
- Implement PIN operation retry logic

## Safety Ratings

```
safety_ratings:
  - "get-key: GREEN — read-only key metadata"
  - "generate-pin-data: RED — generates PIN material, highly sensitive"
  - "verify-pin-data: YELLOW — verifies PIN without revealing it"
  - "translate-pin-data: RED — translates PIN blocks between keys"
```

## Escalation Conditions

- PIN operations fail in production affecting cardholder transactions
- PIN verification consistently fails for valid PINs
- PIN translation between systems produces incorrect results
- Key type mismatch requires new key creation and distribution

## Data Sensitivity

- PIN data: CRITICAL — cardholder PIN is highest sensitivity PCI DSS data
- PAN data: CRITICAL — Primary Account Number is cardholder data
- PIN blocks: CRITICAL — encrypted PIN material
- PIN verification values: HIGH — derived PIN verification data

## Prohibited Actions

- NEVER log PIN data, PIN blocks, or PAN in any form
- NEVER store clear-text PINs anywhere, even temporarily
- NEVER use test PINs with production card numbers
- NEVER bypass PIN verification for troubleshooting
- NEVER share PIN key material or PIN verification values

## Phase 3 — Rollback

1. If wrong PIN key was used: identify affected transactions and re-process
2. If PIN block format was changed: revert to standard format
3. If PIN verification parameters were modified: restore original settings
4. Securely destroy any PIN-related data generated during troubleshooting
5. Document all PIN operation changes for PCI DSS audit (dual control required)

## Common Issues

- symptoms: "GeneratePinData returns ValidationException"
  diagnosis: "Key type is not a PIN Encryption Key or PIN block format is invalid."
  resolution: "Use a key with TR31_P0_PIN_ENCRYPTION_KEY usage. Verify PIN block format (ISO_FORMAT_0, ISO_FORMAT_3, etc.)."

- symptoms: "VerifyPinData returns PIN verification failed"
  diagnosis: "Wrong PVK, wrong verification method, or wrong PIN offset."
  resolution: "Verify PVK matches the one used for PIN generation. Check verification method (IBM_3624, VISA_PVV) and parameters."

## Output Format

```yaml
root_cause: "pin_operation — <specific_cause>"
evidence:
  - type: pin_error
    content: "<error details — NO PIN DATA>"
severity: CRITICAL
mitigation:
  immediate: "Fix PIN operation parameters"
  long_term: "Implement PIN operation validation"
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
