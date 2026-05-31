---
title: "E1 — Access Control"
description: "Diagnose IAM and access control issues for Payment Cryptography"
status: active
severity: HIGH
triggers: ["access denied", "permission error", "IAM policy", "cannot access key"]
owner: devops-agent
objective: "Resolve access control and IAM permission issues for Payment Cryptography"
context: "Payment Cryptography has separate control plane and data plane permissions. IAM policies must cover both."
---

## Phase 1 — Triage

MUST:
- Check caller identity: `aws sts get-caller-identity`
- Review IAM policies: `aws iam list-attached-role-policies --role-name <role>`
- Test specific permission: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names payment-cryptography:GetKey --resource-arns <key-arn>`

SHOULD:
- Check for SCPs that may deny Payment Cryptography actions
- Verify both control plane (payment-cryptography) and data plane (payment-cryptography-data) permissions

MAY:
- Review CloudTrail for AccessDenied events
- Check resource-based policies on keys

## Phase 2 — Remediate

MUST:
- Add required permissions for both control plane and data plane
- Scope permissions to specific key ARNs for least privilege
- Separate admin (control plane) and operator (data plane) roles

SHOULD:
- Implement IAM policy conditions (source IP, MFA, time-based)
- Use separate roles for key management and cryptographic operations

MAY:
- Set up AWS Config rules to monitor Payment Cryptography IAM policies
- Implement ABAC using key tags

## Safety Ratings

```
safety_ratings:
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "iam simulate-principal-policy: GREEN — read-only permission test"
  - "IAM policy modification: YELLOW — changes access permissions"
  - "SCP modification: RED — affects entire organization"
```

## Escalation Conditions

- Access denied errors persist after IAM policy fixes
- SCP restrictions require organizational policy changes
- Dual control requirements need multiple IAM roles
- PCI DSS audit requires specific access control patterns

## Data Sensitivity

- `aws sts get-caller-identity`: LOW — returns caller metadata
- `aws iam list-attached-role-policies`: LOW — returns policy names
- IAM policies: MEDIUM — reveals access patterns
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER grant payment-cryptography:* and payment-cryptography-data:* wildcard permissions
- NEVER combine key management and cryptographic operation permissions in one role
- NEVER share IAM credentials or access keys
- NEVER disable SCPs to work around access issues

## Phase 3 — Rollback

1. If IAM policies were modified incorrectly: revert to previous policy version
2. If new IAM roles were created: delete if not needed
3. If SCPs were modified: restore original SCP
4. Verify access patterns are working correctly after changes
5. Document all IAM changes for PCI DSS audit

## Common Issues

- symptoms: "AccessDeniedException when calling EncryptData"
  diagnosis: "IAM policy has payment-cryptography permissions but missing payment-cryptography-data permissions."
  resolution: "Add payment-cryptography-data:EncryptData permission. Control plane and data plane are separate services."

- symptoms: "Can manage keys but cannot perform cryptographic operations"
  diagnosis: "IAM role has control plane permissions but not data plane permissions."
  resolution: "Add payment-cryptography-data:* (or specific operations) to the IAM policy."

## Output Format

```yaml
root_cause: "access_control — <specific_cause>"
evidence:
  - type: access_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions"
  long_term: "Implement role separation per PCI DSS"
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
