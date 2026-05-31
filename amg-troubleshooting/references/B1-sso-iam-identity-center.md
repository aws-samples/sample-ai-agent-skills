---
title: "B1 — SSO / IAM Identity Center Authentication Issues"
description: "Diagnose SSO and IAM Identity Center authentication failures with Amazon Managed Grafana"
status: active
severity: HIGH
triggers:
  - "SSO login failed"
  - "IAM Identity Center"
  - "cannot sign in"
  - "authentication error"
  - "SSO not working"
owner: devops-agent
objective: "Identify why SSO/IAM Identity Center authentication fails for an AMG workspace and restore access"
context: "AMG integrates with IAM Identity Center (formerly AWS SSO) for user authentication. Users must be assigned to the workspace through IAM Identity Center. Common failures include misconfigured SSO instance, users not assigned to the workspace, or IAM Identity Center not enabled in the region."
---

## Phase 1 — Triage

MUST:
- Check workspace authentication mode: `aws grafana describe-workspace-authentication --workspace-id <workspace-id>`
- Verify IAM Identity Center is enabled: `aws sso-admin list-instances`
- Check workspace authentication providers: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.authentication'`
- List workspace permissions to verify user assignments: `aws grafana list-permissions --workspace-id <workspace-id>`

SHOULD:
- Verify the user exists in IAM Identity Center: `aws identitystore list-users --identity-store-id <identity-store-id>`
- Check if the user is assigned to the Grafana application in IAM Identity Center
- Review CloudTrail for authentication events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=grafana.amazonaws.com --max-results 20`

MAY:
- Check IAM Identity Center permission sets and application assignments
- Verify the IAM Identity Center instance is in the same region as the workspace

## Phase 2 — Remediate

MUST:
- Ensure IAM Identity Center is enabled in the workspace region
- Assign users or groups to the workspace: `aws grafana update-permissions --workspace-id <workspace-id> --update-instruction-batch '{"action":"ADD","role":"EDITOR","users":[{"id":"<user-id>","type":"SSO_USER"}]}'`
- Verify the workspace authentication mode is set to AWS_SSO

SHOULD:
- Assign groups rather than individual users for easier management
- Verify users can access the IAM Identity Center user portal first
- Check that the SSO application for Grafana is properly configured

MAY:
- Set up automatic provisioning of users from IAM Identity Center to the workspace
- Configure group-based role mapping for consistent access control

## Common Issues

- symptoms: "User gets 'You do not have permission to access this workspace' error"
  diagnosis: "User is not assigned to the workspace through IAM Identity Center."
  resolution: "Add the user or their group to the workspace permissions via aws grafana update-permissions."

- symptoms: "SSO login redirects but never completes"
  diagnosis: "IAM Identity Center instance is not in the same region as the workspace, or the SSO application is misconfigured."
  resolution: "Verify IAM Identity Center region matches workspace region. Check SSO application configuration."

- symptoms: "User can log in but has no permissions"
  diagnosis: "User is assigned to the workspace but has no Grafana role (ADMIN, EDITOR, VIEWER)."
  resolution: "Update the user's permission with the appropriate role via aws grafana update-permissions."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace-authentication: GREEN — read-only auth config check"
  - "list-permissions: GREEN — read-only permission enumeration"
  - "update-permissions (ADD user): YELLOW — grants workspace access to a user"
  - "update-permissions (REVOKE user): RED — removes workspace access from a user"
```

## Escalation Conditions

- IAM Identity Center is not enabled in the workspace region and requires organizational changes
- Multiple users locked out of a production monitoring workspace
- SSO application configuration requires Identity Center administrator access
- Permission changes affect users across multiple workspaces

## Data Sensitivity

- `aws grafana list-permissions`: MEDIUM — reveals who has access to the workspace and their roles
- `aws identitystore list-users`: MEDIUM — reveals identity store user information
- `aws cloudtrail lookup-events`: MEDIUM — reveals authentication and permission change history

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER grant ADMIN role to users who only need VIEWER or EDITOR access
- NEVER remove all ADMIN users from a workspace (will lock out management access)

## Phase 3 — Rollback

1. If a user was incorrectly removed, re-add them: `aws grafana update-permissions --workspace-id <id> --update-instruction-batch '{"action":"ADD","role":"<role>","users":[{"id":"<user-id>","type":"SSO_USER"}]}'`
2. If a user was granted the wrong role, update it with the correct role
3. If SSO configuration was changed, verify IAM Identity Center settings match workspace region
4. Verify all affected users can log in and have appropriate access
5. Document permission changes for audit trail

## Output Format

```yaml
root_cause: "sso_authentication — <specific_cause>"
evidence:
  - type: auth_config
    content: "<workspace authentication configuration>"
  - type: permissions
    content: "<workspace permission assignments>"
  - type: sso_instance
    content: "<IAM Identity Center instance details>"
severity: HIGH
mitigation:
  immediate: "Assign user to workspace with appropriate role"
  long_term: "Use group-based assignments for scalable access management"
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
