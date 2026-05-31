---
title: "F2 — Permission Set Failures"
description: "Diagnose issues with IAM Identity Center permission set provisioning"
status: active
severity: HIGH
triggers:
  - "Permission set provisioning failed"
  - "Cannot assign permission set"
  - "Role not created in account"
owner: devops-agent
objective: "Resolve permission set provisioning failures"
context: "Permission sets define the IAM roles created in member accounts. Provisioning deploys these roles. Failures occur when the target account blocks role creation, SCPs interfere, or the permission set configuration is invalid."
---

## Phase 1 — Triage

MUST:
- Check permission set status: `aws sso-admin describe-permission-set --instance-arn <arn> --permission-set-arn <ps-arn>`
- Check provisioning status: `aws sso-admin list-permission-set-provisioning-status --instance-arn <arn>`
- Verify the permission set's managed policies and inline policy are valid
- Check if SCPs in the target account block IAM role creation

SHOULD:
- Check if the permission set session duration is valid
- Verify the account assignment exists
- Check if the target account is active (not suspended)

MAY:
- Check CloudTrail in the target account for CreateRole failures
- Review the permission set's customer managed policy references

## Phase 2 — Remediate

MUST:
- Fix invalid policy configurations in the permission set
- Re-provision the permission set after fixes
- If SCP blocking: adjust the SCP to allow SSO role creation

SHOULD:
- Verify the IAM role was created in the target account after provisioning
- Test access through the SSO portal after fix
- Check all accounts where the permission set is assigned

MAY:
- Recreate the permission set if it's corrupted
- Use the SSO console for guided troubleshooting

## Common Issues

- symptoms: "Permission set provisioning fails in specific accounts"
  diagnosis: "SCP in the target account blocks IAM role creation by SSO."
  resolution: "Add an exception to the SCP for the SSO service principal."

- symptoms: "Customer managed policy not found"
  diagnosis: "The permission set references a customer managed policy that doesn't exist in the target account."
  resolution: "Create the referenced policy in all target accounts or remove the reference."

## Safety Ratings

```
safety_ratings:
  - "describe-permission-set: GREEN — read-only permission set details"
  - "list-permission-set-provisioning-status: GREEN — read-only provisioning status"
  - "update permission set policy: YELLOW — recoverable, policy can be reverted"
  - "re-provision permission set: YELLOW — recoverable, redeploys IAM roles"
  - "adjust SCP to allow SSO role creation: RED — modifying SCPs can break governance for all accounts"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Permission set provisioning fails across many accounts
- Customer managed policy referenced by permission set is missing in multiple accounts
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: Permission set policies (expose granted permissions and access levels)
- HIGH: Organization account list (all account IDs and emails)
- HIGH: SCP policy documents (expose security boundaries)
- MEDIUM: OU structure, control enablement status
- MEDIUM: SSO provisioning details (account-role mappings)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If a permission set update breaks access, revert the policy and re-provision to all affected accounts
- Document the permission set configuration before making changes
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- If an SCP was modified to allow SSO role creation, document the change and verify it doesn't weaken other security controls

## Output Format

```yaml
root_cause: "permission_set_failure — <specific cause>"
evidence:
  - type: sso_provisioning
    content: "<provisioning status and error>"
severity: HIGH
mitigation:
  immediate: "Fix the permission set or account configuration and re-provision"
  long_term: "Validate permission sets before assignment and monitor provisioning"
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
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"
