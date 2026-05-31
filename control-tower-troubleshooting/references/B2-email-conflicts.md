---
title: "B2 — Account Email Conflicts"
description: "Diagnose account provisioning failures due to email address conflicts"
status: active
severity: HIGH
triggers:
  - "Email already in use"
  - "Email address already associated"
  - "Account email conflict"
owner: devops-agent
objective: "Resolve email conflicts and provision the account successfully"
context: "AWS account root email addresses must be globally unique across all AWS accounts worldwide. Closed accounts retain their email for 90 days. Email aliases (+tag format) can create unique addresses from a single mailbox."
---

## Phase 1 — Triage

MUST:
- Confirm the error is email-related from the Service Catalog record or CloudFormation events
- Check if the email is already associated with an existing AWS account
- Check if the email was used by a recently closed account (90-day retention)
- Verify the email format is valid and the mailbox is accessible

SHOULD:
- Check Organizations for accounts with the same email: `aws organizations list-accounts`
- Verify the email domain supports aliases (+tag format)

MAY:
- Check if the email is associated with an account in a different Organization

## Phase 2 — Remediate

MUST:
- Use a unique email address that is not associated with any AWS account
- If using aliases: use format like admin+projectname@example.com
- Terminate the failed provisioned product before retrying

SHOULD:
- Establish an email naming convention for account provisioning
- Use a distribution list or shared mailbox for account root emails
- Document the email-to-account mapping

MAY:
- Set up a dedicated email domain for AWS account management
- Use email sub-addressing for scalable unique addresses

## Common Issues

- symptoms: "CreateAccount fails with email already exists"
  diagnosis: "The email is already the root email of another AWS account."
  resolution: "Use a different unique email. Check all Organizations for the existing account."

- symptoms: "Email was used by a closed account"
  diagnosis: "Closed accounts retain their email for 90 days."
  resolution: "Wait 90 days or use a different email address."

## Safety Ratings

```
safety_ratings:
  - "list-accounts: GREEN — read-only account listing"
  - "describe-provisioned-product: GREEN — read-only product status"
  - "terminate failed provisioned product: YELLOW — recoverable, cleans up failed provisioning"
  - "re-provision with new email: YELLOW — creates new account, original failed product is cleaned up"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Email is associated with an account in a different Organization
- Closed account email retention (90-day wait) blocks urgent provisioning
- Fix requires moving the Audit or Log Archive account

## Data Sensitivity

- HIGH: Organization account list (all account IDs and emails)
- HIGH: Account root email addresses (PII, expose account mapping)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Email naming conventions (expose organizational patterns)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If provisioning was attempted with a conflicting email, terminate the failed provisioned product before retrying with a new email
- Document the email-to-account mapping before making changes
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"

## Output Format

```yaml
root_cause: "email_conflict — <email> already in use"
evidence:
  - type: organizations_api
    content: "<account listing showing conflict>"
severity: HIGH
mitigation:
  immediate: "Use a unique email address and retry"
  long_term: "Establish email naming convention and tracking system"
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
