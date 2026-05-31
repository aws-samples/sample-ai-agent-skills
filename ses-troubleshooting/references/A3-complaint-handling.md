---
title: "A3 — Complaint Handling"
description: "Diagnose and manage email complaint issues"
status: active
severity: CRITICAL
triggers:
  - "high complaint rate"
  - "spam complaint"
  - "feedback loop"
  - "complaint notification"
owner: devops-agent
objective: "Identify complaint causes and implement proper complaint handling"
context: "Complaints occur when recipients mark email as spam. Complaint rate above 0.1% triggers account review. This is a much stricter threshold than bounce rate. ISPs report complaints via feedback loops. SES automatically adds complained addresses to the suppression list."
---

## Phase 1 — Triage

MUST:
- Check current complaint rate: `aws cloudwatch get-metric-statistics --namespace AWS/SES --metric-name Reputation.ComplaintRate --start-time <iso> --end-time <iso> --period 86400 --statistics Average`
- Check account status: `aws sesv2 get-account --query '{SendingEnabled:SendingEnabled,EnforcementStatus:EnforcementStatus}'`
- List suppressed destinations from complaints: `aws sesv2 list-suppressed-destinations --reasons COMPLAINT --start-date <iso>`
- Verify complaint notification handling is configured

SHOULD:
- Review complaint notification details from SNS topic
- Identify which email campaigns generated complaints
- Check if unsubscribe links are present and functional

MAY:
- Analyze complaint patterns by recipient domain
- Review email content for spam-like characteristics

## Phase 2 — Remediate

MUST:
- Immediately remove complaining addresses from all mailing lists
- Set up complaint notification processing (SNS or event destination)
- Ensure all marketing emails include working unsubscribe links

SHOULD:
- Implement one-click unsubscribe (List-Unsubscribe header)
- Review email content and sending frequency
- Segment mailing lists to send relevant content

MAY:
- Implement preference center for recipients to manage subscriptions
- Set up CloudWatch alarm for complaint rate at 0.05% (early warning)

## Common Issues

- symptoms: "Complaint rate above 0.1%"
  diagnosis: "Recipients marking emails as spam due to unwanted content or no unsubscribe option."
  resolution: "Add unsubscribe links. Remove complained addresses. Review content relevance."

- symptoms: "Account sending paused"
  diagnosis: "Complaint rate exceeded threshold and AWS paused sending."
  resolution: "Address complaint causes. Respond to AWS review. Request sending re-enablement."

- symptoms: "Complaints from specific ISP"
  diagnosis: "ISP feedback loop reporting complaints. Content may trigger spam filters."
  resolution: "Review content for spam triggers. Ensure proper authentication (DKIM, SPF, DMARC)."

## Output Format

```yaml
root_cause: "complaint_handling — <specific_cause>"
evidence:
  - type: complaint_rate
    content: "<current complaint rate and trend>"
  - type: account_status
    content: "<enforcement status and sending enabled>"
severity: CRITICAL
mitigation:
  immediate: "Remove complained addresses and add unsubscribe links"
  long_term: "Implement preference center and automated complaint processing"
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
