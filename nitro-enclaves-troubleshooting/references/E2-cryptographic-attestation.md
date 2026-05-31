---
title: "E2 — Cryptographic Attestation"
description: "Diagnose cryptographic attestation document validation and third-party verifier issues"
status: active
severity: HIGH
triggers: ["attestation validation failed", "COSE signature invalid", "attestation document rejected", "third-party verifier error"]
owner: devops-agent
objective: "Resolve cryptographic attestation validation issues for enclave identity verification"
context: "Attestation documents are CBOR-encoded, COSE-signed by the Nitro hypervisor. They contain PCR values, public key, user data, and nonce. Third-party verifiers validate the signature chain against AWS Nitro root certificate."
---

## Phase 1 — Triage

MUST:
- Verify enclave is running and producing attestation documents: `nitro-cli describe-enclaves`
- Check enclave console for attestation generation errors: `nitro-cli console --enclave-id <id>`
- Get expected PCR values: `nitro-cli describe-eif --eif-path <path>`

SHOULD:
- Verify the attestation document is being correctly transmitted via vsock to the verifier
- Check if the verifier has the correct AWS Nitro root certificate for signature validation
- Verify attestation document is not expired (check timestamp)

MAY:
- Parse attestation document to inspect CBOR structure and COSE signature
- Verify certificate chain from attestation document to AWS Nitro root CA
- Check if debug mode affects attestation (debug enclaves have different attestation properties)

## Phase 2 — Remediate

MUST:
- Ensure enclave application correctly uses NSM library to generate attestation documents
- Verify verifier uses the correct AWS Nitro Attestation root certificate
- Ensure attestation document is transmitted intact via vsock (no truncation or corruption)

SHOULD:
- Include a nonce in attestation requests to prevent replay attacks
- Validate all PCR values (PCR0, PCR1, PCR2) not just PCR0
- Ensure verifier checks certificate chain validity including expiry

MAY:
- Implement attestation document caching with TTL for performance
- Add custom user data to attestation for application-level verification
- Set up attestation validation monitoring and alerting

## Safety Ratings

```
safety_ratings:
  - "describe-enclaves: GREEN — read-only status check"
  - "describe-eif: GREEN — read-only PCR retrieval"
  - "console: GREEN — read-only console output"
  - "Attestation document generation: GREEN — read-only cryptographic operation inside enclave"
```

## Escalation Conditions

- Attestation signature verification fails consistently (potential hypervisor issue — contact AWS)
- AWS Nitro root certificate rotation requires verifier updates
- Attestation documents contain unexpected PCR values (potential image tampering)
- Third-party verifier incompatible with AWS attestation format

## Data Sensitivity

- Attestation documents: HIGH — contain cryptographic identity proof and optional user data
- PCR values: MEDIUM — identify specific enclave image versions
- AWS Nitro root certificate: LOW — publicly available
- Verifier configuration: MEDIUM — contains validation rules and trusted PCR values

## Prohibited Actions

- NEVER skip attestation signature verification — it is the foundation of enclave trust
- NEVER accept attestation documents without nonce validation in security-critical applications
- NEVER trust attestation from debug-mode enclaves for production security decisions
- NEVER hardcode the AWS Nitro root certificate — use a mechanism that supports rotation

## Phase 3 — Rollback

1. If verifier was misconfigured, restore previous validation rules
2. If enclave image was changed, rebuild from previous version and update PCR expectations
3. If attestation library was updated, revert to previous version
4. Verify attestation validation works end-to-end after fixes
5. Document the attestation configuration for security audit

## Common Issues

- symptoms: "Verifier rejects attestation with 'invalid signature' error"
  diagnosis: "Verifier does not have the correct AWS Nitro root certificate, or attestation document was corrupted during vsock transmission."
  resolution: "Update verifier with current AWS Nitro Attestation root certificate. Verify vsock transmission preserves document integrity."

- symptoms: "Attestation document has PCR values that don't match expected values"
  diagnosis: "Enclave image was rebuilt (even with identical source) or started in debug mode."
  resolution: "Verify EIF was built from the expected Docker image. Ensure enclave is not in debug mode. Update expected PCR values if image was intentionally changed."

## Output Format

```yaml
root_cause: "cryptographic_attestation — <specific_cause>"
evidence:
  - type: attestation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix attestation validation"
  long_term: "Implement attestation monitoring and automated PCR tracking"
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
