---
title: "B2 — Cross-Account Role Assumption"
description: "Diagnose cross-account role assumption failures requiring configuration on both accounts"
status: active
severity: HIGH
triggers:
  - "cross-account assume role"
  - "assume role other account"
  - "cross-account denied"
  - "external account role"
owner: devops-agent
objective: "Ensure both the trusting and trusted accounts are correctly configured for cross-account role assumption"
context: "Cross-account role assumption requires TWO configurations: (1) the target role's trust policy must allow the source account/principal, and (2) the source principal must have sts:AssumeRole permission for the target role ARN. Missing either side causes denial."
---

## Phase 1 — Triage

MUST:
- Identify source and target accounts: `aws sts get-caller-identity`
- Get the target role's trust policy: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'` (run in target account)
- Check source principal's sts:AssumeRole permission: `aws iam simulate-principal-policy --policy-source-arn <source-arn> --action-names sts:AssumeRole --resource-arns arn:aws:iam::<target-account>:role/<role>`
- Attempt the assumption: `aws sts assume-role --role-arn arn:aws:iam::<target-account>:role/<role> --role-session-name test`

SHOULD:
- Check CloudTrail in BOTH accounts for the failed AssumeRole event
- Verify no SCP in either account blocks sts:AssumeRole
- Check if ExternalId is required: look for `sts:ExternalId` condition in trust policy

MAY:
- Check if the target account has an SCP blocking AssumeRole from external accounts
- Verify the source account is not blocked by an SCP from calling sts:AssumeRole

## Phase 2 — Remediate

MUST:
- In target account — trust policy must allow the source:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::<source-account>:role/<source-role>"},
    "Action": "sts:AssumeRole"
  }
  ```
- In source account — identity policy must allow:
  ```json
  {
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::<target-account>:role/<target-role>"
  }
  ```

SHOULD:
- Use specific principal ARNs in trust policy instead of account root
- Add sts:ExternalId condition for third-party access
- Add aws:PrincipalOrgID condition if both accounts are in the same Organization

MAY:
- Use AWS Organizations delegated administrator for centralized role management
- Set up AWS SSO/Identity Center for cross-account access instead of manual role assumption

## Common Issues

- symptoms: "Trust policy allows the account root but specific role is denied"
  diagnosis: "Account root principal delegates to IAM. The source role needs sts:AssumeRole in its identity policy."
  resolution: "Add sts:AssumeRole permission to the source role's identity policy."

- symptoms: "Cross-account assume works from CLI but not from EC2 instance"
  diagnosis: "The EC2 instance role needs sts:AssumeRole permission. The trust policy must allow the instance role ARN."
  resolution: "Add sts:AssumeRole to the instance role and ensure trust policy lists the instance role ARN."

- symptoms: "AssumeRole fails with AccessDenied but both sides look correct"
  diagnosis: "An SCP in either the source or target account may be blocking sts:AssumeRole."
  resolution: "Check SCPs in both accounts. SCPs can block AssumeRole even if IAM policies allow it."

## Output Format

```yaml
root_cause: "cross_account_assume — <specific_cause>"
evidence:
  - type: trust_policy
    content: "<target role trust policy>"
  - type: source_policy
    content: "<source principal's sts:AssumeRole permission>"
  - type: scp_check
    content: "<SCP evaluation in both accounts>"
severity: HIGH
mitigation:
  immediate: "Fix trust policy and/or source identity policy"
  long_term: "Use AWS Organizations and SCPs to govern cross-account access patterns"
```


## Safety Ratings

| Action | Rating | Justification |
|--------|--------|---------------|
| `aws sts get-caller-identity` | GREEN | Read-only diagnostic |
| `aws iam get-role` | GREEN | Read-only diagnostic |
| `aws iam simulate-principal-policy` | GREEN | Read-only diagnostic |
| `aws cloudtrail lookup-events` | GREEN | Read-only diagnostic |
| `aws organizations describe-policy` | GREEN | Read-only diagnostic |
| `aws iam update-assume-role-policy` (target account) | RED | Trust policy changes affect cross-account security boundaries |
| `aws iam attach-role-policy` / `put-role-policy` (source account) | RED | Grants sts:AssumeRole across account boundaries |
| `aws sts assume-role` (test) | YELLOW | Creates a temporary cross-account session — use only for verification |

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

- No temporary resources are created during cross-account assumption troubleshooting

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
