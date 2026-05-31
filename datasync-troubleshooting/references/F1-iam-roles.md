---
title: "F1 — IAM Role Configuration Issues"
description: "Diagnose IAM role and permission problems for DataSync"
status: active
severity: HIGH
triggers:
  - "IAM permission denied"
  - "role assumption failed"
  - "access denied DataSync"
  - "insufficient permissions"
owner: devops-agent
objective: "Identify and resolve IAM role configuration issues for DataSync"
context: "DataSync uses IAM roles for S3 access and cross-account transfers. The S3 location role must trust the DataSync service principal. The service-linked role (AWSServiceRoleForDataSyncDataAccess) is separate from user-created roles. Cross-account transfers require roles in both accounts with proper trust policies."
---

## Phase 1 — Triage

MUST:
- Check the IAM role used for S3 locations: `aws datasync describe-location-s3 --location-arn <location-arn> --query 'S3Config.BucketAccessRoleArn'`
- Verify role trust policy: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`
- Check role permissions: `aws iam list-attached-role-policies --role-name <role-name>` and `aws iam list-role-policies --role-name <role-name>`
- Verify service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForDataSyncDataAccess`

SHOULD:
- Test permissions with IAM policy simulator: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject s3:PutObject s3:ListBucket`
- Check for SCP restrictions that may block DataSync actions
- Verify KMS key policy allows the DataSync role (for encrypted buckets)

MAY:
- Check CloudTrail for AccessDenied events related to DataSync
- Review cross-account trust policies for cross-account transfers

## Phase 2 — Remediate

MUST:
- Ensure trust policy allows DataSync: `{"Service": "datasync.amazonaws.com"}` in Principal
- Grant minimum required S3 permissions: s3:GetBucketLocation, s3:ListBucket, s3:GetObject, s3:PutObject, s3:DeleteObject, s3:GetObjectTagging, s3:PutObjectTagging
- For KMS: kms:Decrypt, kms:GenerateDataKey on the bucket's KMS key

SHOULD:
- Use least-privilege permissions scoped to specific buckets
- For cross-account: configure trust policy in destination account role to allow source account

MAY:
- Create separate roles for different DataSync tasks/locations
- Use IAM conditions to restrict DataSync role usage

## Common Issues

- symptoms: "AssumeRole failed for DataSync"
  diagnosis: "Trust policy does not include datasync.amazonaws.com as trusted principal."
  resolution: "Add datasync.amazonaws.com to the role's trust policy."

- symptoms: "Access Denied on S3 operations"
  diagnosis: "Role missing required S3 permissions or bucket policy denies access."
  resolution: "Add S3 permissions to role. Check bucket policy for explicit denies."

- symptoms: "Cross-account transfer fails with permission error"
  diagnosis: "Cross-account role trust policy or permissions misconfigured."
  resolution: "Configure trust policy in destination account. Grant permissions in both accounts."

## Output Format

```yaml
root_cause: "iam_role_issue — <specific_cause>"
evidence:
  - type: role_trust
    content: "<trust policy document>"
  - type: role_permissions
    content: "<attached and inline policies>"
  - type: access_test
    content: "<policy simulation or CloudTrail results>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role trust policy and permissions"
  long_term: "Use IaC to manage DataSync IAM roles with least-privilege"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — IAM roles grant DataSync access to potentially sensitive data stores |
| Blast Radius | HIGH — Overly permissive roles create security risks across S3 buckets |
| Rollback Complexity | LOW — Revert IAM policy changes |
| Compliance Scope | CRITICAL — IAM roles must follow least-privilege for compliance |

## Escalation Conditions

- Escalate IMMEDIATELY to security if IAM roles have s3:* or overly broad permissions
- Escalate to security if cross-account trust policies are misconfigured
- Escalate to compliance if IAM roles lack required audit controls

## Data Sensitivity

- **IAM role trust policies** define who can access data — security-critical
- **IAM role permissions** define what data can be accessed — must be least-privilege
- **KMS key policies** grant decryption access — highly sensitive
- **Cross-account trust** enables data access across account boundaries

## PROHIBITED Actions

- Do NOT grant s3:* or kms:* permissions to DataSync roles
- Do NOT use wildcard resource ARNs in IAM policies for DataSync roles
- Do NOT share IAM role ARNs or trust policy details outside security team
- Do NOT create cross-account trust without security team review
- Do NOT reuse DataSync roles across unrelated data transfer workflows

## Phase 3 — Rollback

- Revert IAM role permissions to previous policy version
- Revert trust policy changes if they were modified
- Revert KMS key policy changes if they were modified
- Verify DataSync tasks still function with the reverted permissions

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
