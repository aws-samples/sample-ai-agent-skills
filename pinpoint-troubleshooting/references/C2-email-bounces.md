---
title: "C2 — Email Bounces and Complaints"
description: "Diagnose email bounce and complaint issues"
status: active
severity: HIGH
triggers:
  - "high bounce rate"
  - "email bouncing"
  - "complaint rate high"
  - "email reputation"
owner: devops-agent
objective: "Identify and resolve email bounce and complaint issues"
context: "High bounce and complaint rates damage sender reputation and can lead to SES sending suspension. Pinpoint tracks bounces and complaints through SES."
---

## Phase 1 — Triage

MUST:
- Check bounce metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Pinpoint --metric-name EmailBounceRate --dimensions Name=ApplicationId,Value=<app-id> --start-time <start> --end-time <end> --period 86400 --statistics Average`
- Check SES reputation: `aws ses get-account --query '{ReputationMetrics:EnforcementStatus}'`
- Check SES suppression list: `aws sesv2 list-suppressed-destinations --reasons BOUNCE COMPLAINT --start-date <date>`
- Review sending statistics: `aws ses get-send-statistics`

SHOULD:
- Check complaint rate (should be below 0.1%)
- Verify bounce rate (should be below 5%)
- Review SES account-level suppression list settings

MAY:
- Check specific bounced addresses
- Review email content for spam triggers

## Phase 2 — Remediate

MUST:
- Remove hard-bounced addresses from endpoints
- Address complaint sources (add easy unsubscribe)
- Monitor reputation dashboard

SHOULD:
- Implement email validation before adding endpoints
- Use double opt-in for new subscribers
- Set up bounce/complaint notifications via SNS

MAY:
- Implement dedicated IP with warm-up plan
- Use SES configuration sets for per-campaign tracking

## Common Issues

- symptoms: "Bounce rate suddenly spikes"
  diagnosis: "Sending to stale list with invalid addresses."
  resolution: "Clean endpoint list. Remove addresses that bounced. Implement validation."

- symptoms: "SES sending paused"
  diagnosis: "Bounce or complaint rate exceeded SES thresholds."
  resolution: "Address bounce/complaint sources. Contact AWS Support to restore sending."

## Output Format

```yaml
root_cause: "email_bounces — <specific_cause>"
evidence:
  - type: bounce_rate
    content: "<bounce rate metrics>"
  - type: ses_reputation
    content: "<SES reputation status>"
severity: HIGH
mitigation:
  immediate: "Clean endpoint list and address complaints"
  long_term: "Implement email validation and double opt-in"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — bounce data contains email addresses
- Service Disruption Risk: MEDIUM — high bounce rates can lead to sending suspension
- Compliance Impact: HIGH — bounce handling affects sender reputation and compliance

## Escalation Conditions
- Bounce rate exceeds acceptable thresholds
- Hard bounce rate triggers SES account review
- Bounce processing fails causing continued sends to invalid addresses
- Suppression list not being updated with bounced addresses
- Bounce notifications not being delivered

## Data Sensitivity
- Bounced email addresses — RESTRICTED (PII)
- Bounce reasons — INTERNAL
- Suppression list — RESTRICTED
- Sender reputation metrics — INTERNAL
- Never expose bounced email addresses in logs

## Prohibited Actions
- Do NOT continue sending to hard-bounced addresses
- Do NOT ignore bounce rate warnings
- Do NOT remove addresses from suppression list without verification
- Do NOT bypass bounce handling procedures
- Do NOT share bounce reports containing email addresses
- Do NOT ignore SES account review notifications

## Phase 3 — Rollback
- Restore suppression list configurations
- Re-enable bounce notification processing
- Restore previous sending configurations
- Verify bounce handling after changes
- Document bounce issues and sender reputation impact
- Update email list hygiene procedures

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
