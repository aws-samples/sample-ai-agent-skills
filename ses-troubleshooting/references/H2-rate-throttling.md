---
title: "H2 — Rate Throttling"
description: "Diagnose SES per-second rate throttling issues"
status: active
severity: MEDIUM
triggers:
  - "rate throttling"
  - "too many requests"
  - "sending rate exceeded"
  - "throttled SES"
owner: devops-agent
objective: "Resolve SES per-second sending rate throttling"
context: "SES enforces a per-second sending rate separate from the daily quota. Exceeding the rate returns a Throttling error. The rate limit is shown in SendQuota.MaxSendRate. Sandbox rate is 1/sec. Production rates vary."
---

## Phase 1 — Triage

MUST:
- Check current rate limit: `aws sesv2 get-account --query 'SendQuota.MaxSendRate'`
- Check if application is sending faster than the rate limit
- Review error responses for Throttling exceptions
- Check CloudWatch for rejected sends: `aws cloudwatch get-metric-statistics --namespace AWS/SES --metric-name Reject --start-time <iso> --end-time <iso> --period 300 --statistics Sum`

SHOULD:
- Analyze sending pattern for bursts
- Check if multiple applications share the same SES account
- Review application retry logic for exponential backoff

MAY:
- Check if rate limit was recently changed
- Monitor sending rate over time with CloudWatch

## Phase 2 — Remediate

MUST:
- Implement rate limiting in application to stay within MaxSendRate
- Add exponential backoff retry logic for Throttling errors
- Distribute sends evenly rather than in bursts

SHOULD:
- Use SQS queue to buffer and rate-limit outgoing emails
- Request rate limit increase if consistently hitting the limit
- Implement per-application rate tracking

MAY:
- Use multiple SES regions to increase aggregate rate
- Implement priority queuing (transactional emails first)

## Common Issues

- symptoms: "Throttling exception on SendEmail"
  diagnosis: "Application sending faster than MaxSendRate."
  resolution: "Implement rate limiting. Check MaxSendRate and throttle accordingly."

- symptoms: "Intermittent throttling during peak hours"
  diagnosis: "Multiple applications or campaigns sending simultaneously."
  resolution: "Coordinate sending across applications. Use queue-based sending."

- symptoms: "Rate limit seems too low"
  diagnosis: "Account may be in sandbox (1/sec) or quota was reduced."
  resolution: "Check sandbox status. Request rate increase if in production."

## Output Format

```yaml
root_cause: "rate_throttling — <specific_cause>"
evidence:
  - type: rate_limit
    content: "<MaxSendRate and current sending pattern>"
  - type: throttle_errors
    content: "<throttling error frequency>"
severity: MEDIUM
mitigation:
  immediate: "Implement rate limiting and retry logic"
  long_term: "Use queue-based sending and request rate increase"
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
