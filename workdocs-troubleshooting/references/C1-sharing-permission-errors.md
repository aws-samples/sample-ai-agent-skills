---
title: "C1 — Sharing Permission Errors"
description: "Diagnose WorkDocs sharing and permission errors"
status: active
severity: MEDIUM
triggers: ["cannot share", "permission denied", "sharing failed", "access denied to document"]
owner: devops-agent
objective: "Resolve sharing permission issues including role-based access, folder inheritance, and external sharing"
context: "WorkDocs permissions are VIEWER, CONTRIBUTOR, CO_OWNER, and OWNER. Folder permissions cascade to contents unless overridden."
---

## Phase 1 — Triage

MUST:
- Check resource permissions: `aws workdocs describe-resource-permissions --resource-id <resource-id>`
- Verify user exists and is active: `aws workdocs describe-users --organization-id <org-id> --query 'Users[?Username==`<username>`]'`
- Check if the resource is a folder or document (permissions behave differently)
- Verify the sharing user has CO_OWNER or OWNER permission to share

SHOULD:
- Check parent folder permissions for inherited access
- Verify admin sharing policies (external sharing enabled/disabled)
- Check if the target user's account type supports the requested access level

MAY:
- Review activity feed for sharing events: `aws workdocs describe-activities --organization-id <org-id> --limit 20`
- Check if the document is locked by another user

## Phase 2 — Remediate

MUST:
- Grant appropriate permission level to the user: `aws workdocs add-resource-permissions --resource-id <resource-id> --principals '[{"Id":"<user-id>","Type":"USER","Role":"CONTRIBUTOR"}]'`
- Ensure the sharing user has sufficient permission to share
- Verify admin policies allow the type of sharing requested

SHOULD:
- Use folder-level sharing for groups of documents
- Set appropriate permission levels (VIEWER for read-only, CONTRIBUTOR for edit)

MAY:
- Configure sharing notifications to inform users of new access
- Set up permission templates for common sharing patterns

## Safety Ratings

```
safety_ratings:
  - "workdocs describe-resource-permissions: GREEN — read-only permission check"
  - "workdocs describe-users: GREEN — read-only user listing"
  - "workdocs add-resource-permissions: YELLOW — grants access to resources"
  - "workdocs remove-resource-permissions: YELLOW — revokes access to resources"
```

## Escalation Conditions

- Permission changes do not take effect after API confirmation
- Users cannot access shared resources despite correct permissions
- External sharing is required but blocked by admin policy
- Permission inheritance is not working as expected across folder hierarchy

## Data Sensitivity

- `aws workdocs describe-resource-permissions`: MEDIUM — reveals who has access to resources
- `aws workdocs describe-users`: MEDIUM — reveals user identities
- `aws workdocs describe-activities`: MEDIUM — reveals sharing activity patterns

## Prohibited Actions

- NEVER grant OWNER permission to resolve access issues — use CONTRIBUTOR or VIEWER
- NEVER enable external sharing without admin approval
- NEVER remove existing permissions to troubleshoot — add permissions instead
- NEVER share resources with all users as a quick fix

## Phase 3 — Rollback

1. If permissions were granted during troubleshooting, remove unauthorized ones: `aws workdocs remove-resource-permissions --resource-id <id> --principal-id <principal-id>`
2. If folder-level permissions were changed, verify child resources still have correct access
3. If admin sharing policies were modified, restore original settings
4. Verify all affected users have the correct access level after changes
5. Document permission changes for the security team

## Common Issues

- symptoms: "User cannot access a shared document"
  diagnosis: "User has VIEWER permission but needs CONTRIBUTOR, or permission was set on a different folder."
  resolution: "Check permission level and resource ID. Grant the correct permission on the specific resource."

- symptoms: "Cannot share document externally"
  diagnosis: "Admin policy has disabled external sharing for the organization."
  resolution: "Contact admin to enable external sharing if appropriate. Use internal sharing as alternative."

## Output Format

```yaml
root_cause: "sharing_permission — <specific_cause>"
evidence:
  - type: permission_issue
    content: "<resource permissions and user details>"
severity: MEDIUM
mitigation:
  immediate: "Fix permissions"
  long_term: "Implement permission templates and review process"
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
