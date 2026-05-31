---
title: "C2 — MAC Verification"
description: "Diagnose MAC generation and verification failures"
status: active
severity: HIGH
triggers: ["MAC verification failed", "verify-mac error", "generate-mac error", "MAC mismatch"]
owner: devops-agent
objective: "Resolve MAC generation and verification failures in Payment Cryptography"
context: "MAC operations require keys with MAC usage attributes (HMAC or CMAC). Encryption keys cannot be used for MAC."
---

## Phase 1 — Triage

MUST:
- Verify MAC key exists and is enabled: `aws payment-cryptography get-key --key-identifier <mac-key-arn>`
- Check key usage attributes include MAC generation/verification
- Verify IAM permissions for payment-cryptography-data MAC operations

SHOULD:
- Check MAC algorithm matches between generation and verification (HMAC_SHA256, CMAC, etc.)
- Verify message data is identical between generation and verification

MAY:
- Review CloudTrail for MAC operation errors
- Check if MAC key was rotated between generation and verification

## Phase 2 — Remediate

MUST:
- Use a key with MAC usage attributes (not encryption key)
- Ensure MAC algorithm matches between generation and verification
- Verify message data integrity (no modification between MAC generation and verification)

SHOULD:
- Implement MAC key tracking per transaction type
- Document MAC algorithm standards per payment network

MAY:
- Set up MAC verification failure monitoring
- Implement MAC operation retry logic

## Safety Ratings

```
safety_ratings:
  - "get-key: GREEN — read-only key metadata"
  - "generate-mac: YELLOW — generates MAC for message authentication"
  - "verify-mac: GREEN — verifies MAC without revealing key material"
  - "IAM policy modification: YELLOW — changes access permissions"
```

## Escalation Conditions

- MAC verification failures affect production transaction processing
- MAC key mismatch between sending and receiving systems
- MAC algorithm incompatibility between payment networks
- High rate of MAC verification failures indicating potential tampering

## Data Sensitivity

- Message data: HIGH — may contain transaction details
- MAC values: MEDIUM — authentication codes
- MAC key metadata: LOW — describes key capabilities
- MAC algorithm parameters: LOW — configuration data

## Prohibited Actions

- NEVER ignore MAC verification failures (may indicate message tampering)
- NEVER use encryption keys for MAC operations
- NEVER share MAC keys between unrelated systems
- NEVER disable MAC verification for troubleshooting

## Phase 3 — Rollback

1. If MAC key was changed: revert to previous key and notify counterparties
2. If MAC algorithm was modified: restore original algorithm setting
3. If IAM permissions were broadened: revert to least-privilege
4. Verify MAC operations work correctly after changes
5. Document all changes for PCI DSS audit trail

## Common Issues

- symptoms: "VerifyMac returns MAC verification failed"
  diagnosis: "Wrong MAC key, wrong algorithm, or message data was modified."
  resolution: "Verify same key and algorithm used for generation and verification. Check message data integrity."

- symptoms: "GenerateMac returns ValidationException"
  diagnosis: "Key does not have MAC usage attributes."
  resolution: "Use a key with TR31_M1_ISO_9797_1_MAC_KEY or TR31_M3_ISO_9797_3_MAC_KEY usage."

## Output Format

```yaml
root_cause: "mac_verification — <specific_cause>"
evidence:
  - type: mac_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix MAC operation parameters"
  long_term: "Implement MAC key management standards"
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
