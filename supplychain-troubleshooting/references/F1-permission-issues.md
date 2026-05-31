---
title: "F1 — Permission Issues"
description: "Diagnose IAM and access control issues for Supply Chain"
status: active
severity: HIGH
triggers: ["access denied", "permission error", "IAM policy", "cannot access instance"]
owner: devops-agent
objective: "Resolve access control and IAM permission issues for Supply Chain"
context: "Supply Chain uses IAM for access control. Permissions can be scoped to specific instances using resource ARNs."
---

## Phase 1 — Triage

MUST:
- Check caller identity: `aws sts get-caller-identity`
- Review IAM policies: `aws iam list-attached-role-policies --role-name <role>`
- Test specific permission: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names scn:GetInstance --resource-arns <instance-arn>`

SHOULD:
- Check for SCPs that may deny Supply Chain actions
- Verify service-linked role exists for Supply Chain

MAY:
- Review CloudTrail for AccessDenied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=scn.amazonaws.com --max-results 20`

## Phase 2 — Remediate

MUST:
- Add required scn permissions to IAM policy
- Scope permissions to specific instance ARNs for least privilege
- Fix S3 and KMS permissions for data integration

SHOULD:
- Use IAM policy conditions for additional security
- Implement separate roles for admin and read-only access

MAY:
- Set up AWS Config rules to monitor Supply Chain IAM policies
- Implement ABAC using tags

## Safety Ratings

```
safety_ratings:
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "iam simulate-principal-policy: GREEN — read-only permission test"
  - "IAM policy modification: YELLOW — changes access permissions"
  - "SCP modification: RED — affects entire organization"
```

## Escalation Conditions

- Access denied errors persist after IAM policy fixes
- SCP restrictions require organizational policy changes
- Cross-account access needed for shared instances
- Service-linked role creation requires admin permissions

## Data Sensitivity

- `aws sts get-caller-identity`: LOW — returns caller metadata
- `aws iam list-attached-role-policies`: LOW — returns policy names
- `aws iam simulate-principal-policy`: LOW — tests permissions without data access
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER grant scn:* wildcard permissions in production
- NEVER modify IAM policies without understanding the blast radius
- NEVER share IAM credentials or access keys
- NEVER disable SCPs to work around access issues

## Phase 3 — Rollback

1. If IAM policies were modified incorrectly: revert to previous policy version
2. If new IAM roles were created: delete if not needed
3. If SCPs were modified: restore original SCP
4. Verify access patterns are working correctly after changes
5. Document all IAM changes for security audit

## Common Issues

- symptoms: "AccessDeniedException when calling GetInstance"
  diagnosis: "IAM policy missing scn:GetInstance permission or wrong resource ARN."
  resolution: "Add scn:GetInstance permission scoped to the specific instance ARN."

- symptoms: "Data integration flow fails with access denied on S3"
  diagnosis: "Flow execution role lacks S3 permissions."
  resolution: "Add s3:GetObject and s3:ListBucket to the flow execution role for the source bucket."

## Output Format

```yaml
root_cause: "permission — <specific_cause>"
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
