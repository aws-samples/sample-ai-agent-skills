---
title: "D2 — Admin Controls"
description: "Diagnose WorkDocs admin control and policy issues"
status: active
severity: MEDIUM
triggers: ["admin settings", "policy not working", "cannot configure", "admin panel error"]
owner: devops-agent
objective: "Resolve admin control issues including security policies, sharing settings, and organizational configuration"
context: "WorkDocs admins configure sharing policies, storage quotas, IP restrictions, and recovery settings through the admin console or API."
---

## Phase 1 — Triage

MUST:
- Verify caller has admin permissions: `aws workdocs describe-users --organization-id <org-id> --query 'Users[?Username==`<admin-username>`].Type'`
- Check current organization settings through the WorkDocs admin console
- Verify the organization ID is correct: `aws workdocs describe-users --organization-id <org-id> --limit 1`

SHOULD:
- Review current sharing policies (public sharing, external sharing)
- Check storage allocation and usage across the organization
- Verify IP allow/deny list configuration

MAY:
- Review CloudTrail for admin configuration changes
- Check if recent policy changes correlate with reported issues

## Phase 2 — Remediate

MUST:
- Fix admin policy settings through the WorkDocs admin console
- Ensure at least one ADMIN user exists in the organization
- Correct sharing policies if they are blocking legitimate collaboration

SHOULD:
- Document all admin policy settings for disaster recovery
- Configure recovery settings (Recycle Bin retention period)

MAY:
- Set up admin notifications for policy violations
- Create admin runbooks for common configuration tasks

## Safety Ratings

```
safety_ratings:
  - "workdocs describe-users: GREEN — read-only user listing"
  - "Admin console policy changes: YELLOW — affects all users in the organization"
  - "IP allow/deny list changes: RED — can lock out users if misconfigured"
  - "Storage quota changes: YELLOW — affects user upload capabilities"
```

## Escalation Conditions

- Admin console is inaccessible and no API alternative exists
- Policy changes lock out all users including admins
- Storage allocation exceeds organizational budget
- Security policies conflict with compliance requirements

## Data Sensitivity

- `aws workdocs describe-users`: MEDIUM — reveals user identities and types
- Admin console settings: HIGH — reveals organizational security policies
- IP allow/deny lists: MEDIUM — reveals network access patterns

## Prohibited Actions

- NEVER disable all security policies to troubleshoot access issues
- NEVER remove the last ADMIN user from the organization
- NEVER change IP restrictions without a rollback plan
- NEVER modify Recycle Bin retention to 0 days — this prevents content recovery

## Phase 3 — Rollback

1. If sharing policies were changed, restore original settings through admin console
2. If IP restrictions were modified, revert to original allow/deny lists immediately if users are locked out
3. If storage quotas were changed, restore original per-user quotas
4. If admin user was demoted, re-promote them: `aws workdocs update-user --user-id <id> --type ADMIN`
5. Document all admin changes and notify affected users

## Common Issues

- symptoms: "Users cannot share documents externally"
  diagnosis: "Admin policy has disabled external sharing."
  resolution: "Enable external sharing in admin console if appropriate for the organization."

- symptoms: "Admin cannot access admin console"
  diagnosis: "User type was changed from ADMIN or account was deactivated."
  resolution: "Use another admin account or AWS root/IAM to restore admin access."

## Output Format

```yaml
root_cause: "admin_controls — <specific_cause>"
evidence:
  - type: admin_issue
    content: "<policy settings and configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix admin configuration"
  long_term: "Document and version admin policies"
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
