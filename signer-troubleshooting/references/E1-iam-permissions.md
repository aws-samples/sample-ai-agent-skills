---
title: "E1 — IAM Permissions"
description: "Diagnose IAM permission issues for AWS Signer operations"
status: active
severity: HIGH
triggers: ["AccessDeniedException signer", "IAM permissions signing", "cannot sign"]
owner: devops-agent
objective: "Resolve IAM permission issues blocking Signer operations"
context: "AWS Signer requires specific IAM permissions for profile management, signing jobs, and cross-service integration. S3 bucket policies and KMS key policies may also be required."
---

## Phase 1 — Triage

MUST:
- Verify caller identity: `aws sts get-caller-identity`
- Simulate Signer permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names signer:PutSigningProfile signer:StartSigningJob signer:GetSigningProfile`
- Check for explicit deny in policies: review IAM policies, SCPs, and permission boundaries

SHOULD:
- Verify S3 bucket policy allows Signer service access for signing jobs
- Check KMS key policy if S3 buckets use KMS encryption

MAY:
- Review CloudTrail for AccessDeniedException events
- Check if SCPs at the organization level restrict Signer actions

## Phase 2 — Remediate

MUST:
- Add required Signer permissions to the IAM role/user
- For signing jobs: ensure `signer:StartSigningJob`, `signer:DescribeSigningJob`, `s3:GetObject`, `s3:PutObject`
- For profile management: ensure `signer:PutSigningProfile`, `signer:GetSigningProfile`, `signer:ListSigningProfiles`

SHOULD:
- Add S3 bucket policy for Signer service: allow `signer.amazonaws.com` for GetObject and PutObject
- If using KMS-encrypted S3: add `kms:Decrypt` and `kms:GenerateDataKey` for the Signer service

MAY:
- Create a custom IAM policy with least-privilege Signer permissions
- Use IAM policy conditions to restrict signing to specific profiles

## Common Issues

- symptoms: "StartSigningJob returns AccessDeniedException"
  diagnosis: "IAM role missing signer:StartSigningJob permission or S3 bucket policy blocks Signer."
  resolution: "Add signer:StartSigningJob to IAM policy. Add S3 bucket policy for signer.amazonaws.com."

- symptoms: "PutSigningProfile returns AccessDeniedException"
  diagnosis: "IAM role missing signer:PutSigningProfile permission."
  resolution: "Add signer:PutSigningProfile to the IAM policy."

## Safety Ratings

```
safety_ratings:
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "iam simulate-principal-policy: GREEN — read-only permission test"
  - "iam attach-role-policy: YELLOW — modifies IAM permissions"
  - "s3 put-bucket-policy: YELLOW — modifies S3 access policy"
```

## Escalation Conditions

- SCPs block Signer permissions and cannot be modified
- Permission boundaries prevent adding required Signer actions
- S3 bucket policy changes require approval from a different team
- KMS key policy changes require key administrator access

## Data Sensitivity

- `aws sts get-caller-identity`: LOW — reveals current IAM identity
- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- IAM policy modifications: MEDIUM — changes access control
- S3 bucket policy modifications: MEDIUM — changes data access control

## Prohibited Actions

- NEVER grant `signer:*` permissions — use least-privilege specific actions
- NEVER add broad S3 permissions (s3:*) for Signer access
- NEVER modify KMS key policies without key administrator approval
- NEVER bypass SCPs by using root account credentials

## Phase 3 — Rollback

1. If IAM policy was modified incorrectly, detach or revert the policy
2. If S3 bucket policy was changed, restore previous policy
3. If KMS key policy was modified, revert to previous configuration
4. Verify the original access patterns are restored
5. Document all IAM and policy changes for security audit

## Output Format

```yaml
root_cause: "iam_permissions — <specific_cause>"
evidence:
  - type: permission_error
    content: "<AccessDeniedException details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions"
  long_term: "Create standardized Signer IAM policies"
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
