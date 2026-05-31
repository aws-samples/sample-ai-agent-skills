---
title: "B1 — Email Delivery Failures"
description: "Diagnose WorkMail outbound email delivery failures"
status: active
severity: HIGH
triggers: ["email not delivered", "send failed", "bounce", "delivery failure", "email rejected"]
owner: devops-agent
objective: "Resolve outbound email delivery failures including SES limits, reputation, and configuration issues"
context: "WorkMail uses SES for outbound delivery. SES sending limits, reputation, and suppression lists affect delivery."
---

## Phase 1 — Triage

MUST:
- Verify organization is active: `aws workmail describe-organization --organization-id <org-id>`
- Check domain verification status: `aws workmail get-mail-domain --organization-id <org-id> --domain-name <domain>`
- Check SES sending quota: `aws ses get-send-quota`
- Check SES account status: `aws ses get-account`

SHOULD:
- Check SES suppression list for the recipient: `aws sesv2 get-suppressed-destination --email-address <recipient>`
- Verify DKIM is configured and passing
- Check email flow rules: `aws workmail list-mail-flow-rules --organization-id <org-id>` (if available)

MAY:
- Review SES sending statistics: `aws ses get-send-statistics`
- Check CloudTrail for email-related events
- Verify SPF and DMARC records for the sending domain

## Phase 2 — Remediate

MUST:
- Fix domain verification if incomplete (MX, TXT, DKIM records)
- Remove recipient from SES suppression list if bounce was transient
- Fix email flow rules if they are blocking outbound email

SHOULD:
- Request SES sending limit increase if quota is reached
- Improve sender reputation by reducing bounce and complaint rates
- Configure SPF and DMARC for better deliverability

MAY:
- Set up SES event notifications for bounce and complaint tracking
- Implement email validation before sending to reduce bounces

## Safety Ratings

```
safety_ratings:
  - "workmail describe-organization: GREEN — read-only organization inspection"
  - "ses get-send-quota: GREEN — read-only quota check"
  - "sesv2 delete-suppressed-destination: YELLOW — removes address from suppression list"
  - "DNS record changes: RED — affects email authentication for the domain"
```

## Escalation Conditions

- SES account is in sandbox mode and production access request is pending
- SES account is under review or suspended due to reputation issues
- Email delivery fails to specific domains suggesting blocklisting
- Outbound email volume exceeds SES limits and increase request is denied

## Data Sensitivity

- `aws workmail describe-organization`: LOW — returns organization metadata
- `aws ses get-send-quota`: LOW — returns sending limits
- `aws ses get-send-statistics`: MEDIUM — reveals email volume patterns
- Email content and headers: HIGH — contains message content and recipient information

## Prohibited Actions

- NEVER send test emails to external addresses without approval
- NEVER remove addresses from the SES suppression list without verifying the bounce was transient
- NEVER modify SES configuration that affects other services using the same SES account
- NEVER disable DKIM or SPF to troubleshoot delivery issues

## Phase 3 — Rollback

1. If email flow rules were modified, restore original rules
2. If SES suppression list was modified, re-add addresses if removal was incorrect
3. If DNS records were changed for SPF/DKIM, restore original records
4. Verify outbound email delivery is working after changes
5. Document delivery failure patterns for ongoing monitoring

## Common Issues

- symptoms: "Outbound email bounces with 'Message rejected'"
  diagnosis: "SES sending quota exceeded or account in sandbox mode."
  resolution: "Check ses get-send-quota. Request production access or sending limit increase."

- symptoms: "Email delivered but lands in recipient's spam folder"
  diagnosis: "Missing or incorrect SPF, DKIM, or DMARC records."
  resolution: "Configure SPF TXT record, verify DKIM CNAME records, and set up DMARC policy."

## Output Format

```yaml
root_cause: "email_delivery — <specific_cause>"
evidence:
  - type: delivery_error
    content: "<bounce details, SES status, DNS records>"
severity: HIGH
mitigation:
  immediate: "Fix delivery issue"
  long_term: "Monitor SES reputation and delivery metrics"
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
