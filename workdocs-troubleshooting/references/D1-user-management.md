---
title: "D1 — User Management"
description: "Diagnose WorkDocs user management issues"
status: active
severity: MEDIUM
triggers: ["cannot create user", "user activation failed", "user status issue", "user type change"]
owner: devops-agent
objective: "Resolve user management issues including creation, activation, deactivation, and type changes"
context: "WorkDocs users are managed through Directory Service. User types (ADMIN, POWERUSER, USER, MINIMALUSER) determine capabilities and storage."
---

## Phase 1 — Triage

MUST:
- List users and their status: `aws workdocs describe-users --organization-id <org-id> --query 'Users[*].{Username:Username,Status:Status,Type:Type}'`
- Check specific user details: `aws workdocs describe-users --organization-id <org-id> --query 'Users[?Username==`<username>`]'`
- Verify directory is healthy: `aws ds describe-directories --directory-ids <directory-id>`
- Check if user exists in the directory

SHOULD:
- Verify user activation status (ACTIVE, INACTIVE, PENDING)
- Check user storage quota and usage
- Verify the caller has admin permissions for user management

MAY:
- Review CloudTrail for user management events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=ActivateUser --max-results 10`
- Check if user count is approaching organizational limits

## Phase 2 — Remediate

MUST:
- Activate pending users: `aws workdocs activate-user --user-id <user-id>`
- Fix user type if incorrectly assigned: `aws workdocs update-user --user-id <user-id> --type <USER|POWERUSER|ADMIN>`
- Ensure directory sync is current for new users

SHOULD:
- Set appropriate storage quotas for user type
- Configure user notification preferences

MAY:
- Bulk activate users via API script for large deployments
- Set up automated user provisioning based on directory group membership

## Safety Ratings

```
safety_ratings:
  - "workdocs describe-users: GREEN — read-only user listing"
  - "workdocs activate-user: YELLOW — activates a user account with cost implications"
  - "workdocs update-user: YELLOW — modifies user type and capabilities"
  - "workdocs deactivate-user: RED — disables user access and may affect content"
```

## Escalation Conditions

- User activation fails with internal service errors
- Users cannot be found in WorkDocs despite existing in the directory
- Bulk user operations fail or time out
- User type changes do not take effect

## Data Sensitivity

- `aws workdocs describe-users`: MEDIUM — reveals user identities, email, and storage usage
- `aws workdocs activate-user`: LOW — activates an account
- `aws cloudtrail lookup-events`: MEDIUM — reveals admin actions on user accounts

## Prohibited Actions

- NEVER deactivate users without explicit approval — this affects their content access
- NEVER change user type from ADMIN without verifying another admin exists
- NEVER bulk-activate users without understanding cost implications
- NEVER delete user accounts as a troubleshooting step

## Phase 3 — Rollback

1. If a user was activated in error, deactivate them: `aws workdocs deactivate-user --user-id <user-id>` (with approval)
2. If user type was changed incorrectly, revert: `aws workdocs update-user --user-id <user-id> --type <original-type>`
3. If storage quota was modified, restore original quota settings
4. Verify user can access their content after any changes
5. Document all user management changes for audit compliance

## Common Issues

- symptoms: "New employee cannot access WorkDocs"
  diagnosis: "User exists in directory but has not been activated in WorkDocs."
  resolution: "Activate the user: aws workdocs activate-user --user-id <user-id>."

- symptoms: "User cannot upload files despite being active"
  diagnosis: "User type is MINIMALUSER which has view-only access."
  resolution: "Upgrade user type to USER or POWERUSER: aws workdocs update-user --user-id <id> --type USER."

## Output Format

```yaml
root_cause: "user_management — <specific_cause>"
evidence:
  - type: user_issue
    content: "<user status, type, and directory details>"
severity: MEDIUM
mitigation:
  immediate: "Fix user account"
  long_term: "Automate user provisioning"
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
