---
title: "B1 — Encryption Operation Failures"
description: "Diagnose payment data encryption failures"
status: active
severity: HIGH
triggers: ["encryption failed", "encrypt-data error", "cannot encrypt", "encryption operation"]
owner: devops-agent
objective: "Resolve payment data encryption operation failures"
context: "Encryption uses the data plane API (payment-cryptography-data). Key must have encrypt usage attribute."
---

## Phase 1 — Triage

MUST:
- Verify key exists and is enabled: `aws payment-cryptography get-key --key-identifier <key-arn>`
- Check key usage attributes include encryption capability
- Verify IAM permissions for payment-cryptography-data:EncryptData

SHOULD:
- Check encryption parameters (mode, IV, padding) match key algorithm
- Verify input data format (hex-encoded)

MAY:
- Review CloudTrail for EncryptData errors
- Check service quotas for data plane operations

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for payment-cryptography-data:EncryptData
- Use a key with correct usage attributes for encryption
- Fix encryption parameters to match key algorithm

SHOULD:
- Implement input validation before encryption calls
- Use consistent encryption parameters across systems

MAY:
- Set up CloudWatch alarms for encryption error rates
- Implement retry logic with exponential backoff

## Safety Ratings

```
safety_ratings:
  - "get-key: GREEN — read-only key metadata"
  - "encrypt-data: YELLOW — performs cryptographic operation on payment data"
  - "IAM policy modification: YELLOW — changes access permissions"
  - "CloudTrail lookup: GREEN — read-only audit log review"
```

## Escalation Conditions

- Encryption failures affect production payment processing
- Key usage attributes don't match required operations
- Encryption latency exceeds payment processing SLA
- Throttling errors during peak transaction volume

## Data Sensitivity

- Plain text payment data: CRITICAL — contains cardholder data (PCI DSS scope)
- Encrypted output: HIGH — contains encrypted cardholder data
- Key metadata: LOW — describes key capabilities
- Encryption parameters: LOW — algorithm configuration

## Prohibited Actions

- NEVER log plain text payment data during troubleshooting
- NEVER use test keys for production payment data
- NEVER bypass encryption for payment data processing
- NEVER share encryption parameters that could aid decryption

## Phase 3 — Rollback

1. If IAM permissions were broadened: revert to least-privilege after fix
2. If a wrong key was used for encryption: re-encrypt with correct key
3. If encryption parameters were changed: revert to standard parameters
4. Verify encrypted data can be decrypted correctly after changes
5. Document all changes for PCI DSS audit trail

## Common Issues

- symptoms: "EncryptData returns ValidationException"
  diagnosis: "Key usage attributes don't include encryption or input format is wrong."
  resolution: "Verify key has encrypt usage. Ensure input is hex-encoded and encryption attributes match key algorithm."

- symptoms: "EncryptData returns AccessDeniedException"
  diagnosis: "IAM policy missing payment-cryptography-data:EncryptData permission."
  resolution: "Add payment-cryptography-data:EncryptData to IAM policy. Note: this is the data plane, not control plane."

## Output Format

```yaml
root_cause: "encryption_operation — <specific_cause>"
evidence:
  - type: encryption_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix encryption issue"
  long_term: "Implement encryption monitoring"
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
