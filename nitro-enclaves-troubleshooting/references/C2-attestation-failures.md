---
title: "C2 — Attestation Failures"
description: "Diagnose enclave attestation document generation and verification failures"
status: active
severity: CRITICAL
triggers: ["attestation failed", "attestation document invalid", "PCR mismatch", "attestation verification error"]
owner: devops-agent
objective: "Resolve attestation failures to enable cryptographic identity verification"
context: "Attestation documents are cryptographically signed by the Nitro hypervisor. They contain PCR values, enclave metadata, and optional user data. Attestation is used by KMS and custom verifiers to confirm enclave identity."
---

## Phase 1 — Triage

MUST:
- Verify enclave is running: `nitro-cli describe-enclaves`
- Get expected PCR values from EIF: `nitro-cli describe-eif --eif-path <path>`
- Check enclave console for attestation errors: `nitro-cli console --enclave-id <id>`

SHOULD:
- Verify the enclave application correctly calls the attestation API (NSM library)
- Compare expected PCR values with those in the KMS key policy or verifier configuration
- Check if enclave was built with debug mode (debug mode changes PCR values)

MAY:
- Review attestation document structure for completeness
- Check Nitro Security Module (NSM) availability inside the enclave
- Verify attestation document timestamp is within acceptable window

## Phase 2 — Remediate

MUST:
- Ensure enclave image matches the PCR values expected by verifiers
- Rebuild enclave in production mode (not debug) for production attestation
- Update KMS key policies or custom verifiers with correct PCR values after image changes

SHOULD:
- Use all three PCR values (PCR0, PCR1, PCR2) for comprehensive attestation
- Implement PCR value tracking as part of the build pipeline
- Verify attestation works end-to-end before deploying to production

MAY:
- Include custom user data in attestation documents for additional context
- Implement attestation document caching for performance (within validity window)
- Set up monitoring for attestation failures

## Safety Ratings

```
safety_ratings:
  - "describe-enclaves: GREEN — read-only status check"
  - "describe-eif: GREEN — read-only PCR retrieval"
  - "console: GREEN — read-only console output"
  - "kms put-key-policy: RED — modifies cryptographic access controls"
```

## Escalation Conditions

- PCR values change unexpectedly without image rebuild (potential tampering)
- Attestation document signature verification fails (potential hypervisor issue)
- Production enclave attestation breaks after routine image update
- Custom attestation verifier rejects valid enclave documents

## Data Sensitivity

- `nitro-cli describe-eif`: MEDIUM — reveals enclave identity (PCR values)
- Attestation documents: HIGH — contain cryptographic identity proof
- PCR values: MEDIUM — identify specific enclave image versions
- KMS key policies: HIGH — control access to encrypted data

## Prohibited Actions

- NEVER deploy debug-mode enclaves in production — debug mode has different PCR values and weaker isolation
- NEVER ignore PCR mismatches — they indicate the enclave image has changed
- NEVER bypass attestation verification to work around failures
- NEVER share attestation documents or PCR values publicly — they reveal enclave identity

## Phase 3 — Rollback

1. If enclave image was changed, rebuild from the previous Docker image version
2. Update KMS key policies with PCR values from the restored image
3. Restart enclave with the restored EIF
4. Verify attestation succeeds with KMS and any custom verifiers
5. Document the PCR values for the restored image version

## Common Issues

- symptoms: "KMS rejects attestation with PCR mismatch"
  diagnosis: "Enclave image was rebuilt (even minor changes) causing PCR values to change. KMS key policy has old PCR values."
  resolution: "Get new PCR values from nitro-cli describe-eif. Update KMS key policy with new values."

- symptoms: "Attestation document generation fails inside enclave"
  diagnosis: "Enclave application is not correctly using the NSM library, or enclave was started in a mode that does not support attestation."
  resolution: "Verify NSM library is included in the Docker image. Ensure enclave is started without --debug-mode for production attestation."

## Output Format

```yaml
root_cause: "attestation — <specific_cause>"
evidence:
  - type: attestation_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix attestation configuration"
  long_term: "Automate PCR tracking and attestation validation in CI/CD"
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
