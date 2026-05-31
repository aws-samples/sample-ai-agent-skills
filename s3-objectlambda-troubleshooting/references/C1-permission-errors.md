---
title: "C1 — Permission Errors"
description: "Diagnose permission errors when accessing Object Lambda access points"
status: active
severity: HIGH
triggers: ["access denied", "403 error", "permission denied", "unauthorized OLAP access"]
owner: devops-agent
objective: "Resolve permission errors for S3 Object Lambda access"
context: "OLAP access requires permissions at three layers: OLAP policy, supporting access point policy, and IAM identity policy. Lambda execution role also needs specific permissions."
---

## Phase 1 — Triage

MUST:
- Check OLAP policy: `aws s3control get-access-point-policy-for-object-lambda --account-id <account-id> --name <olap-name>`
- Check supporting access point policy: `aws s3control get-access-point-policy --account-id <account-id> --name <supporting-ap-name>`
- Verify caller's IAM permissions include s3-object-lambda:GetObject on the OLAP ARN

SHOULD:
- Check Lambda execution role permissions: `aws iam get-role-policy --role-name <lambda-role> --policy-name <policy-name>`
- Verify Lambda role has s3-object-lambda:WriteGetObjectResponse permission
- Check bucket policy for Lambda role's s3:GetObject access

MAY:
- Use IAM Policy Simulator to test permissions: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names s3-object-lambda:GetObject --resource-arns <olap-arn>`
- Review CloudTrail for AccessDenied events

## Phase 2 — Remediate

MUST:
- Grant caller s3-object-lambda:GetObject on the OLAP ARN in IAM policy
- Ensure OLAP policy allows the caller principal
- Ensure Lambda execution role has s3-object-lambda:WriteGetObjectResponse and s3:GetObject permissions

SHOULD:
- Align supporting access point policy with OLAP policy
- Use least-privilege principles — scope permissions to specific OLAP ARNs

MAY:
- Implement resource-based policies on the OLAP for cross-account access
- Set up AWS Config rules to monitor OLAP policy compliance

## Common Issues

- symptoms: "Caller gets 403 AccessDenied when accessing OLAP"
  diagnosis: "Caller IAM policy missing s3-object-lambda:GetObject permission."
  resolution: "Add s3-object-lambda:GetObject permission for the OLAP ARN to the caller's IAM policy."

- symptoms: "Lambda function gets AccessDenied calling WriteGetObjectResponse"
  diagnosis: "Lambda execution role missing s3-object-lambda:WriteGetObjectResponse permission."
  resolution: "Add s3-object-lambda:WriteGetObjectResponse to the Lambda execution role."

## Safety Ratings

```
safety_ratings:
  - "get-access-point-policy-for-object-lambda: GREEN — read-only policy retrieval"
  - "get-access-point-policy: GREEN — read-only policy retrieval"
  - "iam get-role-policy: GREEN — read-only IAM inspection"
  - "iam simulate-principal-policy: GREEN — simulates without making changes"
  - "put-access-point-policy-for-object-lambda: RED — modifies access control"
```

## Escalation Conditions

- Permission errors persist after fixing all three policy layers
- Cross-account access requires organizational SCP changes
- VPC endpoint policy blocks OLAP access
- Permission changes require security team approval in regulated environments

## Data Sensitivity

- `aws s3control get-access-point-policy-for-object-lambda`: MEDIUM — reveals access control configuration
- `aws iam get-role-policy`: MEDIUM — reveals IAM permissions
- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- `aws cloudtrail lookup-events`: MEDIUM — reveals principal identities and actions

## Prohibited Actions

- NEVER grant s3-object-lambda:* to * (wildcard principal with wildcard actions)
- NEVER remove existing OLAP policies without understanding current access patterns
- NEVER modify Lambda execution roles shared by other functions without impact analysis
- NEVER bypass permission errors by making the supporting access point public

## Phase 3 — Rollback

1. If OLAP policy was modified incorrectly, restore previous policy: `aws s3control put-access-point-policy-for-object-lambda --account-id <account-id> --name <olap-name> --policy file://previous-policy.json`
2. If supporting access point policy was changed, restore it: `aws s3control put-access-point-policy --account-id <account-id> --name <ap-name> --policy file://previous-policy.json`
3. If Lambda execution role was modified, revert IAM policy changes
4. Verify access patterns return to normal after rollback
5. Document the permission change attempt and why it failed

## Output Format

```yaml
root_cause: "permission_error — <specific_cause>"
evidence:
  - type: access_denied
    content: "<error details and policy analysis>"
severity: HIGH
mitigation:
  immediate: "Fix permission policies"
  long_term: "Implement policy-as-code with automated validation"
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
