---
title: "E2 — IAM Permissions"
description: "Diagnose IAM permission issues for Clean Rooms ML service roles and users"
status: active
severity: HIGH
triggers: ["AccessDeniedException", "IAM permission error", "not authorized", "role assumption failed"]
owner: devops-agent
objective: "Resolve IAM permission issues for Clean Rooms ML operations"
context: "Clean Rooms ML requires service roles with permissions for both cleanrooms and cleanroomsml APIs, plus S3 and KMS access. Trust policies must allow both service principals."
---

## Phase 1 — Triage

MUST:
- Check the service role exists and has correct trust policy: `aws iam get-role --role-name <role-name>`
- List attached policies: `aws iam list-attached-role-policies --role-name <role-name>`
- List inline policies: `aws iam list-role-policies --role-name <role-name>`

SHOULD:
- Simulate permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names cleanroomsml:GetTrainingDataset --resource-arns '*'`
- Check for SCP restrictions: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`

MAY:
- Review CloudTrail for AccessDenied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=GetTrainingDataset --max-results 10`
- Check permission boundaries on the role

## Phase 2 — Remediate

MUST:
- Ensure trust policy allows cleanroomsml.amazonaws.com and cleanrooms.amazonaws.com service principals
- Add cleanroomsml:* and cleanrooms:* permissions to the service role (or specific actions needed)
- Add s3:GetObject, s3:PutObject, s3:ListBucket for data buckets
- Add kms:Decrypt, kms:GenerateDataKey, kms:DescribeKey for KMS keys

SHOULD:
- Use least-privilege permissions scoped to specific resource ARNs
- Add condition keys to restrict role usage to specific collaborations

MAY:
- Create a managed policy for Clean Rooms ML service roles
- Enable IAM Access Analyzer to validate permissions

## Common Issues

- symptoms: "AccessDeniedException when calling cleanroomsml API"
  diagnosis: "Service role missing cleanroomsml permissions or trust policy incorrect."
  resolution: "Add cleanroomsml:* to role policy. Update trust policy to include cleanroomsml.amazonaws.com."

- symptoms: "Role assumption fails during audience generation"
  diagnosis: "Trust policy does not include the cleanroomsml.amazonaws.com service principal."
  resolution: "Update trust policy: add cleanroomsml.amazonaws.com to the Principal.Service array."

## Safety Ratings

```
safety_ratings:
  - "iam get-role: GREEN — read-only role details"
  - "iam list-attached-role-policies: GREEN — read-only policy listing"
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
  - "iam put-role-policy: YELLOW — modifies IAM role permissions"
  - "iam update-assume-role-policy: RED — modifies who can assume the role"
```

## Escalation Conditions

- SCP at organization level blocks cleanroomsml actions
- Permission boundary prevents adding required permissions
- Cross-account role assumption required for collaboration
- Security team must approve IAM policy changes

## Data Sensitivity

- `aws iam get-role`: LOW — returns role metadata and trust policy
- `aws iam list-attached-role-policies`: LOW — returns policy names
- `aws iam get-role-policy`: MEDIUM — reveals inline policy details
- `aws cloudtrail lookup-events`: MEDIUM — may reveal principal identities

## Prohibited Actions

- NEVER grant iam:* or admin permissions to service roles
- NEVER modify trust policies to allow arbitrary principals
- NEVER remove condition keys from existing policies without understanding impact
- NEVER share IAM role ARNs or policy details outside the organization

## Phase 3 — Rollback

1. If role policy was changed incorrectly, restore: `aws iam put-role-policy --role-name <role> --policy-name <policy> --policy-document file://previous-policy.json`
2. If trust policy was modified, restore: `aws iam update-assume-role-policy --role-name <role> --policy-document file://previous-trust-policy.json`
3. If managed policy was attached incorrectly, detach: `aws iam detach-role-policy --role-name <role> --policy-arn <policy-arn>`
4. Verify no unauthorized actions were performed during the misconfiguration
5. Document all IAM changes for security audit

## Output Format

```yaml
root_cause: "iam_permissions — <specific_cause>"
evidence:
  - type: iam_error
    content: "<error details from IAM inspection or CloudTrail>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role permissions and trust policy"
  long_term: "Use managed policies and IaC for service role management"
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
