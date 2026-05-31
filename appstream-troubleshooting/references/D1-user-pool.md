---
title: "D1 — User Pool Issues"
description: "Diagnose AppStream 2.0 user pool management problems"
status: active
severity: MEDIUM
triggers:
  - "user pool error"
  - "cannot create user"
  - "user authentication failed"
  - "user pool management"
owner: devops-agent
objective: "Identify and resolve user pool management issues"
context: "AppStream user pool provides built-in user management with email-based accounts. Users receive welcome emails with temporary passwords. User pool is an alternative to SAML federation. They cannot be used together on the same stack."
---

## Phase 1 — Triage

MUST:
- List users: `aws appstream describe-users --authentication-type USERPOOL`
- Check if the user exists and is enabled
- Verify the stack uses USERPOOL authentication
- Check if the user has been assigned to the stack

SHOULD:
- Verify the user's email address is correct
- Check if the welcome email was delivered
- Verify the user's status (CONFIRMED, FORCE_CHANGE_PASSWORD, etc.)

MAY:
- Check CloudTrail for user management events
- Verify email delivery settings

## Phase 2 — Remediate

MUST:
- Create user if missing: `aws appstream create-user --user-name <email> --authentication-type USERPOOL`
- Enable disabled users: `aws appstream enable-user --user-name <email> --authentication-type USERPOOL`
- Assign user to stack: `aws appstream batch-associate-user-stack --user-stack-associations StackName=<stack>,UserName=<email>,AuthenticationType=USERPOOL,SendEmailNotification=true`

SHOULD:
- Resend welcome email if user didn't receive it
- Reset user password if needed

MAY:
- Set up user management automation
- Document user onboarding process

## Common Issues

- symptoms: "User cannot log in"
  diagnosis: "User not created, not enabled, or not assigned to the stack."
  resolution: "Create user, enable, and assign to the stack."

- symptoms: "User didn't receive welcome email"
  diagnosis: "Email delivery issue or incorrect email address."
  resolution: "Verify email address. Resend by disabling and re-enabling the user."

- symptoms: "User gets 'not authorized' error"
  diagnosis: "User not assigned to the stack."
  resolution: "Associate user with the stack: batch-associate-user-stack."

## Output Format

```yaml
root_cause: "user_pool_issue — <specific_cause>"
evidence:
  - type: user_status
    content: "<user existence and status>"
  - type: stack_association
    content: "<user-stack association>"
  - type: authentication_type
    content: "<stack authentication configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix user configuration"
  long_term: "Automate user management"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Create user | GREEN | New user creation |
| Enable disabled users | GREEN | User management |
| Assign user to stack | GREEN | User-stack association |
| Resend welcome email | GREEN | Email notification |
| Reset user password | YELLOW | Credential change |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **User data**: Email addresses, user status — CONFIDENTIAL
- **Stack associations**: User-to-stack mappings — INTERNAL
- **Authentication config**: User pool settings — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER delete user accounts without confirmation
- NEVER share user credentials or temporary passwords

## Phase 3 — Rollback

1. Disable newly created users if they were created in error
2. Remove user-stack associations if they were incorrectly assigned
3. Restore previous user pool configuration
4. Verify existing users are not affected
5. Communicate changes to affected users

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
