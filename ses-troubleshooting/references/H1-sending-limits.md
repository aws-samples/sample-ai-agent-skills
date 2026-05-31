---
title: "H1 — Sending Limits"
description: "Diagnose SES sending quota and limit issues"
status: active
severity: MEDIUM
triggers:
  - "sending quota exceeded"
  - "daily limit reached"
  - "cannot send more emails"
  - "quota increase"
owner: devops-agent
objective: "Resolve SES sending quota issues and manage limits"
context: "SES enforces a daily sending quota (rolling 24-hour period) and per-second rate limit. Sandbox: 200/day, 1/sec. Production quotas vary and can be increased. Quota resets on a rolling basis, not at midnight."
---

## Phase 1 — Triage

MUST:
- Check current quota and usage: `aws sesv2 get-account --query 'SendQuota'`
- Check sending statistics: `aws ses get-send-statistics`
- Verify if account is in sandbox or production: `aws sesv2 get-account --query 'ProductionAccessEnabled'`
- Check CloudWatch sending metrics: `aws cloudwatch get-metric-statistics --namespace AWS/SES --metric-name Send --start-time <iso> --end-time <iso> --period 3600 --statistics Sum`

SHOULD:
- Calculate remaining quota for the current 24-hour period
- Check if sending volume has recently spiked
- Review which applications or campaigns are consuming quota

MAY:
- Check if quota was recently reduced by AWS due to reputation issues
- Review historical sending patterns

## Phase 2 — Remediate

MUST:
- Wait for rolling 24-hour quota to reset if exceeded
- Request quota increase if consistently hitting limits
- Implement sending rate management in application

SHOULD:
- Set up CloudWatch alarms for quota utilization (e.g., 80% threshold)
- Distribute sending across the day to avoid bursts
- Implement queue-based sending for large volumes

MAY:
- Use multiple SES regions to distribute sending load
- Implement priority-based sending (transactional before marketing)

## Common Issues

- symptoms: "Throttling: Maximum sending rate exceeded"
  diagnosis: "Sending faster than per-second rate limit."
  resolution: "Implement rate limiting in application. Check current rate limit in SendQuota."

- symptoms: "Daily message quota exceeded"
  diagnosis: "Sent more emails than 24-hour quota allows."
  resolution: "Wait for quota reset. Request increase if needed."

- symptoms: "Quota seems lower than expected"
  diagnosis: "AWS may have reduced quota due to reputation issues."
  resolution: "Check account status for any reviews. Fix reputation issues first."

## Output Format

```yaml
root_cause: "sending_limits — <specific_cause>"
evidence:
  - type: quota_status
    content: "<current quota, usage, and rate limit>"
  - type: sending_stats
    content: "<recent sending volume>"
severity: MEDIUM
mitigation:
  immediate: "Wait for reset or request quota increase"
  long_term: "Implement rate management and quota monitoring"
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
