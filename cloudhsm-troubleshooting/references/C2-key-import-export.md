---
title: "C2 — Key Import/Export"
description: "Diagnose and resolve CloudHSM key import and export failures"
status: active
severity: HIGH
triggers:
  - "key import failed"
  - "key export failed"
  - "wrapKey error"
  - "unwrapKey error"
  - "importKey failed"
owner: devops-agent
objective: "Resolve key import/export failures and ensure secure key transfer to/from CloudHSM"
context: "CloudHSM supports importing and exporting keys using wrapping mechanisms. Keys can be imported as wrapped (encrypted) key material using wrapKey/unWrapKey in KMU. Import/export failures commonly arise from wrapping key mismatches, key attribute restrictions (extractable flag), unsupported key types, or incorrect file formats. FIPS 140-2 Level 3 compliance restricts how keys can leave the HSM boundary."
---

## Phase 1 — Triage

MUST:
- Verify the wrapping key exists and is accessible: `findKey -t <wrapping-key-type>` (in KMU)
- Check the key attributes of the target key: `getAttribute <key-handle> <output-file>` (in KMU)
- Verify the extractable attribute for export operations: check OBJ_ATTR_EXTRACTABLE in attributes
- Confirm the CU has ownership or sharing access to both wrapping and target keys: `getKeyInfo <key-handle>`
- Check CloudHSM client logs for detailed error: `sudo journalctl -u cloudhsm-client --no-pager -n 50`

SHOULD:
- Verify the wrapping key algorithm is compatible with the target key type
- Check the key file format for import operations (DER, PEM, raw)
- Verify the HSM firmware version supports the requested import/export mechanism

MAY:
- Review the CloudHSM SDK documentation for supported wrapping mechanisms
- Check if the key was generated with the correct attributes for future export
- Test import/export with a non-production key first

## Phase 2 — Remediate

MUST:
- If wrapping key not found: generate a suitable AES wrapping key: `genSymKey -t 31 -s 32 -l wrapping-key` (in KMU)
- If extractable flag is false: the key cannot be exported; generate a new key with extractable=true if export is required
- If import file format is wrong: convert the key material to the expected format (DER for RSA, raw for AES)
- If unwrapKey fails: verify the wrapping key matches the one used for wrapKey

SHOULD:
- Use AES_KEY_WRAP (mechanism 0x1091) for FIPS-compliant key wrapping
- Document the wrapping key handle and mechanism used for future reference
- Verify the imported key works correctly after import by performing a test operation

MAY:
- Set up a dedicated wrapping key pair for regular key transfer operations
- Implement automated key import procedures for key rotation workflows
- Create a key import/export runbook specific to the organization's compliance requirements

## Common Issues

- symptoms: "wrapKey returns Cfm3WrapKey error with INVALID_ARGUMENT"
  diagnosis: "Incompatible wrapping mechanism or key type combination."
  resolution: "Use AES wrapping key with AES_KEY_WRAP mechanism. Verify target key type is supported for the chosen mechanism."

- symptoms: "unWrapKey fails with Cfm3UnWrapKey error"
  diagnosis: "Wrapping key mismatch, corrupted wrapped key file, or wrong mechanism specified."
  resolution: "Ensure the same wrapping key and mechanism are used for wrap and unwrap. Verify file integrity."

- symptoms: "Cannot export key — extractable attribute is false"
  diagnosis: "Key was generated with OBJ_ATTR_EXTRACTABLE=false (default for some key types)."
  resolution: "This attribute cannot be changed after creation. Generate a new key with -attr OBJ_ATTR_EXTRACTABLE=true."

- symptoms: "importPrivateKey fails with parsing error"
  diagnosis: "Key file format mismatch — expected DER but received PEM or vice versa."
  resolution: "Convert the key file: `openssl rsa -in key.pem -outform DER -out key.der` (for RSA keys)."

## Output Format

```yaml
root_cause: "Key import/export failure — <specific_cause>"
evidence:
  - type: key_attributes
    content: "<getAttribute output for target and wrapping keys>"
  - type: error_details
    content: "<Cfm3 error code and message>"
severity: HIGH
mitigation:
  immediate: "Fix wrapping key/mechanism mismatch or key attributes"
  long_term: "Standardize key import/export procedures with documented wrapping keys and mechanisms"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (findKey/getAttribute/getKeyInfo): GREEN — read-only"
  - "Phase 2 wrapping key generation: YELLOW — creates new key material"
  - "Phase 2 key export operations: RED — key material leaves HSM boundary"

## Escalation Conditions

escalation_conditions:
  - "Key export is required for compliance or audit purposes"
  - "Imported key material may have been compromised during transfer"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Key import/export involves cross-region or cross-account transfer"
  - "Issue involves FIPS compliance requirements for key wrapping"

## Data Sensitivity

data_sensitivity:
  - command: "wrapKey output file"
    sensitivity: CRITICAL
    contains: "Encrypted key material — handle as sensitive cryptographic data"
  - command: "getAttribute"
    sensitivity: HIGH
    contains: "Key attributes and metadata"
  - command: "findKey"
    sensitivity: HIGH
    contains: "Key handles and types"

## Prohibited Actions

prohibited_actions:
  - "NEVER store unwrapped key material on disk in plaintext"
  - "NEVER transfer wrapped key files over unencrypted channels"
  - "NEVER use weak or test wrapping keys for production key transfers"
  - "NEVER skip integrity verification after key import"
  - "NEVER export production keys without documented authorization and audit trail"

## Phase 3 — Rollback

- If an imported key is incorrect or compromised, delete it immediately from the HSM
- If a wrapping key was created for the operation, retain it if needed for future unwrap or delete if single-use
- If key export produced a file, securely delete the wrapped key file after transfer: `shred -u <wrapped-key-file>`
- Verify applications using the original key are unaffected by import/export operations
- Document all key transfers in the organization's key management audit log
