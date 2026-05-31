---
title: "D1 — Permission Boundary Conflicts"
description: "Diagnose permission boundary blocking expected access"
status: active
severity: HIGH
triggers:
  - "permission boundary"
  - "permissions boundary"
  - "boundary blocking"
  - "boundary conflict"
  - "effective permissions reduced"
owner: devops-agent
objective: "Identify when a permission boundary is restricting access and determine the correct fix"
context: "Permission boundaries set the MAXIMUM permissions a principal can have. They do NOT grant permissions — identity policies do that. Effective permissions = intersection of identity policy AND permission boundary. If the boundary doesn't allow an action, no identity policy can grant it."
---

## Phase 1 — Triage

MUST:
- Check if a permission boundary is set: `aws iam get-user --user-name <user>` or `aws iam get-role --role-name <role>` (look for PermissionsBoundary field)
- Get the boundary policy: `aws iam get-policy-version --policy-arn <boundary-arn> --version-id <version>`
- Get the identity policy: `aws iam get-policy-version --policy-arn <identity-policy-arn> --version-id <version>`
- Verify the denied action is in BOTH the boundary AND the identity policy

SHOULD:
- Simulate with boundary context: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names <action> --resource-arns <resource>`
- Compare the boundary policy with the identity policy to find the gap
- Check if the boundary uses conditions that don't match the request context

MAY:
- Check if the boundary was recently changed: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=PutUserPermissionsBoundary`
- Review the boundary policy for overly restrictive scope

## Phase 2 — Remediate

MUST:
- If the action should be allowed: add it to the permission boundary policy
- If the boundary is correct: the identity policy is granting more than intended — this is working as designed
- Update boundary: `aws iam create-policy-version --policy-arn <boundary-arn> --policy-document file://boundary.json --set-as-default`

SHOULD:
- Never remove the boundary to fix access — adjust the boundary or accept the restriction
- Document why the boundary exists and what it's protecting
- Test changes with simulate-principal-policy before applying

MAY:
- Use separate boundaries for different teams/roles
- Implement boundary policies through IaC for consistency

## Common Issues

- symptoms: "Identity policy allows s3:* but S3 calls are denied"
  diagnosis: "Permission boundary does not include S3 actions. Effective permissions = intersection."
  resolution: "Add the required S3 actions to the permission boundary, or accept the restriction."

- symptoms: "New permissions added to identity policy don't take effect"
  diagnosis: "The permission boundary doesn't include the new actions."
  resolution: "Update the boundary to include the new actions. Both boundary and identity policy must allow."

- symptoms: "Permission boundary allows the action but it's still denied"
  diagnosis: "The identity policy doesn't have the Allow statement. Boundaries don't grant — they only restrict."
  resolution: "Add the Allow statement to the identity policy. The boundary only sets the ceiling."

## Output Format

```yaml
root_cause: "boundary_conflict — <action> allowed in <identity|boundary> but not in <identity|boundary>"
evidence:
  - type: boundary_policy
    content: "<permission boundary document>"
  - type: identity_policy
    content: "<identity policy document>"
  - type: effective_permissions
    content: "intersection analysis"
severity: HIGH
mitigation:
  immediate: "Update boundary or identity policy to align"
  long_term: "Maintain boundary policies through IaC with regular review"
```


## Safety Ratings

safety_ratings:
  - "Permission boundary modification (changes affect the maximum permissions ceiling for a principal): RED — Boundary changes directly affect the maximum permissions available to the principal and can inadvertently grant access to previously restricted actions"

## Escalation Conditions

escalation_conditions:
  - "ALWAYS escalate to a human before modifying a permission boundary"
  - "Escalate if the boundary is applied to multiple principals"
  - "Escalate if the boundary was set by a security team as a guardrail"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Restore the original permission boundary by creating a new policy version with the original boundary document using `aws iam create-policy-version --policy-arn <boundary-arn> --policy-document file://original-boundary.json --set-as-default`
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam get-policy-version` for the boundary policy returns the full maximum permissions ceiling — classification: HIGH
- `aws iam get-user` or `aws iam get-role` reveals whether a boundary is set and its ARN — classification: MEDIUM
- `aws iam simulate-principal-policy` reveals effective permissions after boundary intersection — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- If a new boundary policy version was created, delete old non-default versions to stay below the 5-version limit

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
