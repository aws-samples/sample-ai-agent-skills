---
title: "C2 — Cross-Account KMS for S3"
description: "Diagnose cross-account KMS key access issues for S3 encryption"
status: active
severity: HIGH
triggers:
  - "Cross-account KMS"
  - "KMS key other account"
  - "Cannot decrypt cross-account"
  - "KMS key alias cross-account"
owner: devops-agent
objective: "Configure cross-account KMS key access for S3 encryption operations"
context: "Cross-account KMS access requires the key policy to explicitly allow the external account or principal. KMS key aliases do NOT work cross-account — you must use the key ARN. The encryption context for S3 is aws:s3:arn with the bucket ARN."
---

## Phase 1 — Triage

MUST:
- Identify the KMS key ARN (not alias): `aws s3api head-object --bucket <bucket> --key <key> --query 'SSEKMSKeyId'`
- Check the KMS key policy for cross-account grants: `aws kms get-key-policy --key-id <key-arn> --policy-name default`
- Verify the requesting principal's IAM policy includes KMS permissions on the key ARN
- Confirm the key ARN is used (not alias) in all cross-account references

SHOULD:
- Check for KMS grants that allow cross-account access: `aws kms list-grants --key-id <key-arn>`
- Verify the encryption context matches what S3 expects
- Check if the key is a customer-managed key (AWS managed keys cannot be shared cross-account)

MAY:
- Check CloudTrail in the key-owning account for denied KMS requests
- Verify SCPs allow KMS actions in both accounts

## Phase 2 — Remediate

MUST:
- Update the KMS key policy to allow the external account/principal:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::<external-account>:root"},
    "Action": ["kms:Decrypt", "kms:GenerateDataKey", "kms:DescribeKey"],
    "Resource": "*"
  }
  ```
- Add KMS permissions to the requesting principal's IAM policy using the full key ARN
- Use key ARN, not alias, in all cross-account configurations

SHOULD:
- Scope the key policy to specific principals rather than the entire account
- Add conditions to restrict usage to S3 context: `"kms:ViaService": "s3.<region>.amazonaws.com"`
- Document which external accounts have access to the key

MAY:
- Use KMS grants for more granular, revocable cross-account access
- Consider creating a KMS key in each account to avoid cross-account complexity

## Common Issues

- symptoms: "AccessDenied when using KMS alias cross-account"
  diagnosis: "KMS aliases are account-scoped and cannot be resolved cross-account."
  resolution: "Use the full KMS key ARN (arn:aws:kms:region:account:key/key-id) instead of alias."

- symptoms: "Cannot use AWS managed key (aws/s3) cross-account"
  diagnosis: "AWS managed keys cannot be shared cross-account."
  resolution: "Create a customer-managed KMS key and share it via key policy."

- symptoms: "Decrypt works but GenerateDataKey fails cross-account"
  diagnosis: "Key policy only grants kms:Decrypt but not kms:GenerateDataKey."
  resolution: "Add kms:GenerateDataKey to the key policy for the external principal."

## Output Format

```yaml
root_cause: "cross_account_kms — <specific_cause>"
evidence:
  - type: kms_key_policy
    content: "<key policy statements>"
  - type: iam_policy
    content: "<KMS permissions in requesting account>"
severity: HIGH
mitigation:
  immediate: "Update KMS key policy and IAM policy for cross-account access"
  long_term: "Use key ARNs consistently and document cross-account key sharing"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying KMS key policies to grant cross-account access. Incorrect key policy changes can expose encryption keys to unintended accounts or break decryption for existing consumers. |

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
