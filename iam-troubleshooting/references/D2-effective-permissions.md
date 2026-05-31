---
title: "D2 — Calculating Effective Permissions"
description: "Calculate effective permissions when permission boundaries are in use"
status: active
severity: MEDIUM
triggers:
  - "effective permissions"
  - "what can this user do"
  - "actual permissions"
  - "permission intersection"
owner: devops-agent
objective: "Determine the actual effective permissions of a principal considering all policy layers"
context: "Effective permissions are the intersection of all applicable policy types. With a permission boundary: effective = identity policy ∩ boundary. With SCPs: effective = identity policy ∩ boundary ∩ SCP. Resource-based policies can grant access independently for same-account."
---

## Phase 1 — Triage

MUST:
- Get all identity policies:
  - `aws iam list-attached-role-policies --role-name <role>`
  - `aws iam list-role-policies --role-name <role>`
  - `aws iam get-policy-version --policy-arn <arn> --version-id <version>` (for each)
- Check for permission boundary: `aws iam get-role --role-name <role>` (PermissionsBoundary field)
- If boundary exists, get its policy document
- Simulate key actions: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names <action1> <action2> <action3>`

SHOULD:
- Check SCPs if in an Organization: `aws organizations list-policies-for-target --target-id <account-id> --filter SERVICE_CONTROL_POLICY`
- Use Access Advisor for actual usage: `aws iam generate-service-last-accessed-details --arn <arn>` then `aws iam get-service-last-accessed-details --job-id <id>`
- Check group memberships for additional policies: `aws iam list-groups-for-user --user-name <user>`

MAY:
- Use IAM Access Analyzer policy generation: `aws accessanalyzer start-policy-generation --policy-generation-details '{"principalArn":"<arn>"}'`
- Generate a comprehensive permissions report

## Phase 2 — Remediate

MUST:
- Calculate: effective = identity_policies ∩ permission_boundary ∩ SCPs
- For each action, verify it appears in ALL applicable policy layers
- Document the effective permissions for the principal

SHOULD:
- Use simulate-principal-policy to verify specific actions rather than manual calculation
- Create a least-privilege policy based on actual usage from Access Advisor
- Remove unused permissions identified by Access Advisor

MAY:
- Automate effective permission calculation in CI/CD
- Use IAM Access Analyzer to generate least-privilege policies

## Common Issues

- symptoms: "User has AdministratorAccess but cannot perform certain actions"
  diagnosis: "A permission boundary or SCP is restricting the effective permissions."
  resolution: "Check boundary and SCPs. AdministratorAccess is still subject to boundary and SCP restrictions."

- symptoms: "simulate-principal-policy shows allowed but actual call is denied"
  diagnosis: "The simulator doesn't evaluate SCPs, resource policies, or VPC endpoint policies."
  resolution: "Manually check SCPs and resource policies. The simulator only evaluates identity policies and boundaries."

- symptoms: "Cannot determine all effective permissions for a role"
  diagnosis: "Multiple policy layers make manual calculation complex."
  resolution: "Use IAM Access Analyzer policy generation based on CloudTrail activity for accurate effective permissions."

## Output Format

```yaml
root_cause: "effective_permissions — <specific_finding>"
evidence:
  - type: identity_policies
    content: "<combined identity policy actions>"
  - type: permission_boundary
    content: "<boundary policy actions or 'not set'>"
  - type: scps
    content: "<SCP restrictions or 'not applicable'>"
  - type: effective_result
    content: "<intersection of all layers>"
severity: MEDIUM
mitigation:
  immediate: "Align policy layers for required access"
  long_term: "Use Access Analyzer for ongoing effective permission monitoring"
```


## Safety Ratings

safety_ratings:
  - "Effective permissions calculation (read-only analysis of policy intersection): GREEN — All triage commands are read-only API calls that do not modify any IAM resources"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the analysis reveals the principal has significantly more permissions than expected"
  - "Escalate if the effective permissions include sensitive actions (IAM, Organizations, KMS) that were not intended"
  - "Escalate if aligning policy layers requires changes to permission boundaries or SCPs"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Revert any policy modifications by restoring the previous policy version or reattaching the original policy
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam get-policy-version` returns full policy documents — classification: HIGH
- `aws iam simulate-principal-policy` reveals the complete effective permission model — classification: HIGH
- `aws iam generate-service-last-accessed-details` reveals actual service usage history — classification: MEDIUM
- `aws accessanalyzer start-policy-generation` generates policies based on CloudTrail activity — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- Clean up IAM Access Analyzer policy generation jobs if started: `aws accessanalyzer cancel-policy-generation --job-id <id>`

## Data Sensitivity

data_sensitivity:
  - command: "get-account-authorization-details"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-policy-version"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-role"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest Action: * or Resource: * in any policy"
  - "NEVER suggest removing permission boundaries"
  - "NEVER suggest AdministratorAccess for troubleshooting"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
