---
title: "A1 — Send Failures"
description: "Diagnose why emails fail to send through SES"
status: active
severity: HIGH
triggers:
  - "email not sending"
  - "MessageRejected"
  - "send email failed"
  - "SES error"
owner: devops-agent
objective: "Identify and resolve SES email sending failures"
context: "Send failures are caused by unverified identities, sandbox restrictions, quota exceeded, IAM permission issues, malformed requests, or account suspension. The error message in the API response provides the specific reason."
---

## Phase 1 — Triage

MUST:
- Check account status and sandbox mode: `aws sesv2 get-account --query '{ProductionAccess:ProductionAccessEnabled,SendingEnabled:SendingEnabled}'`
- Verify sender identity: `aws sesv2 get-email-identity --email-identity <from-address-or-domain>`
- Check sending quota: `aws sesv2 get-account --query 'SendQuota'`
- Review the specific error message from the failed send attempt

SHOULD:
- Check if recipient is on suppression list: `aws sesv2 get-suppressed-destination --email-address <recipient>`
- Verify IAM permissions for the calling identity include ses:SendEmail
- Check CloudWatch sending metrics: `aws cloudwatch get-metric-statistics --namespace AWS/SES --metric-name Send --start-time <iso> --end-time <iso> --period 3600 --statistics Sum`

MAY:
- Check CloudTrail for SendEmail/SendRawEmail API errors
- Verify email content does not trigger content filtering

## Phase 2 — Remediate

MUST:
- Verify the From address identity (email or domain)
- If in sandbox, verify recipient addresses or request production access
- Ensure sending is not paused on the account

SHOULD:
- Remove recipient from suppression list if appropriate: `aws sesv2 delete-suppressed-destination --email-address <recipient>`
- Fix IAM permissions to include ses:SendEmail and ses:SendRawEmail

MAY:
- Implement retry logic with exponential backoff for transient failures
- Set up SNS notifications for bounce and complaint events

## Common Issues

- symptoms: "MessageRejected: Email address is not verified"
  diagnosis: "From address not verified, or in sandbox and recipient not verified."
  resolution: "Verify the identity: `aws sesv2 create-email-identity --email-identity <address>`"

- symptoms: "Throttling: Daily message quota exceeded"
  diagnosis: "Account has exceeded the 24-hour sending quota."
  resolution: "Wait for quota to reset (rolling 24h) or request quota increase."

- symptoms: "AccessDenied on SendEmail"
  diagnosis: "IAM policy does not grant ses:SendEmail permission."
  resolution: "Add ses:SendEmail and ses:SendRawEmail to the IAM policy."

## Output Format

```yaml
root_cause: "send_failure — <specific_cause>"
evidence:
  - type: account_status
    content: "<account status and sending enabled>"
  - type: identity_verification
    content: "<identity verification state>"
severity: HIGH
mitigation:
  immediate: "Fix identity verification, permissions, or quota"
  long_term: "Implement proper error handling and monitoring"
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
