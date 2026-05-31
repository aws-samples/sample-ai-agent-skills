---
title: "C1 — SSE-KMS Encryption Errors"
description: "Diagnose SSE-KMS encryption and decryption failures on S3 objects"
status: active
severity: HIGH
triggers:
  - "KMS access denied"
  - "SSE-KMS error"
  - "kms:Decrypt denied"
  - "kms:GenerateDataKey denied"
  - "KMS key disabled"
owner: devops-agent
objective: "Identify and fix KMS key permission or state issues blocking S3 encryption operations"
context: "SSE-KMS requires BOTH S3 permissions AND KMS key permissions. Reading requires kms:Decrypt. Writing requires kms:GenerateDataKey. The KMS key policy, IAM policy, and grants all factor into access. Key state (enabled, disabled, pending deletion) also affects access."
---

## Phase 1 — Triage

MUST:
- Identify the KMS key used: `aws s3api head-object --bucket <bucket> --key <key> --query 'SSEKMSKeyId'`
- Check the KMS key state: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.KeyState'`
- Check the KMS key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Verify the principal has kms:Decrypt (for reads) or kms:GenerateDataKey (for writes)
- Check bucket default encryption: `aws s3api get-bucket-encryption --bucket <bucket>`

SHOULD:
- Check if the key is in a different account (cross-account KMS — see C2)
- Verify IAM policy includes KMS permissions on the specific key ARN
- Check for KMS grants: `aws kms list-grants --key-id <key-id>`

MAY:
- Check CloudTrail for KMS denied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com`
- Verify the encryption context if the key policy requires it

## Phase 2 — Remediate

MUST:
- For read access: add kms:Decrypt to the principal's IAM policy for the key ARN
- For write access: add kms:GenerateDataKey to the principal's IAM policy for the key ARN
- Ensure the KMS key policy allows the principal (key policy + IAM policy must both allow)
- If key is disabled: re-enable it: `aws kms enable-key --key-id <key-id>`

SHOULD:
- Use key ARN (not alias) in IAM policies for cross-account scenarios
- Add kms:DescribeKey permission alongside Decrypt/GenerateDataKey
- If key is pending deletion: cancel deletion if within the waiting period: `aws kms cancel-key-deletion --key-id <key-id>`

MAY:
- Use KMS grants for temporary or service-linked access
- Consider switching to SSE-S3 if KMS key management is too complex

## Common Issues

- symptoms: "AccessDenied on GetObject for KMS-encrypted objects"
  diagnosis: "Principal lacks kms:Decrypt on the KMS key."
  resolution: "Add kms:Decrypt to the IAM policy and ensure the KMS key policy allows the principal."

- symptoms: "PutObject fails with KMS error"
  diagnosis: "Principal lacks kms:GenerateDataKey on the bucket's default encryption key."
  resolution: "Add kms:GenerateDataKey to the IAM policy for the key ARN."

- symptoms: "KMS key suddenly stopped working"
  diagnosis: "Key was disabled or scheduled for deletion."
  resolution: "Check key state with describe-key. Re-enable or cancel deletion."

## Output Format

```yaml
root_cause: "sse_kms_error — <specific_cause>"
evidence:
  - type: kms_key_state
    content: "<key state and policy>"
  - type: iam_policy
    content: "<KMS permissions>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key permissions or key state"
  long_term: "Include KMS permissions in standard IAM policy templates"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟢 GREEN | Primarily diagnostic — uses head-object, describe-key, get-key-policy, get-bucket-encryption. Remediation adds KMS permissions to IAM policies, not direct bucket security changes. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Bucket contains sensitive/regulated data (PII, PHI, financial)
- Cross-account access changes are needed
- Encryption configuration changes affect multiple consumers

## Rollback
1. Before any bucket policy change: Save current policy with `aws s3api get-bucket-policy`
2. Before ACL changes: Save current ACL with `aws s3api get-bucket-acl`
3. After change: Verify access works without granting excessive permissions
4. If change causes issues: Restore the saved policy/ACL immediately
5. Cleanup: Remove any temporary access grants

## Data Sensitivity
| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-bucket-policy` | HIGH | Contains access rules — redact principals |
| `get-bucket-acl` | MEDIUM | Shows grantees — summarize |
| `get-public-access-block` | MEDIUM | Security posture — safe to include |
| `list-objects` | LOW | Object keys only — safe to include |

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` in bucket policy without Condition keys
- NEVER suggest removing bucket encryption to fix access issues
- NEVER suggest making a bucket public to resolve CORS or access issues
- NEVER suggest `s3:*` in any IAM or bucket policy fix
- ALWAYS use least-privilege: grant only the specific S3 action needed
- ALWAYS check both account-level AND bucket-level Block Public Access

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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
