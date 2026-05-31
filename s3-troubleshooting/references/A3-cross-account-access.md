---
title: "A3 — Cross-Account S3 Access"
description: "Diagnose cross-account access failures requiring both IAM and bucket policy"
status: active
severity: HIGH
triggers:
  - "Cross-account access denied"
  - "Other account cannot access bucket"
  - "Cross-account S3"
owner: devops-agent
objective: "Ensure both sides of cross-account access are configured correctly"
context: "Cross-account S3 access requires BOTH an IAM policy in the requesting account AND a bucket policy in the bucket-owning account. Missing either side causes AccessDenied. Object ownership also matters — with BucketOwnerEnforced, the bucket owner always owns objects regardless of who uploaded them."
---

## Phase 1 — Triage

MUST:
- Identify the requesting account and the bucket-owning account
- Check the bucket policy for cross-account Allow: `aws s3api get-bucket-policy --bucket <bucket>`
- Verify the requesting principal has IAM permissions for the S3 actions
- Check object ownership setting: `aws s3api get-bucket-ownership-controls --bucket <bucket>`

SHOULD:
- Verify the principal ARN format in the bucket policy matches the actual requester
- Check for Deny statements that might block cross-account access
- Check if the bucket uses SSE-KMS (requires cross-account KMS key access too)
- Simulate from the requesting account: `aws iam simulate-principal-policy --policy-source-arn <requester-arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::<bucket>/*`

MAY:
- Check if an S3 Access Point would simplify the cross-account setup
- Verify SCPs in both accounts allow the S3 actions

## Phase 2 — Remediate

MUST:
- Add bucket policy allowing the requesting account's principal:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::<requester-account>:role/<role>"},
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::<bucket>/*"
  }
  ```
- Ensure the requesting principal's IAM policy allows the S3 actions on the bucket ARN
- Set object ownership to BucketOwnerEnforced to avoid ownership issues on uploads

SHOULD:
- Use specific principal ARNs instead of account-wide access
- If SSE-KMS is used, configure cross-account KMS key access (see C2)
- Add `--acl bucket-owner-full-control` for uploads if not using BucketOwnerEnforced

MAY:
- Use S3 Access Points for cleaner cross-account access management
- Use aws:PrincipalOrgID condition to allow all accounts in the Organization

## Common Issues

- symptoms: "Bucket policy allows the account but access is still denied"
  diagnosis: "The requesting principal lacks IAM permissions for S3 actions."
  resolution: "Add S3 permissions to the IAM role/user in the requesting account."

- symptoms: "PutObject succeeds but bucket owner cannot read the object"
  diagnosis: "Object ownership is ObjectWriter and the uploader owns the object."
  resolution: "Set bucket ownership to BucketOwnerEnforced. This disables ACLs and the bucket owner always owns objects."

- symptoms: "Access works for some objects but not others"
  diagnosis: "Objects uploaded before BucketOwnerEnforced was set are still owned by the uploader."
  resolution: "Copy the objects in place to update ownership, or use S3 Batch Operations."

## Output Format

```yaml
root_cause: "cross_account_access — <missing_side>: <detail>"
evidence:
  - type: bucket_policy
    content: "<cross-account statement or lack thereof>"
  - type: iam_policy
    content: "<requester IAM policy>"
severity: HIGH
mitigation:
  immediate: "Configure both IAM policy and bucket policy"
  long_term: "Use BucketOwnerEnforced and consider S3 Access Points"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying bucket policies to grant cross-account access and changing ownership controls. Incorrect configuration can expose data to unintended accounts. Uses put-bucket-policy with cross-account principals. |

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
