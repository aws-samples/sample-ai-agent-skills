---
title: "B2 — Receiving Issues"
description: "Diagnose WorkMail inbound email receiving issues"
status: active
severity: HIGH
triggers: ["not receiving email", "email missing", "inbound email failed", "email not arriving"]
owner: devops-agent
objective: "Resolve inbound email receiving issues including MX records, email flow rules, and mailbox configuration"
context: "Inbound email requires correct MX records, active user mailbox, and no blocking email flow rules or access control rules."
---

## Phase 1 — Triage

MUST:
- Verify MX records point to WorkMail: `dig MX <domain>`
- Check user mailbox is enabled: `aws workmail describe-user --organization-id <org-id> --user-id <user-id>`
- Verify user state is ENABLED and mailbox is not full
- Check email flow rules for inbound blocking: `aws workmail list-mail-flow-rules --organization-id <org-id>` (if available)

SHOULD:
- Check if the sender is on any block list
- Verify the email address alias is correctly configured
- Check retention policies that may have deleted the email

MAY:
- Ask sender for bounce-back message details
- Check CloudTrail for inbound email processing events
- Verify no access control rules are blocking the sender's IP or protocol

## Phase 2 — Remediate

MUST:
- Fix MX records if incorrect or missing
- Enable user mailbox if disabled
- Remove or modify blocking email flow rules

SHOULD:
- Free mailbox space if near the 50 GB limit
- Verify email aliases are correctly mapped to the user

MAY:
- Configure email forwarding as a temporary workaround
- Set up message tracking for ongoing monitoring

## Safety Ratings

```
safety_ratings:
  - "dig MX: GREEN — read-only DNS query"
  - "workmail describe-user: GREEN — read-only user inspection"
  - "workmail list-mail-flow-rules: GREEN — read-only rule listing"
  - "DNS MX record changes: RED — affects email routing for the domain"
```

## Escalation Conditions

- MX records are correct but email is not being received by WorkMail
- Email from specific senders consistently fails while others succeed
- Mailbox is not full and no flow rules block, but email still missing
- Inbound email failures affect all users in the organization

## Data Sensitivity

- `aws workmail describe-user`: MEDIUM — reveals user mailbox details
- MX records: LOW — publicly queryable
- Email flow rules: MEDIUM — reveals email filtering configuration
- Bounce-back messages: MEDIUM — may contain sender and recipient details

## Prohibited Actions

- NEVER delete email flow rules without understanding their purpose
- NEVER change MX records without a rollback plan
- NEVER access another user's mailbox to check for received email without authorization
- NEVER disable email flow rules organization-wide to troubleshoot one user

## Phase 3 — Rollback

1. If MX records were changed, restore original records immediately if email stops flowing
2. If email flow rules were modified, restore original rules
3. If email aliases were changed, revert to original configuration
4. Monitor inbound email delivery for 24 hours after changes
5. Document the receiving issue and resolution for future reference

## Common Issues

- symptoms: "No email received for custom domain"
  diagnosis: "MX records not pointing to WorkMail or still pointing to previous email provider."
  resolution: "Update MX record to: 10 inbound-smtp.<region>.amazonaws.com. Remove old MX records."

- symptoms: "Some emails received but others missing"
  diagnosis: "Email flow rules may be filtering specific senders or subjects."
  resolution: "Review email flow rules. Check if retention policies deleted the email. Check spam/junk folder."

## Output Format

```yaml
root_cause: "receiving_issue — <specific_cause>"
evidence:
  - type: inbound_error
    content: "<MX records, user status, flow rules>"
severity: HIGH
mitigation:
  immediate: "Fix receiving configuration"
  long_term: "Monitor inbound email flow and set up alerts"
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
