---
title: "G1 — Data Encryption"
description: "Diagnose Amazon AppFlow data encryption and KMS issues"
status: active
severity: HIGH
triggers: ["encryption error", "KMS error", "key access denied", "decrypt failed"]
owner: devops-agent
objective: "Resolve data encryption issues including KMS key permissions, key policies, and encryption configuration"
context: "AppFlow encrypts data at rest using KMS. Default uses AWS-managed key. Custom KMS keys require explicit grants to appflow.amazonaws.com."
---

## Phase 1 — Triage

MUST:
- Check flow encryption config: `aws appflow describe-flow --flow-name <flow-name>`
- Verify KMS key exists and is enabled: `aws kms describe-key --key-id <key-id>`
- Check KMS key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`

SHOULD:
- Verify appflow.amazonaws.com has GenerateDataKey and Decrypt permissions in key policy
- Check if key is in the same region as the flow

MAY:
- Review CloudTrail for KMS access denied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com --max-results 20`

## Phase 2 — Remediate

MUST:
- Update KMS key policy to grant appflow.amazonaws.com GenerateDataKey and Decrypt
- Ensure KMS key is enabled and not pending deletion
- Verify key is in the same region as the AppFlow flow

SHOULD:
- Use key aliases for easier management
- Document which flows use which KMS keys

MAY:
- Implement key rotation for compliance
- Use separate keys for different data classification levels

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow details"
  - "kms describe-key: GREEN — read-only key metadata"
  - "kms get-key-policy: GREEN — read-only key policy"
  - "kms put-key-policy: RED — modifies KMS key access policy"
```

## Escalation Conditions

- KMS key is managed by a different team or account
- Key policy changes affect other services using the same key
- Key is scheduled for deletion and cannot be cancelled
- Cross-account KMS key access requires both account policies

## Data Sensitivity

- `aws kms describe-key`: LOW — reveals key metadata only
- `aws kms get-key-policy`: HIGH — reveals key access policy and trusted principals
- `aws cloudtrail lookup-events`: MEDIUM — reveals KMS API call history

## Prohibited Actions

- NEVER disable KMS encryption to work around key permission errors
- NEVER grant kms:* permissions as a troubleshooting shortcut
- NEVER modify key policies without understanding the impact on other services
- NEVER use the AWS-managed key for production workloads requiring key rotation control

## Phase 3 — Rollback

1. If KMS key policy was changed, revert to previous policy: `aws kms put-key-policy --key-id <key-id> --policy-name default --policy file://previous-policy.json`
2. If flow encryption was changed, restore original encryption configuration
3. Verify other services using the same KMS key are not affected
4. Confirm flow executions work with the restored encryption settings
5. Document the correct KMS key policy for AppFlow access

## Common Issues

- symptoms: "Flow execution fails with 'AccessDeniedException' for KMS key"
  diagnosis: "KMS key policy does not grant appflow.amazonaws.com GenerateDataKey permission."
  resolution: "Add key policy statement: Allow appflow.amazonaws.com to use kms:GenerateDataKey and kms:Decrypt on the key."

## Output Format

```yaml
root_cause: "data_encryption — <specific_cause>"
evidence:
  - type: kms_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key policy"
  long_term: "Audit key policies regularly"
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
