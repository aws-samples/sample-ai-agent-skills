---
title: "B3 — User and Role Mapping Issues"
description: "Diagnose user and role mapping problems in Amazon Managed Grafana"
status: active
severity: MEDIUM
triggers:
  - "wrong role"
  - "user permissions"
  - "role mapping"
  - "cannot edit dashboard"
  - "viewer instead of editor"
owner: devops-agent
objective: "Identify why users have incorrect roles or permissions in the AMG workspace and fix the mapping"
context: "AMG has three Grafana roles: ADMIN, EDITOR, and VIEWER. Role assignment depends on the authentication mode — SSO uses workspace permission assignments, SAML uses attribute mapping from the IdP. Incorrect role mapping leads to users having too much or too little access."
---

## Phase 1 — Triage

MUST:
- List current workspace permissions: `aws grafana list-permissions --workspace-id <workspace-id>`
- Check the user's assigned role in the workspace
- Verify authentication mode: `aws grafana describe-workspace-authentication --workspace-id <workspace-id>`
- If SAML, check role attribute mapping configuration

SHOULD:
- Verify the user's identity in the identity provider
- Check if group-based role assignment is configured
- Review recent permission changes in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdatePermissions --max-results 10`

MAY:
- Check if the user belongs to multiple groups with conflicting role assignments
- Review the SAML assertion for the role attribute value

## Phase 2 — Remediate

MUST:
- Update user role if incorrect: `aws grafana update-permissions --workspace-id <workspace-id> --update-instruction-batch '{"action":"ADD","role":"EDITOR","users":[{"id":"<user-id>","type":"SSO_USER"}]}'`
- For SAML, ensure the IdP sends the correct role attribute value

SHOULD:
- Use group-based role assignment for consistent management
- Document role assignment policies for the team
- Audit permissions periodically

MAY:
- Implement automated role assignment based on IdP group membership
- Set up alerts for permission changes via CloudTrail

## Common Issues

- symptoms: "User has VIEWER role but should be EDITOR"
  diagnosis: "User was assigned VIEWER role in workspace permissions, or SAML role attribute maps to VIEWER."
  resolution: "Update the user's role via aws grafana update-permissions or fix the SAML role attribute mapping."

- symptoms: "New users get no role assigned"
  diagnosis: "Users are authenticated but not assigned to the workspace, or SAML role mapping is not configured."
  resolution: "Assign users to the workspace with a role, or configure SAML role attribute mapping."

- symptoms: "User role changes are not taking effect"
  diagnosis: "User needs to log out and log back in for role changes to apply."
  resolution: "Have the user sign out completely and sign back in to pick up the new role."

## Safety Ratings

```
safety_ratings:
  - "list-permissions: GREEN — read-only permission enumeration"
  - "describe-workspace-authentication: GREEN — read-only auth check"
  - "update-permissions (change role): YELLOW — changes user access level"
  - "update SAML role attribute mapping: YELLOW — changes role assignment for all SAML users"
```

## Escalation Conditions

- Users have ADMIN role who should only have VIEWER or EDITOR access
- Role mapping changes affect all users authenticating via SAML
- Permission audit reveals unauthorized access to production monitoring workspace
- Group-based role assignment conflicts with individual user assignments

## Data Sensitivity

- `aws grafana list-permissions`: MEDIUM — reveals user identities and access levels
- SAML role attribute mapping: MEDIUM — reveals how roles are derived from IdP attributes
- Permission change history: MEDIUM — reveals who changed access and when

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER grant ADMIN role without explicit approval from workspace owner
- NEVER change role mappings during an active incident when monitoring access is critical

## Phase 3 — Rollback

1. If a user's role was changed incorrectly, restore it: `aws grafana update-permissions --workspace-id <id> --update-instruction-batch '{"action":"ADD","role":"<correct-role>","users":[{"id":"<user-id>","type":"SSO_USER"}]}'`
2. If SAML role mapping was changed, restore the previous attribute mapping configuration
3. Have affected users log out and log back in to pick up corrected roles
4. Verify all users have appropriate access levels after rollback
5. Document the permission change and rollback for audit purposes

## Output Format

```yaml
root_cause: "user_role_mapping — <specific_cause>"
evidence:
  - type: permissions
    content: "<workspace permission assignments>"
  - type: auth_mode
    content: "<SSO or SAML configuration>"
  - type: role_mapping
    content: "<role attribute mapping if SAML>"
severity: MEDIUM
mitigation:
  immediate: "Update user role assignment to the correct value"
  long_term: "Implement group-based role mapping for automated management"
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
