---
title: "D2 — Cryptographic Computing Issues"
description: "Diagnose Clean Rooms cryptographic computing problems including C3R encryption and configuration"
status: active
severity: HIGH
triggers:
  - "cryptographic computing"
  - "c3r encryption"
  - "encrypted table"
  - "crypto clean rooms"
  - "client-side encryption"
owner: devops-agent
objective: "Resolve cryptographic computing configuration and encryption issues"
context: "Clean Rooms cryptographic computing enables queries on encrypted data using the C3R (Clean Rooms Cryptographic Computing) encryption client. Data must be encrypted before upload. Common issues include encryption format errors, key management, and schema mismatches."
---

## Phase 1 — Triage

MUST:
- Verify the collaboration has cryptographic computing enabled
- Check configured table encryption settings
- Verify data was encrypted with the C3R client before upload
- Check the collaboration's crypto parameters match the encryption schema

SHOULD:
- Verify the C3R encryption schema matches the Glue table schema
- Check if the shared secret or key material is correctly configured
- Verify all collaborating parties used compatible encryption parameters

MAY:
- Check CloudTrail for cryptographic computing events
- Verify the C3R client version is compatible

## Phase 2 — Remediate

MUST:
- If data not encrypted, encrypt with C3R client and re-upload to S3
- Ensure collaboration crypto parameters match encryption schema
- Verify all parties use the same shared secret for join columns

SHOULD:
- Document the encryption schema and share with collaborators
- Test with a small dataset before encrypting full data

## Output Format

```yaml
root_cause: "cryptographic_computing — <specific_cause>"
evidence:
  - type: crypto_config
    content: "<cryptographic computing configuration>"
  - type: encryption_status
    content: "<data encryption status>"
severity: HIGH
mitigation:
  immediate: "Fix encryption configuration or re-encrypt data"
  long_term: "Automate C3R encryption in data pipeline"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Encrypt data with C3R client and re-upload | YELLOW |
| Ensure collaboration crypto parameters match | YELLOW |
| Verify all parties use the same shared secret | RED |
| Document encryption schema | GREEN |
| Test with small dataset | GREEN |

## Escalation Conditions

- Fix requires modifying cryptographic computing parameters
- Fix involves sharing or rotating encryption keys or shared secrets

## Data Sensitivity

- **CRITICAL**: cryptographic computing configurations contain encryption keys and shared secrets
- C3R encryption parameters and shared secrets must be handled with extreme care

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If data was re-encrypted with wrong parameters, re-encrypt with correct C3R configuration and re-upload
2. If crypto parameters were changed, coordinate with all collaborators to restore previous settings
3. If shared secret was modified, ensure all parties update to the correct secret
4. Verify rollback by confirming encrypted data can be queried successfully in the collaboration

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
