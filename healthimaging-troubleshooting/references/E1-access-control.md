---
title: "E1 — Access Control"
description: "Diagnose IAM and access control issues for HealthImaging"
status: active
severity: HIGH
triggers: ["access denied", "permission error", "IAM policy", "cannot access datastore"]
owner: devops-agent
objective: "Resolve access control and IAM permission issues for HealthImaging"
context: "HealthImaging uses IAM for access control. Permissions can be scoped to specific datastores using resource ARNs."
---

## Phase 1 — Triage

MUST:
- Check caller identity: `aws sts get-caller-identity`
- Review IAM policies attached to the principal: `aws iam list-attached-user-policies --user-name <user>` or `aws iam list-attached-role-policies --role-name <role>`
- Test specific permission: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names medical-imaging:GetImageSet --resource-arns <datastore-arn>`

SHOULD:
- Check for SCPs that may deny HealthImaging actions
- Verify resource-based policies on KMS keys used by datastores

MAY:
- Review CloudTrail for AccessDenied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=medical-imaging.amazonaws.com --max-results 20`

## Phase 2 — Remediate

MUST:
- Add required medical-imaging permissions to IAM policy
- Scope permissions to specific datastore ARNs for least privilege
- Fix KMS key policies if encryption-related access errors

SHOULD:
- Use IAM policy conditions for additional security (e.g., source IP, MFA)
- Implement separate roles for read-only and read-write access

MAY:
- Set up AWS Config rules to monitor HealthImaging IAM policies
- Implement attribute-based access control (ABAC) using tags

## Safety Ratings

```
safety_ratings:
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "iam simulate-principal-policy: GREEN — read-only permission test"
  - "IAM policy modification: YELLOW — changes access permissions"
  - "KMS key policy modification: RED — changes encryption access"
```

## Escalation Conditions

- Access denied errors persist after IAM policy fixes
- SCP restrictions require organizational policy changes
- Cross-account access needed for shared datastores
- Compliance requires specific access control patterns not achievable with current IAM

## Data Sensitivity

- `aws sts get-caller-identity`: LOW — returns caller metadata
- `aws iam list-attached-role-policies`: LOW — returns policy names
- `aws iam simulate-principal-policy`: LOW — tests permissions without data access
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER grant medical-imaging:* wildcard permissions in production
- NEVER modify IAM policies without understanding the blast radius
- NEVER share IAM credentials or access keys
- NEVER disable KMS key policies to work around access issues

## Phase 3 — Rollback

1. If IAM policies were modified incorrectly: revert to previous policy version
2. If KMS key policies were changed: restore original key policy
3. If new IAM roles were created: delete if not needed, or restrict permissions
4. Verify access patterns are working correctly after changes
5. Document all IAM changes for security audit

## Common Issues

- symptoms: "AccessDeniedException when calling GetImageSet"
  diagnosis: "IAM policy missing medical-imaging:GetImageSet permission or wrong resource ARN."
  resolution: "Add medical-imaging:GetImageSet permission scoped to the specific datastore ARN."

- symptoms: "KMS access denied when retrieving image frames"
  diagnosis: "IAM principal lacks kms:Decrypt permission on the datastore's KMS key."
  resolution: "Add kms:Decrypt permission for the KMS key used by the datastore."

## Output Format

```yaml
root_cause: "access_control — <specific_cause>"
evidence:
  - type: access_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions"
  long_term: "Implement least-privilege access patterns"
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
