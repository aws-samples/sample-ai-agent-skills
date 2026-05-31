---
title: "A2 — Key Import Errors"
description: "Diagnose key import failures using TR-31 key blocks"
status: active
severity: HIGH
triggers: ["key import failed", "import-key error", "TR-31 error", "wrapping key"]
owner: devops-agent
objective: "Resolve key import failures in Payment Cryptography"
context: "Key import requires TR-31 key block format with wrapping keys. The wrapping key exchange must be completed first."
---

## Phase 1 — Triage

MUST:
- Verify wrapping key was obtained: `aws payment-cryptography get-parameters-for-import --key-material-type TR31_KEY_BLOCK --wrapping-key-algorithm RSA_2048`
- Check if wrapping key certificate has expired (valid for limited time)
- Verify TR-31 key block format is correct

SHOULD:
- Verify the key material type matches the import parameters
- Check that the wrapping key algorithm matches between export and import

MAY:
- Review CloudTrail for ImportKey errors
- Verify source system key export format

## Phase 2 — Remediate

MUST:
- Obtain fresh wrapping key parameters if certificate expired
- Fix TR-31 key block format (header, key material, MAC)
- Ensure wrapping key algorithm matches between systems

SHOULD:
- Implement automated key import workflows
- Document key exchange procedures

MAY:
- Use TR-34 for asymmetric key distribution
- Implement key import validation checks

## Safety Ratings

```
safety_ratings:
  - "get-parameters-for-import: YELLOW — generates temporary wrapping key pair"
  - "import-key: RED — imports external key material into HSM"
  - "CloudTrail lookup: GREEN — read-only audit log review"
  - "Key block validation: GREEN — read-only format check"
```

## Escalation Conditions

- Key import fails with cryptographic validation errors
- Wrapping key exchange with external HSM requires coordination
- TR-31 key block format incompatibility between systems
- Key import needed for production payment processing deadline

## Data Sensitivity

- Wrapping key certificate: HIGH — used to protect key material in transit
- TR-31 key block: CRITICAL — contains encrypted key material
- Import parameters: MEDIUM — contains wrapping key metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals import attempts

## Prohibited Actions

- NEVER transmit unwrapped key material over insecure channels
- NEVER reuse expired wrapping key certificates
- NEVER import keys without verifying the source system identity
- NEVER store TR-31 key blocks in plain text logs

## Phase 3 — Rollback

1. If key was imported with wrong attributes: schedule deletion of the imported key
2. If wrapping key parameters were generated unnecessarily: they expire automatically
3. If import created a duplicate key: identify and remove the duplicate
4. Verify imported key works correctly before using in production
5. Document the import operation for PCI DSS key management audit

## Common Issues

- symptoms: "ImportKey returns ValidationException for key block"
  diagnosis: "TR-31 key block format is incorrect or wrapping key mismatch."
  resolution: "Verify TR-31 header, key usage, algorithm, and MAC. Ensure wrapping key from get-parameters-for-import was used."

- symptoms: "ImportKey returns expired wrapping key error"
  diagnosis: "Wrapping key certificate from get-parameters-for-import has expired."
  resolution: "Call get-parameters-for-import again to get fresh wrapping key parameters. Complete import within the validity window."

## Output Format

```yaml
root_cause: "key_import — <specific_cause>"
evidence:
  - type: import_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix import issue and retry"
  long_term: "Automate key import workflows"
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
