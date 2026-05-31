---
title: "D2 — Key Export Errors"
description: "Diagnose key export failures using TR-31 key blocks"
status: active
severity: HIGH
triggers: ["key export failed", "export-key error", "cannot export key", "non-exportable"]
owner: devops-agent
objective: "Resolve key export failures in Payment Cryptography"
context: "Key export requires Exportable=true set at creation. Export uses TR-31 format with wrapping keys."
---

## Phase 1 — Triage

MUST:
- Check key exportability: `aws payment-cryptography get-key --key-identifier <key-arn>` (check Exportable attribute)
- Verify IAM permissions for payment-cryptography:ExportKey
- Get export parameters: `aws payment-cryptography get-parameters-for-export --key-material-type TR31_KEY_BLOCK --signing-key-algorithm RSA_2048`

SHOULD:
- Verify the wrapping key from the receiving system is valid
- Check export key material type matches requirements

MAY:
- Review CloudTrail for ExportKey errors
- Verify receiving system can process TR-31 key blocks

## Phase 2 — Remediate

MUST:
- If key is not exportable: create a new key with Exportable=true
- Fix IAM permissions for export operations
- Use correct wrapping key from the receiving system

SHOULD:
- Implement key export workflows with dual control
- Document key export procedures per PCI DSS requirements

MAY:
- Automate key export for scheduled key distribution
- Implement export validation checks

## Safety Ratings

```
safety_ratings:
  - "get-key: GREEN — read-only key metadata"
  - "get-parameters-for-export: YELLOW — generates signing key pair for export"
  - "export-key: RED — exports key material (wrapped) from HSM"
  - "CloudTrail lookup: GREEN — read-only audit log review"
```

## Escalation Conditions

- Key export needed for production system but key is non-exportable
- Export wrapping key exchange with external system requires coordination
- TR-31 format incompatibility with receiving system
- Key export required for PCI DSS compliance deadline

## Data Sensitivity

- Exported key block: CRITICAL — contains wrapped key material
- Signing key certificate: HIGH — used to authenticate export
- Export parameters: MEDIUM — contains wrapping key metadata
- Key exportability status: LOW — key attribute metadata

## Prohibited Actions

- NEVER transmit exported key blocks over insecure channels
- NEVER store exported key blocks in plain text
- NEVER export keys without dual control authorization
- NEVER export keys to unauthorized receiving systems

## Phase 3 — Rollback

1. If key was exported to wrong system: notify security team immediately
2. If export parameters were generated unnecessarily: they expire automatically
3. If new exportable key was created: schedule deletion if not needed
4. Verify exported key was received correctly by the destination system
5. Document all export operations for PCI DSS key management audit

## Common Issues

- symptoms: "ExportKey returns key is not exportable"
  diagnosis: "Key was created with Exportable=false."
  resolution: "Create a new key with Exportable=true and the same attributes. Migrate operations to the new key."

- symptoms: "ExportKey returns wrapping key validation error"
  diagnosis: "Wrapping key from receiving system is invalid or expired."
  resolution: "Obtain fresh wrapping key certificate from the receiving system. Verify certificate format and validity."

## Output Format

```yaml
root_cause: "key_export — <specific_cause>"
evidence:
  - type: export_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix export issue"
  long_term: "Implement key distribution automation"
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
