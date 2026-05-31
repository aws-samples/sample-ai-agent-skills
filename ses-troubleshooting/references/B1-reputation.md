---
title: "B1 — Reputation"
description: "Diagnose SES sending reputation issues"
status: active
severity: CRITICAL
triggers:
  - "reputation warning"
  - "account under review"
  - "sending paused"
  - "deliverability declining"
owner: devops-agent
objective: "Monitor and restore SES sending reputation"
context: "SES reputation is based on bounce rate (<5%) and complaint rate (<0.1%). Poor reputation leads to account review, sending pause, or termination. Reputation metrics are visible in the SES console and via CloudWatch."
---

## Phase 1 — Triage

MUST:
- Check account reputation metrics: `aws sesv2 get-account`
- Check bounce rate: `aws cloudwatch get-metric-statistics --namespace AWS/SES --metric-name Reputation.BounceRate --start-time <iso> --end-time <iso> --period 86400 --statistics Average`
- Check complaint rate: `aws cloudwatch get-metric-statistics --namespace AWS/SES --metric-name Reputation.ComplaintRate --start-time <iso> --end-time <iso> --period 86400 --statistics Average`
- Verify sending is still enabled

SHOULD:
- Check deliverability dashboard (requires paid subscription): `aws sesv2 get-deliverability-dashboard-options`
- Review sending volume trends: `aws cloudwatch get-metric-statistics --namespace AWS/SES --metric-name Send --start-time <iso> --end-time <iso> --period 86400 --statistics Sum`
- Identify which campaigns or sending sources are causing issues

MAY:
- Check domain reputation with external tools
- Review IP reputation if using dedicated IPs

## Phase 2 — Remediate

MUST:
- Immediately stop sending to unvalidated or purchased lists
- Clean mailing lists by removing bounced and complained addresses
- Respond to any AWS account review within the specified timeframe

SHOULD:
- Implement warm-up schedule for new sending volumes
- Set up CloudWatch alarms for bounce rate >3% and complaint rate >0.05%
- Enable the deliverability dashboard for ongoing monitoring

MAY:
- Consider dedicated IP addresses for reputation isolation
- Implement IP warm-up plan for dedicated IPs

## Common Issues

- symptoms: "Account under review notification"
  diagnosis: "Bounce or complaint rate exceeded thresholds."
  resolution: "Clean lists, fix bounce/complaint handling, respond to AWS review."

- symptoms: "Emails going to spam folder"
  diagnosis: "Domain reputation degraded due to high bounce/complaint rates."
  resolution: "Fix authentication (DKIM, SPF, DMARC). Clean lists. Reduce sending volume temporarily."

- symptoms: "Sending paused by AWS"
  diagnosis: "Reputation metrics exceeded thresholds and AWS paused sending."
  resolution: "Address root causes. Submit appeal with remediation plan to AWS."

## Output Format

```yaml
root_cause: "reputation — <specific_cause>"
evidence:
  - type: reputation_metrics
    content: "<bounce rate, complaint rate, sending volume>"
  - type: account_status
    content: "<enforcement status and review details>"
severity: CRITICAL
mitigation:
  immediate: "Stop problematic sending, clean lists, respond to review"
  long_term: "Implement reputation monitoring and automated list hygiene"
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
