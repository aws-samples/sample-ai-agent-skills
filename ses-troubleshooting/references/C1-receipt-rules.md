---
title: "C1 — Receipt Rules"
description: "Diagnose SES email receiving and receipt rule issues"
status: active
severity: MEDIUM
triggers:
  - "receipt rule not working"
  - "incoming email not processed"
  - "email receiving failed"
  - "rule set inactive"
owner: devops-agent
objective: "Resolve SES email receiving and receipt rule configuration issues"
context: "SES email receiving processes incoming email through receipt rules. Rules are evaluated in order within the active rule set. Only the first matching rule processes each email. Receiving is only available in us-east-1, us-west-2, and eu-west-1."
---

## Phase 1 — Triage

MUST:
- Check active receipt rule set: `aws ses describe-active-receipt-rule-set`
- List receipt rules in the active set: `aws ses describe-active-receipt-rule-set --query 'Rules[].{Name:Name,Enabled:Enabled,Recipients:Recipients}'`
- Verify MX record points to SES: `dig MX <domain>`
- Confirm the domain is verified for receiving: `aws sesv2 get-email-identity --email-identity <domain>`

SHOULD:
- Check rule ordering (first match wins)
- Verify rule actions are properly configured
- Check if the region supports receiving (us-east-1, us-west-2, eu-west-1)

MAY:
- Check CloudWatch metrics for received emails
- Review SNS notifications for receipt rule processing

## Phase 2 — Remediate

MUST:
- Activate a receipt rule set if none is active: `aws ses set-active-receipt-rule-set --rule-set-name <name>`
- Configure MX record: `10 inbound-smtp.<region>.amazonaws.com`
- Ensure domain is verified in the receiving region

SHOULD:
- Order rules correctly (most specific first)
- Enable rules that are disabled
- Test with a simple rule before adding complex actions

MAY:
- Set up IP address filters for spam prevention
- Configure receipt rules with SNS notification for monitoring

## Common Issues

- symptoms: "Incoming emails not being processed"
  diagnosis: "No active receipt rule set or MX record not pointing to SES."
  resolution: "Activate rule set and configure MX record for the domain."

- symptoms: "Some emails processed, others not"
  diagnosis: "Rule ordering issue — earlier rule matching before intended rule."
  resolution: "Review rule order. Reorder rules: `aws ses reorder-receipt-rule-set --rule-set-name <name> --rule-names <ordered-list>`"

- symptoms: "Receipt rules not available in region"
  diagnosis: "SES receiving only supported in us-east-1, us-west-2, eu-west-1."
  resolution: "Set up receiving in a supported region."

## Output Format

```yaml
root_cause: "receipt_rules — <specific_cause>"
evidence:
  - type: rule_set
    content: "<active rule set and rules>"
  - type: dns_config
    content: "<MX record configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix rule set activation, MX records, or rule ordering"
  long_term: "Monitor receipt rule processing and set up alerting"
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
