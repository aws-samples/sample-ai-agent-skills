---
title: "F1 — SSO Configuration Issues"
description: "Diagnose IAM Identity Center (SSO) configuration problems in Control Tower"
status: active
severity: HIGH
triggers:
  - "SSO not working"
  - "IAM Identity Center error"
  - "Cannot access accounts via SSO"
  - "Identity source issues"
owner: devops-agent
objective: "Resolve IAM Identity Center configuration issues"
context: "Control Tower sets up IAM Identity Center as the default access method. It creates permission sets and assigns them to groups/users for account access. Issues arise from identity source changes, permission set misconfigurations, or SSO instance problems."
---

## Phase 1 — Triage

MUST:
- Check SSO instance status: `aws sso-admin list-instances`
- Verify identity source configuration (Identity Center directory, Active Directory, or external IdP)
- Check permission sets: `aws sso-admin list-permission-sets --instance-arn <arn>`
- Verify account assignments: `aws sso-admin list-account-assignments --instance-arn <arn> --account-id <id> --permission-set-arn <arn>`

SHOULD:
- Check if the SSO instance region matches the CT home region
- Verify the identity source is reachable (for external IdP or AD)
- Check if users/groups exist in the identity source

MAY:
- Check CloudTrail for SSO-related API calls
- Review SSO access portal URL and configuration

## Phase 2 — Remediate

MUST:
- If identity source changed: reconfigure user/group assignments
- If permission set broken: recreate or update the permission set
- If SSO instance issue: check AWS service health

SHOULD:
- Verify MFA configuration is not blocking access
- Check session duration settings
- Test access with a known-good user/group

MAY:
- Reconfigure the identity source if it was changed accidentally
- Contact AWS Support for SSO instance-level issues

## Common Issues

- symptoms: "Users cannot see any accounts in SSO portal"
  diagnosis: "No account assignments exist for the user's group."
  resolution: "Create account assignments linking the user/group to accounts with permission sets."

- symptoms: "SSO login fails with identity source error"
  diagnosis: "External IdP or AD connector is misconfigured or unreachable."
  resolution: "Verify IdP metadata, SAML configuration, or AD connector health."

- symptoms: "Permission set changes not taking effect"
  diagnosis: "Permission sets must be provisioned to accounts after changes."
  resolution: "Provision the updated permission set to the affected accounts."

## Safety Ratings

```
safety_ratings:
  - "list-instances: GREEN — read-only SSO instance listing"
  - "list-permission-sets: GREEN — read-only permission set listing"
  - "list-account-assignments: GREEN — read-only assignment listing"
  - "reconfigure user/group assignments: YELLOW — recoverable, assignments can be changed back"
  - "update permission set: YELLOW — recoverable, permission set can be reverted"
  - "change identity source: RED — can break all user access across all accounts"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Identity source was changed accidentally (affects all users)
- SSO instance-level issue requires AWS Support
- Fix requires moving the Audit or Log Archive account

## Data Sensitivity

- HIGH: Organization account list (all account IDs and emails)
- HIGH: SSO user/group assignments (expose who has access to what)
- HIGH: Permission set policies (expose granted permissions)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Identity source configuration (IdP metadata, SAML config)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If identity source was changed, reverting requires reconfiguring all user/group assignments — document the current assignments before any changes
- If permission sets were modified, re-provision them to all affected accounts after reverting
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- If SSO access is broken for all users, use IAM credentials as a fallback while investigating — do NOT make hasty SSO changes

## Output Format

```yaml
root_cause: "sso_configuration — <specific issue>"
evidence:
  - type: sso_admin_api
    content: "<SSO instance and configuration details>"
severity: HIGH
mitigation:
  immediate: "Fix the SSO configuration issue"
  long_term: "Document SSO configuration and implement monitoring"
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
