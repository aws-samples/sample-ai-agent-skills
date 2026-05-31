---
title: "F1 — SSO Integration"
description: "Diagnose and resolve SSO/IAM Identity Center integration issues with AFT"
status: active
severity: HIGH
triggers: ["SSO assignment failed", "permission set not found", "SSO error", "IAM Identity Center integration"]
owner: devops-agent
objective: "Ensure SSO permission sets are correctly assigned to AFT-provisioned accounts"
context: "AFT assigns existing SSO permission sets to provisioned accounts. Permission sets must exist before AFT references them. SSO integration failures are typically caused by non-existent permission sets, incorrect SSO instance configuration, or permission issues."
---

## Phase 1 — Triage

MUST:
- List SSO instances: `aws sso-admin list-instances`
- List available permission sets: `aws sso-admin list-permission-sets --instance-arn <instance-arn>`
- Check if the referenced permission set exists: `aws sso-admin describe-permission-set --instance-arn <instance-arn> --permission-set-arn <ps-arn>`

SHOULD:
- Verify SSO group/user assignments for the account: `aws sso-admin list-account-assignments --instance-arn <instance-arn> --account-id <account-id> --permission-set-arn <ps-arn>`
- Check the account request configuration for SSO references
- Review AFT Lambda logs for SSO-related errors

MAY:
- Check SSO provisioning status for the permission set
- Verify SSO identity source configuration (external IdP, AD, Identity Center directory)

## Phase 2 — Remediate

MUST:
- Create missing permission sets before re-running AFT: `aws sso-admin create-permission-set --instance-arn <instance-arn> --name <name>`
- Fix permission set names in the account request to match existing sets exactly
- Ensure the AFT management account has `sso-admin:*` permissions

SHOULD:
- Provision the permission set to the target account after creation
- Verify SSO group names match the account request configuration
- Check for SSO permission set provisioning failures

MAY:
- Implement SSO permission set creation as part of AFT global customizations
- Set up SSO permission set validation in the account request pipeline

## Safety Ratings

```
safety_ratings:
  - "sso-admin list-instances: GREEN — read-only SSO instance listing"
  - "sso-admin list-permission-sets: GREEN — read-only permission set listing"
  - "sso-admin create-permission-set: YELLOW — creates new SSO permission set"
  - "sso-admin create-account-assignment: YELLOW — assigns SSO access to account"
```

## Escalation Conditions

- SSO instance is managed by a different team and permission sets cannot be created
- SSO integration requires changes to the external identity provider
- Permission set provisioning fails with internal SSO errors
- SSO assignments conflict with existing manual assignments

## Data Sensitivity

- `aws sso-admin list-instances`: LOW — SSO instance metadata
- `aws sso-admin list-permission-sets`: MEDIUM — reveals SSO access structure
- `aws sso-admin list-account-assignments`: HIGH — reveals who has access to which accounts
- SSO group memberships: HIGH — reveals organizational access patterns

## Prohibited Actions

- NEVER delete SSO permission sets that are assigned to production accounts
- NEVER modify SSO permission set policies without security team approval
- NEVER assign AdministratorAccess permission sets without explicit approval
- NEVER expose SSO user/group information in logs or outputs

## Phase 3 — Rollback

1. If incorrect SSO assignments were made, remove them: `aws sso-admin delete-account-assignment`
2. If a permission set was created in error, delete it (only if not assigned to any account)
3. If SSO provisioning failed, re-provision: `aws sso-admin provision-permission-set`
4. Document all SSO changes and notify the identity team
5. Verify account access is correct after rollback

## Common Issues

- symptoms: "AFT account request fails with 'Permission set not found'"
  diagnosis: "The permission set name in the account request does not match any existing SSO permission set."
  resolution: "List permission sets: `aws sso-admin list-permission-sets --instance-arn <arn>`. Fix the name in the account request."

- symptoms: "SSO assignment succeeds but users cannot access the account"
  diagnosis: "Permission set is not provisioned to the account, or SSO group membership is incorrect."
  resolution: "Provision the permission set: `aws sso-admin provision-permission-set`. Verify group membership."

## Output Format

```yaml
root_cause: "sso_integration — <specific_cause>"
evidence:
  - type: sso_error
    content: "<permission set details, assignment status, and error>"
severity: HIGH
mitigation:
  immediate: "Fix SSO configuration or create missing permission sets"
  long_term: "Implement SSO permission set validation in account request pipeline"
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
