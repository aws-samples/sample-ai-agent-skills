---
title: "H3 — KMS / Encryption Issues"
description: "Diagnose EBS encryption and KMS key access failures"
status: active
severity: HIGH
triggers:
  - "KMS.*AccessDenied"
  - "VolumeInUse.*encrypted"
  - "Client.InternalError.*encrypted"
  - "KMSKeyNotAccessible"
owner: devops-agent
objective: "Restore KMS key access for encrypted EBS volumes"
context: "Encrypted EBS volumes require KMS key access for every I/O operation. If the KMS key is deleted, disabled, or the instance role loses access, the volume becomes unusable. Default EBS encryption uses the aws/ebs key. Custom CMKs require explicit grants."
---

## Phase 1 — Triage

MUST:
- Check volume encryption: `aws ec2 describe-volumes --volume-ids <vol-id>` → Encrypted, KmsKeyId
- Check KMS key state: `aws kms describe-key --key-id <key-id>` — must be Enabled
- Check if instance role has kms:Decrypt and kms:CreateGrant on the key
- Check KMS key policy allows the instance role

SHOULD:
- Check if the key is in a different account (cross-account KMS)
- Verify KMS key grants: `aws kms list-grants --key-id <key-id>`

## Common Issues

- symptoms: "Instance fails to start with encrypted root volume"
  diagnosis: "KMS key disabled, deleted, or instance role lacks permissions."
  resolution: "Re-enable key, or grant kms:Decrypt and kms:CreateGrant to the instance role."

- symptoms: "Client.InternalError on encrypted volume operations"
  diagnosis: "KMS key not accessible. Often a key policy or grant issue."
  resolution: "Check key policy. Add grant for the instance role. Verify key is enabled."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-volumes, describe-key, list-grants: GREEN — read-only"
  - "Re-enable disabled KMS key: YELLOW — restores key access, recoverable by disabling again"
  - "Add KMS grant for instance role: YELLOW — expands key access, recoverable by revoking grant"
  - "Modify KMS key policy: YELLOW — changes key access control, recoverable by reverting policy"
  - "Schedule KMS key deletion: RED — key permanently deleted after waiting period (7-30 days)"
  - "Cancel key deletion: YELLOW — prevents scheduled deletion, recoverable"
```

## Escalation Conditions
- KMS key has been deleted (pending deletion) and volumes are inaccessible
- KMS key is in a different AWS account and cross-account access needs coordination
- Key policy change would affect multiple services or accounts using the same key
- Encrypted volume contains critical data and key access cannot be restored
- KMS key is managed by a security team and policy changes require approval

## Data Sensitivity
- HIGH: describe-key (reveals key policy, key state, key administrators)
- HIGH: list-grants (reveals all principals with access to the key)
- HIGH: describe-volumes KmsKeyId (reveals encryption key associations)
- MEDIUM: CloudTrail KMS events (reveals key usage patterns)

## Prohibited Actions
- NEVER suggest deleting a KMS key to resolve encryption issues
- NEVER suggest creating unencrypted copies of encrypted volumes to bypass KMS issues
- NEVER suggest granting kms:* (all KMS actions) to resolve specific permission issues
- NEVER suggest disabling key rotation as a troubleshooting step

## Phase 3 — Rollback
- If KMS key was re-enabled: disable again with `disable-key` if re-enabling was incorrect
- If KMS grant was added: revoke with `revoke-grant --grant-id <id>`
- If key policy was modified: update key policy to previous version
- If key deletion was scheduled: cancel with `cancel-key-deletion` before the waiting period expires
- If IAM policy was modified for KMS access: revert IAM policy to previous version

## Output Format

```yaml
root_cause: "<key_disabled|key_deleted|missing_permission|cross_account|key_policy>"
evidence:
  - type: kms_key_state
    content: "<describe-key output>"
severity: HIGH
mitigation:
  immediate: "Re-enable key or fix permissions"
  long_term: "Use key policies with least privilege, monitor key state"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
