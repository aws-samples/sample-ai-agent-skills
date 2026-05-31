---
title: "C3 — Email Deliverability Issues"
description: "Diagnose email deliverability and inbox placement problems"
status: active
severity: HIGH
triggers:
  - "emails going to spam"
  - "low open rates"
  - "deliverability issues"
  - "inbox placement"
owner: devops-agent
objective: "Identify and resolve email deliverability issues"
context: "Email deliverability depends on sender reputation, authentication (DKIM/SPF/DMARC), content quality, and sending patterns. Poor deliverability means emails land in spam."
---

## Phase 1 — Triage

MUST:
- Check DKIM status: `aws ses get-identity-dkim-attributes --identities <domain>`
- Verify email channel config: `aws pinpoint get-email-channel --application-id <app-id>`
- Check sending statistics: `aws ses get-send-statistics`
- Review bounce/complaint rates: `aws cloudwatch get-metric-statistics --namespace AWS/SES --metric-name Reputation.BounceRate --start-time <start> --end-time <end> --period 86400 --statistics Average`

SHOULD:
- Verify SPF record exists for sending domain
- Check DMARC policy for the domain
- Review email content for spam trigger words

MAY:
- Use SES Deliverability Dashboard (if available)
- Check IP reputation if using dedicated IPs

## Phase 2 — Remediate

MUST:
- Enable DKIM signing for the sending domain
- Add SPF record to DNS
- Maintain bounce rate below 5% and complaint rate below 0.1%

SHOULD:
- Implement DMARC policy
- Warm up new sending domains/IPs gradually
- Segment engaged vs unengaged recipients

MAY:
- Use dedicated IPs for reputation isolation
- Implement feedback loop processing

## Common Issues

- symptoms: "Emails delivered but going to spam folder"
  diagnosis: "Missing DKIM/SPF authentication or poor sender reputation."
  resolution: "Enable DKIM. Add SPF record. Improve sending practices."

- symptoms: "Open rates suddenly dropped"
  diagnosis: "Reputation degradation or ISP blocking."
  resolution: "Check bounce/complaint rates. Clean list. Reduce sending volume temporarily."

## Output Format

```yaml
root_cause: "email_deliverability — <specific_cause>"
evidence:
  - type: dkim_status
    content: "<DKIM verification status>"
  - type: reputation_metrics
    content: "<bounce and complaint rates>"
severity: HIGH
mitigation:
  immediate: "Fix authentication and clean sending list"
  long_term: "Implement full email authentication and list hygiene"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — email content may contain personalized data
- Service Disruption Risk: MEDIUM — deliverability issues affect campaign effectiveness
- Compliance Impact: HIGH — email deliverability affects CAN-SPAM and GDPR compliance

## Escalation Conditions
- Email deliverability rate drops below acceptable thresholds
- Domain reputation degraded
- IP address blacklisted
- DKIM/SPF/DMARC authentication failures
- Email sending suspended by SES

## Data Sensitivity
- Email addresses — RESTRICTED (PII)
- Email content — INTERNAL
- Domain configurations — INTERNAL
- Deliverability metrics — INTERNAL
- Never expose recipient email addresses in logs

## Prohibited Actions
- Do NOT send emails to unverified or purchased email lists
- Do NOT ignore bounce and complaint rates
- Do NOT bypass email authentication (DKIM/SPF/DMARC)
- Do NOT send emails without proper unsubscribe mechanisms
- Do NOT ignore domain reputation warnings
- Do NOT send emails from unverified domains

## Phase 3 — Rollback
- Restore previous email sending configurations
- Re-verify domain authentication records
- Restore suppression list configurations
- Reduce sending volume to recover reputation
- Verify deliverability metrics after changes
- Document deliverability issues and remediation steps

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
