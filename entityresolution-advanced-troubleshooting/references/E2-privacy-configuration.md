---
title: "E2 — Privacy Configuration"
description: "Diagnose privacy-enhancing configuration issues including hashing, encryption, and data protection settings"
status: active
severity: MEDIUM
triggers: ["privacy config error", "hashed output unexpected", "encryption issue", "data protection settings", "privacy configuration"]
owner: devops-agent
objective: "Resolve privacy configuration issues while maintaining data protection requirements"
context: "Privacy-enhancing configurations affect output content. Hashing, encryption, and redaction change how data appears in output. These are by design for data protection. Misconfiguration can make output unusable or insufficiently protected."
---

## Phase 1 — Triage

MUST:
- Get workflow configuration with privacy settings: `aws entityresolution get-matching-workflow --workflow-name <workflow-name>`
- Review output data to understand what privacy transformations were applied
- Verify KMS key configuration if encryption is used

SHOULD:
- Check if privacy settings match business requirements
- Verify which fields are hashed, encrypted, or redacted
- Review the schema mapping for privacy-related field configurations

MAY:
- Compare output with and without privacy settings (using test data)
- Check KMS key permissions for encryption/decryption
- Review compliance requirements for data protection

## Phase 2 — Remediate

MUST:
- Align privacy configuration with business and compliance requirements
- Fix KMS key permissions if encryption fails
- Ensure output consumers can handle privacy-transformed data

SHOULD:
- Document which fields are protected and how
- Test privacy configuration with sample data before production
- Verify downstream systems can decrypt or process hashed data

MAY:
- Implement field-level privacy controls for granular protection
- Set up audit logging for privacy configuration changes
- Create separate workflows for different privacy requirements

## Safety Ratings

```
safety_ratings:
  - "get-matching-workflow: GREEN — read-only workflow configuration"
  - "update-matching-workflow (privacy): YELLOW — modifies data protection settings"
  - "kms describe-key: GREEN — read-only key metadata"
  - "kms create-grant: YELLOW — grants key usage permissions"
```

## Escalation Conditions

- Privacy configuration changes require compliance team approval
- Output data is insufficiently protected for regulatory requirements
- KMS key management changes affect multiple services
- Privacy settings make output unusable for business purposes

## Data Sensitivity

- `aws entityresolution get-matching-workflow`: LOW — workflow configuration
- Privacy configuration: MEDIUM — reveals data protection strategy
- Output data: HIGH — contains PII (protected or unprotected)
- KMS key configuration: MEDIUM — reveals encryption setup

## Prohibited Actions

- NEVER disable privacy protections without compliance team approval
- NEVER store unencrypted PII in output when privacy is required
- NEVER share KMS keys used for privacy protection broadly
- NEVER log or display privacy-protected field values in plain text

## Phase 3 — Rollback

1. If privacy settings were weakened, restore previous configuration immediately
2. If output was written without required privacy protections, delete it
3. If KMS key configuration was changed, restore previous settings
4. Verify privacy protections are active after rollback
5. Report any privacy configuration incidents to the compliance team

## Common Issues

- symptoms: "Output contains hashed values instead of expected plain text"
  diagnosis: "Privacy configuration is hashing output fields as designed."
  resolution: "This is expected behavior. If plain text is needed, review privacy requirements. Adjust privacy settings only with compliance approval."

- symptoms: "Privacy encryption fails with KMS error"
  diagnosis: "The service role cannot use the KMS key for encryption."
  resolution: "Add the service role to the KMS key policy with kms:GenerateDataKey and kms:Encrypt permissions."

- symptoms: "Downstream system cannot process privacy-protected output"
  diagnosis: "The downstream system does not support hashed or encrypted data formats."
  resolution: "Update the downstream system to handle protected data, or adjust privacy configuration to use a compatible format."

## Output Format

```yaml
root_cause: "privacy_configuration — <specific_cause>"
evidence:
  - type: privacy_error
    content: "<privacy configuration and output details>"
severity: MEDIUM
mitigation:
  immediate: "Fix privacy configuration"
  long_term: "Implement privacy configuration governance"
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
