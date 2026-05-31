---
title: "G1 — Automation Service Role"
description: "Diagnose automation service role (AssumeRole) permission and trust policy issues"
status: active
severity: HIGH
triggers: ["service role error", "AssumeRole denied", "automation permission denied", "role trust policy", "insufficient permissions"]
owner: devops-agent
objective: "Resolve automation service role configuration and permission issues"
context: "The automation service role (AssumeRole parameter) is assumed by SSM to execute automation steps. It needs ssm.amazonaws.com in its trust policy AND permissions for every AWS API call in the document. This is different from the maintenance window service role."
---

## Phase 1 — Triage

MUST:
- Check the execution's AssumeRole: `aws ssm get-automation-execution --automation-execution-id <execution-id> --query 'AutomationExecution.{Role:ResolvedTargets,ExecutedBy:ExecutedBy}'`
- Verify the role exists: `aws iam get-role --role-name <role-name>`
- Check the role trust policy allows ssm.amazonaws.com

SHOULD:
- List role policies: `aws iam list-attached-role-policies --role-name <role-name>`
- Get inline policies: `aws iam list-role-policies --role-name <role-name>`
- Simulate permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names <action> --resource-arns <resource-arn>`

MAY:
- Check CloudTrail for AssumeRole failures
- Compare role permissions against document step requirements
- Review IAM Access Advisor for the role's last accessed services

## Phase 2 — Remediate

MUST:
- Fix trust policy to include ssm.amazonaws.com as trusted principal
- Add permissions for all AWS API calls made by document steps
- Ensure the role ARN in the document matches the actual role

SHOULD:
- Follow least-privilege: grant only permissions needed by the document
- Add condition keys to limit role assumption to specific documents or accounts
- Document which automation documents use which service roles

MAY:
- Create separate roles for different automation documents
- Use IAM Access Analyzer to generate least-privilege policies
- Implement role permission auditing

## Safety Ratings

```
safety_ratings:
  - "iam get-role: GREEN — read-only role metadata"
  - "iam list-attached-role-policies: GREEN — read-only policy listing"
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
  - "iam attach-role-policy: RED — modifies IAM role permissions"
  - "iam put-role-policy: RED — adds inline policy to role"
```

## Escalation Conditions

- Service role requires permissions that violate organizational security policies
- Role is shared across multiple automation documents and changes affect all
- Trust policy modification requires security team approval
- Role permissions cannot be determined from automation failure messages

## Data Sensitivity

- `aws iam get-role`: LOW — returns role metadata and trust policy
- `aws iam list-attached-role-policies`: LOW — returns policy names and ARNs
- `aws iam get-role-policy`: MEDIUM — returns full inline policy document
- `aws iam simulate-principal-policy`: LOW — returns allow/deny simulation results

## Prohibited Actions

- NEVER attach AdministratorAccess or PowerUserAccess to automation service roles
- NEVER modify shared service roles without understanding all automation documents that use them
- NEVER add wildcard resource (*) permissions without justification
- NEVER remove conditions from trust policies that limit role assumption scope

## Phase 3 — Rollback

1. If role permissions were changed incorrectly, detach the new policy: `aws iam detach-role-policy --role-name <name> --policy-arn <arn>`
2. If an inline policy was added, remove it: `aws iam delete-role-policy --role-name <name> --policy-name <policy>`
3. If the trust policy was modified, restore it: `aws iam update-assume-role-policy --role-name <name> --policy-document file://original-trust.json`
4. Verify the role works correctly with existing automation documents after rollback
5. Document the permission change and rollback for security audit

## Common Issues

- symptoms: "Automation fails with AccessDeniedException on a specific step"
  diagnosis: "The service role lacks permissions for the AWS API call in that step."
  resolution: "Add the specific permission to the role. Check the step type and API call to determine required permissions."

- symptoms: "Automation fails immediately with AssumeRole error"
  diagnosis: "The role trust policy does not allow ssm.amazonaws.com to assume it."
  resolution: "Update trust policy: add ssm.amazonaws.com as a trusted principal with sts:AssumeRole action."

- symptoms: "Automation works for some steps but fails on others"
  diagnosis: "The role has partial permissions — some API calls are allowed, others are not."
  resolution: "Review each step's required permissions. Add missing permissions to the role."

## Output Format

```yaml
root_cause: "service_role — <specific_cause>"
evidence:
  - type: iam_error
    content: "<role ARN, trust policy, and permission details>"
severity: HIGH
mitigation:
  immediate: "Fix role permissions"
  long_term: "Implement least-privilege role management"
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
