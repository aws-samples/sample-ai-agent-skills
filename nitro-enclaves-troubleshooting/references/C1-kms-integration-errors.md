---
title: "C1 — KMS Integration Errors"
description: "Diagnose KMS decrypt/encrypt failures with Nitro Enclave attestation"
status: active
severity: CRITICAL
triggers: ["KMS decrypt failed", "KMS attestation error", "enclave KMS access denied", "key policy error"]
owner: devops-agent
objective: "Resolve KMS integration issues for enclave-based cryptographic operations"
context: "Enclaves use KMS with attestation-based key policies. The enclave sends an attestation document with KMS requests via a vsock proxy. KMS verifies PCR values in the attestation before allowing operations. PCR values change when the enclave image changes."
---

## Phase 1 — Triage

MUST:
- Check KMS key exists and is enabled: `aws kms describe-key --key-id <key-id> --query '{KeyState:KeyMetadata.KeyState,Enabled:KeyMetadata.Enabled}'`
- Review KMS key policy for attestation conditions: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Get current enclave PCR values: `nitro-cli describe-eif --eif-path <path>`

SHOULD:
- Verify vsock proxy (kmstool-enclave-cli or custom proxy) is running on parent
- Check enclave console for KMS-related errors: `nitro-cli console --enclave-id <id>`
- Verify IAM role on parent instance has kms:Decrypt/kms:Encrypt permissions

MAY:
- Check CloudTrail for KMS API failures: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com --max-results 20`
- Verify KMS endpoint is reachable from parent instance
- Test KMS access from parent (without attestation) to isolate enclave-specific issues

## Phase 2 — Remediate

MUST:
- Update KMS key policy with correct PCR values from the current EIF
- Ensure key policy condition uses correct format: `kms:RecipientAttestation:PCR0` for PCR0
- Verify vsock proxy forwards KMS requests from enclave to AWS KMS via parent

SHOULD:
- Include multiple PCR conditions (PCR0, PCR1, PCR2) for stronger attestation
- Ensure parent instance IAM role has kms:Decrypt and kms:Encrypt permissions
- Test KMS integration after any enclave image rebuild (PCR values change)

MAY:
- Implement PCR value tracking in CI/CD pipeline for automatic key policy updates
- Use KMS key aliases for easier key management
- Set up CloudTrail alerts for KMS access denied events

## Safety Ratings

```
safety_ratings:
  - "kms describe-key: GREEN — read-only key inspection"
  - "kms get-key-policy: GREEN — read-only policy inspection"
  - "describe-eif: GREEN — read-only PCR value retrieval"
  - "kms put-key-policy: RED — modifies key access controls"
  - "kms decrypt: YELLOW — decrypts data, verify authorization"
```

## Escalation Conditions

- KMS key policy locked out all principals (including admin)
- PCR values changed unexpectedly indicating potential image tampering
- KMS key is in PendingDeletion state
- Cross-account KMS access required for enclave attestation

## Data Sensitivity

- `aws kms describe-key`: LOW — returns key metadata
- `aws kms get-key-policy`: HIGH — reveals access control policy
- `nitro-cli describe-eif`: MEDIUM — reveals PCR values (enclave identity)
- KMS decrypt output: CRITICAL — contains decrypted sensitive data

## Prohibited Actions

- NEVER remove attestation conditions from KMS key policy to "fix" access — this defeats enclave security
- NEVER log or store decrypted data from KMS outside the enclave
- NEVER use wildcard PCR conditions that match any enclave image
- NEVER modify KMS key policy without testing with the current enclave image first

## Phase 3 — Rollback

1. If key policy was incorrectly modified, restore previous policy: `aws kms put-key-policy --key-id <id> --policy-name default --policy file://previous-policy.json`
2. If enclave image was rebuilt, update key policy with new PCR values
3. If vsock proxy was misconfigured, restore previous configuration and restart
4. Verify KMS operations work from the enclave after fixes
5. Document PCR values and key policy for audit trail

## Common Issues

- symptoms: "KMS Decrypt returns AccessDeniedException with attestation error"
  diagnosis: "PCR values in KMS key policy do not match the running enclave image. Image was rebuilt without updating the key policy."
  resolution: "Get current PCR values: nitro-cli describe-eif --eif-path <path>. Update KMS key policy with new PCR0/PCR1/PCR2 values."

- symptoms: "KMS request times out from enclave"
  diagnosis: "Vsock proxy is not running on parent, or proxy cannot reach KMS endpoint."
  resolution: "Verify vsock proxy is running and configured with correct KMS endpoint. Check parent instance network connectivity to KMS."

## Output Format

```yaml
root_cause: "kms_integration — <specific_cause>"
evidence:
  - type: kms_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix KMS key policy or vsock proxy"
  long_term: "Automate PCR tracking and key policy updates in CI/CD"
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
