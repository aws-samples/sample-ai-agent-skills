---
title: "F2 — IAM Role Issues"
description: "Diagnose IAM role configuration issues for CodeCatalyst"
status: active
severity: HIGH
triggers: ["IAM role error", "AssumeRole failed", "permission denied", "trust policy error"]
owner: devops-agent
objective: "Resolve IAM role configuration issues for CodeCatalyst workflows and connections"
context: "CodeCatalyst requires IAM roles with specific trust policies allowing codecatalyst.amazonaws.com service principal."
---

## Phase 1 — Triage

MUST:
- Check IAM role exists: `aws iam get-role --role-name <codecatalyst-role>`
- Verify trust policy: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'`
- Check role permissions: `aws iam list-attached-role-policies --role-name <role>`

SHOULD:
- Review CloudTrail for AssumeRole failures: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole --max-results 20`
- Check for SCPs blocking role assumption
- Verify inline policies: `aws iam list-role-policies --role-name <role>`

MAY:
- Simulate role permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names <actions>`
- Check permission boundaries on the role

## Phase 2 — Remediate

MUST:
- Fix trust policy to include codecatalyst.amazonaws.com as trusted principal
- Add required permissions for workflow actions
- Ensure external ID or condition keys match CodeCatalyst configuration

SHOULD:
- Follow least-privilege principle for role permissions
- Add condition keys to trust policy (sts:ExternalId, aws:SourceArn)
- Document role purpose and permissions

MAY:
- Create separate roles for different workflow stages (build, test, deploy)
- Implement role permission monitoring with Access Analyzer

## Safety Ratings

```
safety_ratings:
  - "iam get-role: GREEN — read-only role details"
  - "iam list-attached-role-policies: GREEN — read-only policy listing"
  - "iam update-assume-role-policy: YELLOW — modifies trust relationship"
  - "iam attach-role-policy: YELLOW — grants additional permissions"
```

## Escalation Conditions

- IAM role modifications blocked by organizational SCPs
- Trust policy changes require security team approval
- Role permissions insufficient but cannot be expanded due to policy
- Multiple roles need simultaneous updates across accounts

## Data Sensitivity

- `aws iam get-role`: MEDIUM — reveals role configuration and trust relationships
- `aws iam list-attached-role-policies`: MEDIUM — reveals granted permissions
- Trust policy document: MEDIUM — reveals cross-service trust relationships
- CloudTrail AssumeRole events: MEDIUM — reveals access patterns

## Prohibited Actions

- NEVER grant AdministratorAccess or PowerUserAccess to CodeCatalyst roles
- NEVER use wildcard (*) for all actions in role policies
- NEVER remove condition keys from trust policies without security review
- NEVER share IAM role credentials or session tokens

## Phase 3 — Rollback

1. If trust policy was modified, restore original trust policy document
2. If permissions were added, remove the added policies
3. Verify CodeCatalyst workflows still function with restored configuration
4. Confirm no unauthorized access occurred through the modified role
5. Document all IAM changes and notify the security team

## Common Issues

- symptoms: "AssumeRole fails with 'is not authorized to perform sts:AssumeRole'"
  diagnosis: "Trust policy does not include codecatalyst.amazonaws.com."
  resolution: "Update trust policy: add codecatalyst.amazonaws.com as Principal Service."

- symptoms: "Role assumption succeeds but actions fail with AccessDenied"
  diagnosis: "Role lacks specific permissions for the AWS services used in workflows."
  resolution: "Add required service permissions to the role policy. Use IAM policy simulator to test."

## Output Format

```yaml
root_cause: "iam_role — <specific_cause>"
evidence:
  - type: role_configuration
    content: "<role details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role configuration"
  long_term: "Implement role monitoring and least-privilege reviews"
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
