---
title: "E1 — User Management"
description: "Diagnose user provisioning, access, and role management issues"
status: active
severity: MEDIUM
triggers: ["user access denied", "cannot add user", "user not found", "admin access", "SSO error"]
owner: devops-agent
objective: "Resolve user management issues for Monitron project access"
context: "Monitron uses AWS IAM Identity Center for user management. Users must be provisioned in IAM Identity Center before being added to projects. Admin and technician roles have different permissions."
---

## Phase 1 — Triage

MUST:
- List project admin users: `aws monitron list-project-admin-users --project-id <project-id>`
- Verify IAM Identity Center is configured: `aws sso-admin list-instances`
- Check if the user exists in IAM Identity Center

SHOULD:
- Verify the user's email matches the IAM Identity Center identity
- Check if the user has accepted the project invitation
- Confirm the user has the Monitron mobile app installed

MAY:
- Review CloudTrail for user management events
- Check if the user's IAM Identity Center session is active

## Phase 2 — Remediate

MUST:
- Provision the user in IAM Identity Center if not already present
- Add the user to the Monitron project with the correct role (admin or technician)
- Ensure the user's email matches their IAM Identity Center identity

SHOULD:
- Send project invitation and confirm user acceptance
- Verify the user can log in to the Monitron mobile app

MAY:
- Set up user groups in IAM Identity Center for easier management
- Configure automatic user provisioning via SCIM

## Common Issues

- symptoms: "User cannot access Monitron project"
  diagnosis: "User not provisioned in IAM Identity Center or not added to project."
  resolution: "Create user in IAM Identity Center. Add to Monitron project: aws monitron create-project-admin-user."

- symptoms: "User can log in but has limited functionality"
  diagnosis: "User has technician role instead of admin role."
  resolution: "Verify user role. If admin access needed, add as project admin user."

## Safety Ratings

```
safety_ratings:
  - "monitron list-project-admin-users: GREEN — read-only user listing"
  - "sso-admin list-instances: GREEN — read-only SSO metadata"
  - "monitron create-project-admin-user: YELLOW — grants admin access to project"
  - "monitron delete-project-admin-user: YELLOW — removes admin access"
```

## Escalation Conditions

- IAM Identity Center is not enabled and requires organization-level setup
- User provisioning requires SCIM integration with external identity provider
- Admin user removal affects project management capability
- Cross-account user access requires organization-level coordination

## Data Sensitivity

- `aws monitron list-project-admin-users`: MEDIUM — reveals admin user identities
- `aws sso-admin list-instances`: LOW — returns SSO instance metadata
- User email addresses: MEDIUM — personally identifiable information
- User role assignments: LOW — access level information

## Prohibited Actions

- NEVER remove the last admin user from a project — it becomes unmanageable
- NEVER share user credentials or SSO tokens
- NEVER grant admin access without proper authorization
- NEVER modify IAM Identity Center configuration without understanding impact on other services

## Phase 3 — Rollback

1. If a user was added with wrong role, remove and re-add with correct role
2. If an admin user was removed incorrectly, re-add: `aws monitron create-project-admin-user --project-id <id> --user-arn <arn>`
3. If IAM Identity Center was modified, consult SSO admin for restoration
4. Verify project access is restored for affected users
5. Document user management changes for audit trail

## Output Format

```yaml
root_cause: "user_management — <specific_cause>"
evidence:
  - type: user_error
    content: "<user status, SSO details, project access>"
severity: MEDIUM
mitigation:
  immediate: "Fix user provisioning and project access"
  long_term: "Implement user lifecycle management with IAM Identity Center"
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
