---
title: "E2 — Encryption Configuration"
description: "Diagnose KMS encryption configuration issues for HealthImaging"
status: active
severity: HIGH
triggers: ["encryption error", "KMS key issue", "cannot decrypt", "encryption configuration"]
owner: devops-agent
objective: "Resolve KMS encryption configuration issues for HealthImaging datastores"
context: "Datastores use either AWS-managed or customer-managed KMS keys. Encryption is set at creation and cannot be changed."
---

## Phase 1 — Triage

MUST:
- Check datastore encryption configuration: `aws medical-imaging get-datastore --datastore-id <datastore-id>`
- Verify KMS key status: `aws kms describe-key --key-id <key-id>`
- Check KMS key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`

SHOULD:
- Verify KMS grants for HealthImaging service: `aws kms list-grants --key-id <key-id>`
- Check if key is in a different account (cross-account KMS)

MAY:
- Review CloudTrail for KMS-related errors
- Check KMS key rotation status: `aws kms get-key-rotation-status --key-id <key-id>`

## Phase 2 — Remediate

MUST:
- Re-enable disabled KMS keys: `aws kms enable-key --key-id <key-id>`
- Cancel scheduled key deletion: `aws kms cancel-key-deletion --key-id <key-id>`
- Fix KMS key policy to allow HealthImaging service access

SHOULD:
- Enable automatic key rotation for customer-managed keys
- Set up CloudWatch alarms for KMS key state changes

MAY:
- Implement key policy templates for HealthImaging datastores
- Use AWS Config rules to monitor KMS key compliance

## Safety Ratings

```
safety_ratings:
  - "kms describe-key: GREEN — read-only key metadata"
  - "kms get-key-policy: GREEN — read-only policy retrieval"
  - "kms enable-key: YELLOW — re-enables a disabled key"
  - "kms cancel-key-deletion: RED — cancels scheduled key deletion, affects security posture"
```

## Escalation Conditions

- KMS key has been deleted (not just scheduled) — data is permanently inaccessible
- Cross-account KMS key access requires organizational policy changes
- Key policy changes require security team approval
- Encryption configuration change requires datastore migration

## Data Sensitivity

- `aws kms describe-key`: LOW — returns key metadata
- `aws kms get-key-policy`: MEDIUM — reveals key access policy
- `aws kms list-grants`: MEDIUM — reveals which services have key access
- KMS key material: CRITICAL — never exposed through API

## Prohibited Actions

- NEVER delete or schedule deletion of KMS keys used by active datastores
- NEVER disable KMS keys without understanding impact on datastore access
- NEVER share KMS key policies containing account IDs publicly
- NEVER create KMS grants broader than necessary for HealthImaging

## Phase 3 — Rollback

1. If KMS key was disabled: re-enable immediately: `aws kms enable-key --key-id <key-id>`
2. If key deletion was scheduled: cancel: `aws kms cancel-key-deletion --key-id <key-id>`
3. If key policy was modified incorrectly: restore previous policy version
4. If grants were revoked: recreate necessary grants for HealthImaging
5. Verify all datastores using the key are accessible after changes

## Common Issues

- symptoms: "Operations fail with KMSKeyDisabledException"
  diagnosis: "Customer-managed KMS key has been disabled."
  resolution: "Re-enable the key: aws kms enable-key --key-id <key-id>. Investigate why it was disabled."

- symptoms: "Operations fail with KMSKeyNotFoundException"
  diagnosis: "KMS key has been deleted or the key ID is incorrect."
  resolution: "If key was scheduled for deletion, cancel it. If already deleted, data is permanently inaccessible — escalate immediately."

## Output Format

```yaml
root_cause: "encryption_configuration — <specific_cause>"
evidence:
  - type: encryption_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key issue"
  long_term: "Implement KMS key monitoring and protection"
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
