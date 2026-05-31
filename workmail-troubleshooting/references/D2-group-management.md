---
title: "D2 — Group Management"
description: "Diagnose WorkMail group and distribution list issues"
status: active
severity: MEDIUM
triggers: ["group email not working", "distribution list error", "cannot add member", "group delivery failed"]
owner: devops-agent
objective: "Resolve group management issues including creation, membership, delivery, and email aliases"
context: "WorkMail groups function as distribution lists. Email sent to a group is delivered to all members. Groups do not have their own mailbox."
---

## Phase 1 — Triage

MUST:
- List groups: `aws workmail list-groups --organization-id <org-id>`
- Check group details: `aws workmail describe-group --organization-id <org-id> --group-id <group-id>`
- List group members: `aws workmail list-group-members --organization-id <org-id> --group-id <group-id>`
- Verify group state is ENABLED

SHOULD:
- Check group email aliases: `aws workmail list-aliases --organization-id <org-id> --entity-id <group-id>`
- Verify all group members have active mailboxes
- Check if the group has reached member limits

MAY:
- Test sending email to the group from an internal user
- Review email flow rules that may affect group delivery

## Phase 2 — Remediate

MUST:
- Enable disabled groups: `aws workmail register-to-work-mail --organization-id <org-id> --entity-id <group-id> --email <group-email>`
- Add missing members: `aws workmail associate-member-to-group --organization-id <org-id> --group-id <group-id> --member-id <user-id>`
- Fix group email alias if incorrect

SHOULD:
- Remove inactive members from the group
- Verify group email address is on the correct domain

MAY:
- Create nested groups for organizational hierarchy
- Set up group management delegation

## Safety Ratings

```
safety_ratings:
  - "workmail list-groups: GREEN — read-only group listing"
  - "workmail describe-group: GREEN — read-only group inspection"
  - "workmail associate-member-to-group: YELLOW — adds member to distribution list"
  - "workmail disassociate-member-from-group: YELLOW — removes member from distribution list"
```

## Escalation Conditions

- Group email delivery fails for all members
- Group member limit reached and cannot be increased
- Group email causes mail loops or excessive delivery
- Group management permissions are insufficient for the admin

## Data Sensitivity

- `aws workmail list-groups`: LOW — returns group names and IDs
- `aws workmail list-group-members`: MEDIUM — reveals group membership
- `aws workmail describe-group`: LOW — returns group metadata
- Group email content: HIGH — delivered to all members

## Prohibited Actions

- NEVER delete a group without notifying all members
- NEVER add external email addresses to groups without approval
- NEVER create groups that could cause mail loops (group A includes group B which includes group A)
- NEVER remove members from groups without verifying the change is authorized

## Phase 3 — Rollback

1. If members were removed from a group, re-add them: `aws workmail associate-member-to-group`
2. If a group was deregistered, re-register it with the original email address
3. If group aliases were changed, restore original aliases
4. If a group was deleted, recreate it and re-add all members
5. Verify group email delivery works correctly after changes

## Common Issues

- symptoms: "Email sent to group not delivered to all members"
  diagnosis: "Some group members have disabled or full mailboxes."
  resolution: "Check each member's mailbox status. Enable disabled mailboxes. Free space in full mailboxes."

- symptoms: "Cannot add member to group"
  diagnosis: "User is not registered for WorkMail or group has reached member limit."
  resolution: "Register the user for WorkMail first. Check group member limits."

## Output Format

```yaml
root_cause: "group_management — <specific_cause>"
evidence:
  - type: group_issue
    content: "<group status, membership, delivery details>"
severity: MEDIUM
mitigation:
  immediate: "Fix group configuration"
  long_term: "Implement group membership review process"
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
