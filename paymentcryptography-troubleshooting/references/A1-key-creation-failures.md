---
title: "A1 — Key Creation Failures"
description: "Diagnose payment cryptography key creation failures"
status: active
severity: HIGH
triggers: ["cannot create key", "key creation failed", "CreateKey error"]
owner: devops-agent
objective: "Resolve payment cryptography key creation issues"
context: "Keys have specific attributes (class, usage, algorithm) set at creation. Attributes are immutable after creation."
---

## Phase 1 — Triage

MUST:
- Check existing keys: `aws payment-cryptography list-keys`
- Verify IAM permissions for payment-cryptography:CreateKey
- Check service quotas: `aws service-quotas list-service-quotas --service-code payment-cryptography`

SHOULD:
- Verify key attributes are valid combinations (class + usage + algorithm)
- Check if account has reached key limit in the region

MAY:
- Review CloudTrail for CreateKey errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=payment-cryptography.amazonaws.com --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions: ensure payment-cryptography:CreateKey is allowed
- Use valid key attribute combinations per API documentation
- Request quota increase if key limit reached

SHOULD:
- Tag keys for identification and cost allocation
- Document key purpose and usage in tags

MAY:
- Automate key creation with CloudFormation or Terraform
- Implement key naming conventions

## Safety Ratings

```
safety_ratings:
  - "list-keys: GREEN — read-only enumeration of existing keys"
  - "CreateKey: YELLOW — creates cryptographic key material in HSM"
  - "service-quotas list-service-quotas: GREEN — read-only quota check"
  - "CloudFormation key deployment: YELLOW — creates key infrastructure"
```

## Escalation Conditions

- Key creation fails with internal service errors after IAM fixes
- Required key attribute combination is not supported
- Service quota increase request is denied
- Key creation needed for PCI DSS compliance deadline

## Data Sensitivity

- `aws payment-cryptography list-keys`: LOW — returns key metadata only
- Key material: CRITICAL — never exposed through API, managed by HSM
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities
- Key attributes: LOW — describes key capabilities, not key material

## Prohibited Actions

- NEVER create keys without documenting their intended purpose
- NEVER create exportable keys unless export is explicitly required
- NEVER use overly broad key usage attributes
- NEVER create keys outside of approved key management procedures

## Phase 3 — Rollback

1. If a key was created with wrong attributes: schedule deletion: `aws payment-cryptography delete-key --key-identifier <key-arn> --delete-key-in-days 7`
2. If key was created in wrong region: create in correct region and delete the incorrect one
3. If tags were applied incorrectly: update tags: `aws payment-cryptography tag-resource --resource-arn <arn> --tags Key=Purpose,Value=<value>`
4. Verify no operations are using the incorrectly created key before deletion
5. Document the key creation and any rollback for PCI DSS audit trail

## Common Issues

- symptoms: "CreateKey returns ValidationException for key attributes"
  diagnosis: "Invalid combination of key class, usage, and algorithm."
  resolution: "Review supported attribute combinations. For example, SYMMETRIC_KEY with TR31_P0_PIN_ENCRYPTION_KEY usage requires TDES_3KEY or AES_128 algorithm."

- symptoms: "CreateKey returns AccessDeniedException"
  diagnosis: "IAM policy missing payment-cryptography:CreateKey permission."
  resolution: "Add payment-cryptography:CreateKey to the IAM policy."

## Output Format

```yaml
root_cause: "key_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Implement key management automation"
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
