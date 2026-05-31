---
title: "B1 — Role Trust Policy Failures"
description: "Diagnose role assumption failures caused by trust policy misconfiguration"
status: active
severity: HIGH
triggers:
  - "trust policy"
  - "AssumeRole denied"
  - "not authorized to perform sts:AssumeRole"
  - "role assumption failed"
owner: devops-agent
objective: "Identify and fix trust policy misconfigurations preventing role assumption"
context: "A role's trust policy (AssumeRolePolicyDocument) controls WHO can assume the role. The assuming principal must be listed in the trust policy AND must have sts:AssumeRole permission in their own identity policy. Common issues include wrong principal format, missing conditions, and typos in ARNs."
---

## Phase 1 — Triage

MUST:
- Get the role's trust policy: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'`
- Identify the assuming principal: `aws sts get-caller-identity`
- Verify the principal ARN format in the trust policy matches the caller
- Check if the caller has sts:AssumeRole permission: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names sts:AssumeRole --resource-arns <role-arn>`

SHOULD:
- Check CloudTrail for the failed AssumeRole event: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole`
- Verify condition keys in the trust policy (e.g., sts:ExternalId, aws:SourceAccount)
- Check if the trust policy uses the correct principal type (AWS, Service, Federated)

MAY:
- Check if the role has a permission boundary that might affect assumption
- Verify the role is not in a suspended or deleted state

## Phase 2 — Remediate

MUST:
- Fix the trust policy Principal to match the assuming entity's ARN exactly
- For IAM users: `"Principal": {"AWS": "arn:aws:iam::<account-id>:user/<username>"}`
- For IAM roles: `"Principal": {"AWS": "arn:aws:iam::<account-id>:role/<rolename>"}`
- For AWS services: `"Principal": {"Service": "<service>.amazonaws.com"}`
- For SAML: `"Principal": {"Federated": "arn:aws:iam::<account-id>:saml-provider/<name>"}`
- Update trust policy: `aws iam update-assume-role-policy --role-name <role> --policy-document file://trust-policy.json`

SHOULD:
- Add conditions for security (sts:ExternalId for third-party access)
- Use specific ARNs instead of account-wide principals
- Verify the fix with: `aws sts assume-role --role-arn <role-arn> --role-session-name test`

MAY:
- Add aws:PrincipalOrgID condition for organization-wide trust
- Enable CloudTrail to monitor AssumeRole events

## Common Issues

- symptoms: "AssumeRole fails with 'is not authorized to perform sts:AssumeRole on resource'"
  diagnosis: "Either the trust policy doesn't list the caller, or the caller lacks sts:AssumeRole permission."
  resolution: "Check both sides: trust policy Principal AND caller's identity policy."

- symptoms: "Trust policy uses account root ARN but specific user/role is denied"
  diagnosis: "Account root principal (arn:aws:iam::<account>:root) allows any principal in that account IF they have sts:AssumeRole permission."
  resolution: "Verify the caller's identity policy includes sts:AssumeRole for the role ARN."

- symptoms: "Trust policy has correct principal but condition key mismatch"
  diagnosis: "Condition keys like sts:ExternalId must match exactly. Case-sensitive."
  resolution: "Verify the ExternalId or other condition values match exactly between trust policy and AssumeRole call."

## Output Format

```yaml
root_cause: "trust_policy — <specific_cause>"
evidence:
  - type: trust_policy
    content: "<AssumeRolePolicyDocument>"
  - type: caller_identity
    content: "<sts get-caller-identity output>"
  - type: cloudtrail_event
    content: "<failed AssumeRole event>"
severity: HIGH
mitigation:
  immediate: "Fix trust policy Principal and/or caller's sts:AssumeRole permission"
  long_term: "Audit trust policies for overly broad or incorrect principals"
```


## Safety Ratings

| Action | Rating | Justification |
|--------|--------|---------------|
| `aws iam get-role` | GREEN | Read-only diagnostic |
| `aws sts get-caller-identity` | GREEN | Read-only diagnostic |
| `aws iam simulate-principal-policy` | GREEN | Read-only diagnostic |
| `aws cloudtrail lookup-events` | GREEN | Read-only diagnostic |
| `aws iam update-assume-role-policy` | RED | Trust policy changes directly affect which principals can assume the role |
| `aws sts assume-role` (test) | YELLOW | Creates a temporary session — use only for verification |

## Escalation Conditions

- Remediation requires modifying IAM policies in a production account
- Fix involves granting cross-account access
- Root cause involves the management account
- Multiple principals are affected
- The denied action involves security-sensitive services (KMS, STS, Organizations)

## Rollback

1. **Before any change**: Document current state with `aws iam get-policy-version` or `aws iam get-role`
2. **After change**: Verify the fix resolves the issue without granting excessive permissions
3. **If change causes issues**: Revert to the previous policy version or detach the modified policy
4. **Cleanup**: Remove any temporary policies or test principals created during investigation

## Data Sensitivity

| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-account-authorization-details` | HIGH | Contains all IAM entities — redact from stored output |
| `get-policy-version` | HIGH | Contains full policy document — summarize, don't reproduce |
| `simulate-principal-policy` | MEDIUM | Shows effective permissions — summarize results |
| `list-*` | LOW | Resource listings — safe to include |

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting `AdministratorAccess` to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest disabling MFA requirements as a troubleshooting step
- ALWAYS identify the specific denied action from CloudTrail before suggesting a fix
- ALWAYS use least-privilege: grant only the specific action on the specific resource

## Cleanup

- No temporary resources are created during trust policy troubleshooting

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
