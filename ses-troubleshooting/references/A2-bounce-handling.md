---
title: "A2 — Bounce Handling"
description: "Diagnose and manage email bounce issues"
status: active
severity: HIGH
triggers:
  - "high bounce rate"
  - "email bounced"
  - "hard bounce"
  - "soft bounce"
owner: devops-agent
objective: "Identify bounce causes and implement proper bounce handling"
context: "Bounces occur when email cannot be delivered. Hard bounces (permanent) indicate invalid addresses. Soft bounces (transient) are temporary failures. Bounce rate above 5% triggers account review. SES automatically adds hard-bounced addresses to the suppression list."
---

## Phase 1 — Triage

MUST:
- Check current bounce rate: `aws cloudwatch get-metric-statistics --namespace AWS/SES --metric-name Reputation.BounceRate --start-time <iso> --end-time <iso> --period 86400 --statistics Average`
- Check account reputation: `aws sesv2 get-account --query '{BounceRate:Details.ReviewDetails}'`
- List suppressed destinations: `aws sesv2 list-suppressed-destinations --reasons BOUNCE --start-date <iso>`
- Check if bounce notifications are configured

SHOULD:
- Review bounce notification SNS topic for recent bounces
- Check configuration set event destination for bounce events: `aws sesv2 get-configuration-set-event-destinations --configuration-set-name <name>`
- Identify top bouncing recipient domains

MAY:
- Analyze bounce types (hard vs soft) from SNS notifications
- Check if mailing list has been cleaned recently

## Phase 2 — Remediate

MUST:
- Remove invalid addresses from mailing lists immediately
- Set up bounce notification handling (SNS topic or event destination)
- Monitor bounce rate to keep below 5%

SHOULD:
- Implement double opt-in for new subscribers
- Process bounce notifications automatically to remove invalid addresses
- Use SES suppression list to prevent re-sending to bounced addresses

MAY:
- Implement email validation before adding to mailing list
- Set up CloudWatch alarms for bounce rate threshold

## Common Issues

- symptoms: "Bounce rate above 5%"
  diagnosis: "Sending to stale mailing list with many invalid addresses."
  resolution: "Clean mailing list. Remove addresses that bounced. Implement bounce processing."

- symptoms: "Soft bounces for specific domain"
  diagnosis: "Recipient mail server temporarily rejecting (rate limiting, full mailbox)."
  resolution: "SES retries soft bounces automatically. If persistent, check recipient domain reputation."

- symptoms: "Account under review for bounces"
  diagnosis: "Bounce rate exceeded 5% threshold."
  resolution: "Immediately stop sending to unvalidated lists. Clean addresses. Respond to AWS review."

## Output Format

```yaml
root_cause: "bounce_handling — <specific_cause>"
evidence:
  - type: bounce_rate
    content: "<current bounce rate and trend>"
  - type: suppression_list
    content: "<recently suppressed addresses>"
severity: HIGH
mitigation:
  immediate: "Clean mailing list and set up bounce processing"
  long_term: "Implement double opt-in and automated bounce handling"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Email content may contain customer PII and business communications
- Recipient lists contain personal email addresses
- Bounce and complaint data reveals delivery issues with specific addresses
- Classification: HIGHLY CONFIDENTIAL - email content and recipient PII

## Prohibited Actions

- NEVER send test emails to real customer addresses
- NEVER modify production DKIM/SPF/DMARC records without email team approval
- NEVER disable bounce or complaint handling in production
- NEVER remove addresses from the suppression list without investigating the original cause

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest disabling DKIM/SPF/DMARC"
  - "NEVER suggest removing sending limits"
  - "NEVER suggest disabling bounce/complaint handling"
