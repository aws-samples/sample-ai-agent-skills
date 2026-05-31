---
title: "G2 — Encryption"
description: "Diagnose CloudTrail log encryption issues"
status: active
severity: HIGH
triggers:
  - "encryption error"
  - "KMS key error"
  - "cannot decrypt logs"
  - "SSE-KMS"
owner: devops-agent
objective: "Fix CloudTrail log encryption configuration"
context: "CloudTrail logs are encrypted with SSE-S3 by default. SSE-KMS provides additional control with customer-managed keys. KMS key policy must allow CloudTrail to encrypt and users to decrypt. Key policy issues cause delivery failures or inability to read logs."
---

## Phase 1 — Triage

MUST:
- Check trail encryption: `aws cloudtrail get-trail --name <trail-name> --query 'Trail.KmsKeyId'`
- Check KMS key status: `aws kms describe-key --key-id <key-id>`
- Check KMS key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Check trail delivery status for KMS errors: `aws cloudtrail get-trail-status --name <trail-name>`

SHOULD:
- Verify CloudTrail service principal in key policy
- Check if key is enabled and not pending deletion
- Verify users have kms:Decrypt for reading logs

MAY:
- Check key rotation status
- Review key grants

## Phase 2 — Remediate

MUST:
- Add CloudTrail to KMS key policy: allow kms:GenerateDataKey* for cloudtrail.amazonaws.com
- Add log readers to key policy: allow kms:Decrypt
- Ensure key is enabled

SHOULD:
- Enable automatic key rotation
- Update trail with correct KMS key: `aws cloudtrail update-trail --name <trail-name> --kms-key-id <key-arn>`

MAY:
- Create dedicated KMS key for CloudTrail
- Set up key policy with least-privilege access

## Common Issues

- symptoms: "Log delivery fails with KMS error"
  diagnosis: "KMS key policy doesn't allow CloudTrail to encrypt."
  resolution: "Add GenerateDataKey permission for CloudTrail service principal."

- symptoms: "Cannot read CloudTrail logs"
  diagnosis: "User lacks kms:Decrypt permission on the KMS key."
  resolution: "Add Decrypt permission for the user/role in key policy."

- symptoms: "KMS key disabled or pending deletion"
  diagnosis: "Key was disabled or scheduled for deletion."
  resolution: "Re-enable key or cancel deletion. Update trail if key is lost."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add CloudTrail to KMS key policy | YELLOW | Key policy changes affect all key usage |
| Add log readers to key policy | YELLOW | Grants decrypt access to audit logs |
| Enable automatic key rotation | GREEN | Security enhancement, no disruption |
| Update trail with KMS key | YELLOW | Changes encryption for new log files |

## Escalation Conditions

- KMS key disabled or pending deletion — escalate immediately to key administrator
- Log readers unable to decrypt CloudTrail logs — escalate to security operations
- KMS key policy changes — escalate to key administrator and security team
- Encryption configuration changes on compliance-critical trails — escalate to compliance team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws kms get-key-policy` | HIGH | Exposes key access policy with principals |
| `aws kms describe-key` | MEDIUM | Exposes key status and metadata |
| `aws cloudtrail get-trail` (KMS config) | HIGH | Exposes encryption configuration |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** disable or schedule deletion of KMS keys used for CloudTrail encryption
- **NEVER** grant overly broad decrypt permissions on CloudTrail encryption keys

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Updated KMS key policy | Restore previous key policy |
| Updated trail KMS key | Revert to previous KMS key or SSE-S3: `aws cloudtrail update-trail` |
| Enabled key rotation | Disable rotation (not recommended): `aws kms disable-key-rotation` |

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: key_status
    content: "<KMS key status>"
  - type: key_policy
    content: "<key policy summary>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key policy"
  long_term: "Use dedicated key with proper policy and rotation"
```

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
  - command: "lookup-events"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-trail-status"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-trails"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling CloudTrail logging"
  - "NEVER suggest deleting trail logs"
  - "NEVER reduce logging scope in production"
