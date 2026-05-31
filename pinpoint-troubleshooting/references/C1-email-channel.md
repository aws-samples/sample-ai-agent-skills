---
title: "C1 — Email Channel Configuration"
description: "Diagnose email channel setup and configuration issues"
status: active
severity: HIGH
triggers:
  - "email channel not working"
  - "cannot send email"
  - "email configuration error"
  - "SES identity not verified"
owner: devops-agent
objective: "Identify and resolve email channel configuration issues"
context: "Pinpoint email uses SES. Requires verified identity, proper configuration, and production access for full sending. Sandbox mode limits recipients to verified addresses only."
---

## Phase 1 — Triage

MUST:
- Check email channel: `aws pinpoint get-email-channel --application-id <app-id>`
- Verify SES identity: `aws ses get-identity-verification-attributes --identities <domain-or-email>`
- Check SES sending status: `aws ses get-account-sending-enabled`
- Check sandbox status: `aws ses get-account --query 'ProductionAccessEnabled'`

SHOULD:
- Verify sending quota: `aws ses get-send-quota`
- Check SES suppression list for bounced addresses
- Review SES configuration set if used

MAY:
- Check DKIM/SPF records for domain verification
- Review SES event destinations

## Phase 2 — Remediate

MUST:
- Verify sending identity (domain or email) in SES
- Request production access if in sandbox mode
- Configure proper From address matching verified identity

SHOULD:
- Set up DKIM for domain authentication
- Configure SES configuration set for tracking

MAY:
- Implement dedicated IP addresses for high-volume sending

## Common Issues

- symptoms: "Email channel enabled but no emails delivered"
  diagnosis: "SES identity not verified or account in sandbox mode."
  resolution: "Verify identity in SES. Request production access for non-verified recipients."

- symptoms: "Email sending returns MessageRejected"
  diagnosis: "From address doesn't match verified identity."
  resolution: "Use a From address under a verified domain or verified email."

## Output Format

```yaml
root_cause: "email_channel — <specific_cause>"
evidence:
  - type: channel_config
    content: "<email channel configuration>"
  - type: ses_identity
    content: "<SES verification status>"
severity: HIGH
mitigation:
  immediate: "Verify identity and check sandbox status"
  long_term: "Set up domain verification with DKIM/SPF"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — email channel handles customer email addresses
- Service Disruption Risk: MEDIUM — channel issues prevent email delivery
- Compliance Impact: HIGH — email channel must comply with CAN-SPAM, GDPR, and other regulations

## Escalation Conditions
- Email channel completely disabled
- Email sending identity verification fails
- Email channel configuration changes break delivery
- SES integration issues affect Pinpoint email
- Email channel quota exceeded

## Data Sensitivity
- Email addresses — RESTRICTED (PII)
- Email channel configurations — INTERNAL
- SES integration details — INTERNAL
- Sending identity configurations — INTERNAL
- Never expose customer email addresses

## Prohibited Actions
- Do NOT send emails from unverified identities
- Do NOT disable email channel without notifying stakeholders
- Do NOT bypass email authentication requirements
- Do NOT ignore email channel health warnings
- Do NOT share SES credentials
- Do NOT send emails without proper unsubscribe mechanisms

## Phase 3 — Rollback
- Restore email channel configurations
- Re-verify sending identities if needed
- Restore SES integration settings
- Verify email delivery after changes
- Restore suppression list configurations
- Document email channel issues and resolutions

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
