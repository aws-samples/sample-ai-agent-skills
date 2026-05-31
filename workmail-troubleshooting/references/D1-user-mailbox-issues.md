---
title: "D1 — User Mailbox Issues"
description: "Diagnose WorkMail user mailbox issues"
status: active
severity: MEDIUM
triggers: ["mailbox full", "mailbox not working", "cannot access mailbox", "mailbox disabled"]
owner: devops-agent
objective: "Resolve user mailbox issues including capacity, access, configuration, and mailbox export"
context: "Each user mailbox has a 50 GB limit. Mailboxes are tied to directory users. Disabled mailboxes cannot send or receive."
---

## Phase 1 — Triage

MUST:
- Check user and mailbox status: `aws workmail describe-user --organization-id <org-id> --user-id <user-id>`
- List users to find the user ID: `aws workmail list-users --organization-id <org-id> --query 'Users[?Name==`<name>`]'`
- Verify user state is ENABLED
- Check if the mailbox is approaching the 50 GB limit

SHOULD:
- Check if the user has email aliases configured: `aws workmail list-aliases --organization-id <org-id> --entity-id <user-id>`
- Verify the user exists in the directory
- Check mailbox export jobs if data export was requested: `aws workmail list-mailbox-export-jobs --organization-id <org-id>`

MAY:
- Review user activity in CloudTrail
- Check if retention policies are affecting mailbox content

## Phase 2 — Remediate

MUST:
- Enable disabled mailboxes: `aws workmail register-to-work-mail --organization-id <org-id> --entity-id <user-id> --email <email>`
- Free mailbox space by deleting old email or emptying trash
- Fix email aliases if misconfigured

SHOULD:
- Set up mailbox size alerts before reaching the 50 GB limit
- Configure retention policies to manage mailbox growth

MAY:
- Export mailbox for backup: `aws workmail start-mailbox-export-job --organization-id <org-id> --entity-id <user-id> --s3-bucket-name <bucket> --s3-prefix <prefix> --role-arn <role-arn>`
- Set up auto-archive rules for old email

## Safety Ratings

```
safety_ratings:
  - "workmail describe-user: GREEN — read-only user inspection"
  - "workmail list-aliases: GREEN — read-only alias listing"
  - "workmail register-to-work-mail: YELLOW — enables mailbox for a user"
  - "workmail deregister-from-work-mail: RED — disables mailbox, email stops flowing"
```

## Escalation Conditions

- Mailbox is at 50 GB limit and user cannot delete enough email
- Mailbox export job fails repeatedly
- User mailbox is corrupted and cannot be accessed
- Mailbox deregistration was accidental and email was lost

## Data Sensitivity

- `aws workmail describe-user`: MEDIUM — reveals user details
- `aws workmail list-aliases`: LOW — reveals email aliases
- Mailbox export: HIGH — contains all user email content
- `aws workmail start-mailbox-export-job`: HIGH — exports full mailbox to S3

## Prohibited Actions

- NEVER deregister a user's mailbox without explicit approval — this stops all email
- NEVER access mailbox export data without authorization
- NEVER delete email from a user's mailbox without their consent
- NEVER change email aliases without verifying the impact on email delivery

## Phase 3 — Rollback

1. If a mailbox was deregistered in error, re-register it: `aws workmail register-to-work-mail --organization-id <org-id> --entity-id <user-id> --email <email>` (note: email received during deregistration is lost)
2. If email aliases were changed, restore original aliases
3. If retention policies were modified, revert to original settings
4. If a mailbox export was started in error, delete the export from S3
5. Verify the user can send and receive email after changes

## Common Issues

- symptoms: "User cannot receive email — mailbox full"
  diagnosis: "Mailbox has reached the 50 GB limit."
  resolution: "Delete old email, empty trash and deleted items. Consider exporting old email to S3 for archival."

- symptoms: "User exists in directory but has no mailbox"
  diagnosis: "User was not registered for WorkMail."
  resolution: "Register the user: aws workmail register-to-work-mail with the user's email address."

## Output Format

```yaml
root_cause: "mailbox_issue — <specific_cause>"
evidence:
  - type: mailbox_error
    content: "<user status, mailbox size, configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix mailbox issue"
  long_term: "Implement mailbox size monitoring and retention policies"
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
