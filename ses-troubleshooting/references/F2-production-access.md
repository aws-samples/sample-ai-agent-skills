---
title: "F2 — Production Access"
description: "Guide SES production access request and approval process"
status: active
severity: MEDIUM
triggers:
  - "production access"
  - "move out of sandbox"
  - "increase sending limit"
  - "production request denied"
owner: devops-agent
objective: "Successfully obtain SES production access"
context: "Production access removes sandbox restrictions, increases sending quota, and allows sending to any address. Requests are reviewed by AWS and typically take 24 hours. A clear use case and bounce/complaint handling plan are required."
---

## Phase 1 — Triage

MUST:
- Confirm account is in sandbox: `aws sesv2 get-account --query 'ProductionAccessEnabled'`
- Check current sending quota: `aws sesv2 get-account --query 'SendQuota'`
- Verify domain identity is set up with DKIM: `aws sesv2 get-email-identity --email-identity <domain>`
- Confirm bounce and complaint handling is configured

SHOULD:
- Check if a previous request was denied and review feedback
- Verify DMARC record is configured
- Ensure custom MAIL FROM domain is set up

MAY:
- Review AWS documentation for production access requirements
- Check if the account has any existing compliance issues

## Phase 2 — Remediate

MUST:
- Set up domain verification with DKIM before requesting
- Configure bounce and complaint notification handling
- Submit production access request with:
  - Clear use case description
  - Expected sending volume
  - How recipients opt in
  - Bounce and complaint handling procedures

SHOULD:
- Configure DMARC with at least p=none
- Set up custom MAIL FROM domain
- Implement unsubscribe mechanism

MAY:
- Start with a conservative quota request and increase later
- Set up dedicated IP addresses for high-volume sending

## Common Issues

- symptoms: "Production access request pending for days"
  diagnosis: "AWS review typically takes 24 hours but can take longer."
  resolution: "Wait 24-48 hours. Check SES console for status updates."

- symptoms: "Request denied — insufficient information"
  diagnosis: "Use case description too vague or missing bounce handling details."
  resolution: "Resubmit with specific use case, opt-in process, and handling procedures."

- symptoms: "Request denied — new account"
  diagnosis: "Very new AWS accounts may need to establish usage history first."
  resolution: "Use sandbox mode for testing. Build account history. Resubmit after a few days."

## Output Format

```yaml
root_cause: "production_access — <specific_cause>"
evidence:
  - type: account_status
    content: "<sandbox status and current quota>"
  - type: readiness
    content: "<DKIM, DMARC, bounce handling status>"
severity: MEDIUM
mitigation:
  immediate: "Complete readiness checklist and submit request"
  long_term: "Maintain good sending practices to retain production access"
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
