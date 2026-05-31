---
title: "F2 — Key Rotation"
description: "Diagnose key rotation failures and lifecycle management issues"
status: active
severity: HIGH
triggers: ["key rotation failed", "key expired", "rotation schedule", "key lifecycle"]
owner: devops-agent
objective: "Resolve key rotation and lifecycle management issues"
context: "Key rotation is a manual process requiring new key creation, migration, and old key retirement. PCI DSS mandates periodic rotation."
---

## Phase 1 — Triage

MUST:
- Check key creation dates: `aws payment-cryptography get-key --key-identifier <key-arn>` (check CreateTimestamp)
- Verify key is still enabled and not scheduled for deletion
- Check if replacement key has been created

SHOULD:
- Review key rotation schedule documentation
- Verify all systems using the old key have been identified

MAY:
- Check CloudTrail for key usage patterns to identify all consumers
- Review key tags for rotation metadata

## Phase 2 — Remediate

MUST:
- Create new replacement key with same attributes
- Update key alias to point to new key: `aws payment-cryptography update-alias --alias-name alias/<name> --key-arn <new-key-arn>`
- Verify all systems work with the new key before retiring old key

SHOULD:
- Implement a rotation runbook with step-by-step procedures
- Tag keys with rotation dates and schedules
- Maintain old key for decryption of existing data during transition

MAY:
- Automate key rotation with Lambda functions
- Implement key rotation monitoring and alerting

## Safety Ratings

```
safety_ratings:
  - "get-key: GREEN — read-only key metadata"
  - "create-key: YELLOW — creates new key material"
  - "update-alias: RED — changes which key an alias points to (affects all consumers)"
  - "delete-key: RED — schedules key for permanent deletion"
```

## Escalation Conditions

- Key rotation deadline approaching with no replacement key
- Old key cannot be retired due to undiscovered consumers
- Key rotation causes production transaction failures
- PCI DSS audit requires evidence of completed rotation

## Data Sensitivity

- Key creation timestamps: LOW — operational metadata
- Key usage patterns: MEDIUM — reveals which systems use which keys
- Rotation schedules: LOW — operational planning data
- Key ARNs: LOW — resource identifiers

## Prohibited Actions

- NEVER delete old key before confirming all consumers have migrated
- NEVER rotate keys during peak transaction periods without approval
- NEVER skip the verification step after alias reassignment
- NEVER perform key rotation without dual control authorization

## Phase 3 — Rollback

1. If alias was updated to wrong key: revert alias: `aws payment-cryptography update-alias --alias-name alias/<name> --key-arn <old-key-arn>`
2. If old key was scheduled for deletion prematurely: restore it: `aws payment-cryptography restore-key --key-identifier <key-arn>`
3. If new key has issues: revert all consumers to old key
4. Verify all transaction processing is functioning after rollback
5. Document the rotation attempt and rollback for PCI DSS audit

## Common Issues

- symptoms: "Transactions fail after key rotation"
  diagnosis: "Some systems still reference old key ARN instead of alias."
  resolution: "Identify systems using direct key ARN. Update to use alias or new key ARN. Keep old key enabled during transition."

- symptoms: "Cannot delete old key — still in use"
  diagnosis: "Undiscovered consumers still using the old key."
  resolution: "Check CloudTrail for API calls using the old key ARN. Update all consumers before scheduling deletion."

## Output Format

```yaml
root_cause: "key_rotation — <specific_cause>"
evidence:
  - type: rotation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix rotation issue"
  long_term: "Implement automated rotation workflow"
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
