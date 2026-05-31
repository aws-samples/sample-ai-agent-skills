---
title: "E2 — IAM Permissions"
description: "Diagnose SSM Distributor IAM permission issues"
status: active
severity: HIGH
triggers: ["access denied", "iam permission error", "instance profile", "ssm permission", "unauthorized"]
owner: devops-agent
objective: "Resolve IAM permission issues affecting Distributor operations"
context: "Distributor requires IAM permissions for SSM API calls, S3 artifact access, and instance profile configuration."
---

## Phase 1 — Triage

MUST:
- Check instance profile role: `aws ssm describe-instance-information --instance-information-filter-list key=InstanceIds,valueSet=<instance-id> --query 'InstanceInformationList[0].IamRole'`
- Verify role policies: `aws iam list-attached-role-policies --role-name <role-name>`
- Check for AmazonSSMManagedInstanceCore policy attachment

SHOULD:
- Simulate SSM permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names ssm:SendCommand ssm:GetDocument s3:GetObject`
- Check for SCP restrictions: `aws organizations list-policies-for-target --target-id <account-id> --filter SERVICE_CONTROL_POLICY`

MAY:
- Review CloudTrail for AccessDenied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=SendCommand --max-results 10`
- Check resource-based policies on SSM documents

## Phase 2 — Remediate

MUST:
- Attach AmazonSSMManagedInstanceCore managed policy to the instance profile role
- Add s3:GetObject permission for the package artifact bucket
- Ensure the caller has ssm:SendCommand and ssm:CreateAssociation permissions

SHOULD:
- Use least-privilege IAM policies scoped to specific resources
- Add ssm:GetDocument permission for the specific package documents

MAY:
- Create a custom IAM policy for Distributor-specific permissions
- Implement permission boundaries for additional security

## Common Issues

- symptoms: "SendCommand returns AccessDeniedException"
  diagnosis: "Caller IAM role lacks ssm:SendCommand permission."
  resolution: "Add ssm:SendCommand permission to the caller's IAM role. Scope to specific document names if possible."

- symptoms: "Instance cannot execute commands — SSM Agent reports permission error"
  diagnosis: "Instance profile role lacks AmazonSSMManagedInstanceCore policy."
  resolution: "Attach AmazonSSMManagedInstanceCore managed policy to the instance profile role."

## Safety Ratings

```
safety_ratings:
  - "describe-instance-information: GREEN — read-only instance metadata"
  - "iam list-attached-role-policies: GREEN — read-only policy listing"
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
  - "iam attach-role-policy: RED — modifies IAM permissions"
```

## Escalation Conditions

- IAM permissions are managed by a central security team with change control processes
- SCP restrictions block required SSM permissions at the organization level
- Permission boundary prevents adding required permissions to the role
- Cross-account Distributor operations require trust relationship changes

## Data Sensitivity

- `aws iam list-attached-role-policies`: MEDIUM — reveals IAM configuration
- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principals
- IAM policy documents: MEDIUM — reveal access control configuration

## Prohibited Actions

- NEVER attach AdministratorAccess or PowerUserAccess to resolve permission issues
- NEVER use wildcard (*) actions in IAM policies for SSM
- NEVER modify SCPs without organization administrator approval
- NEVER share IAM credentials or access keys to work around permission issues

## Phase 3 — Rollback

1. If IAM policies were broadened, revert to the original: `aws iam detach-role-policy --role-name <role> --policy-arn <broad-policy-arn>`
2. Re-attach the original policy: `aws iam attach-role-policy --role-name <role> --policy-arn <original-policy-arn>`
3. If inline policies were added, remove them: `aws iam delete-role-policy --role-name <role> --policy-name <policy-name>`
4. Verify SSM operations still work with the corrected permissions
5. Document the minimum required permissions for the Distributor use case

## Output Format

```yaml
root_cause: "iam_permissions — <specific_cause>"
evidence:
  - type: permission_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions"
  long_term: "Implement least-privilege IAM policies"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
