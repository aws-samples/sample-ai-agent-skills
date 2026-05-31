---
title: "F1 — Sandbox Limitations"
description: "Diagnose issues caused by SES sandbox mode restrictions"
status: active
severity: MEDIUM
triggers:
  - "sandbox mode"
  - "can only send to verified"
  - "200 email limit"
  - "sandbox restriction"
owner: devops-agent
objective: "Identify sandbox-related limitations and guide production access"
context: "New SES accounts start in sandbox mode. Sandbox limits: send only to verified addresses, 200 emails/day, 1 email/second. Production access must be requested through AWS console or API."
---

## Phase 1 — Triage

MUST:
- Check if account is in sandbox: `aws sesv2 get-account --query 'ProductionAccessEnabled'`
- Verify sending quota: `aws sesv2 get-account --query 'SendQuota'`
- Check if recipient addresses are verified: `aws sesv2 list-email-identities`
- Confirm the error is sandbox-related (MessageRejected for unverified recipient)

SHOULD:
- Check if a production access request is pending
- Verify all test recipients are verified
- Review account age (new accounts are always in sandbox)

MAY:
- Check if the account has been previously moved to production and reverted
- Review AWS Support cases for production access requests

## Phase 2 — Remediate

MUST:
- For testing: verify recipient addresses: `aws sesv2 create-email-identity --email-identity <recipient>`
- For production: request production access through SES console or support case
- Provide use case details, expected volume, and bounce/complaint handling plan

SHOULD:
- Set up bounce and complaint handling before requesting production access
- Verify domain identity (not just email) for production use
- Configure DKIM and custom MAIL FROM before production

MAY:
- Start with low volume and gradually increase after production access
- Implement sending rate management in application code

## Common Issues

- symptoms: "MessageRejected: Email address not verified in sandbox"
  diagnosis: "Account in sandbox mode and recipient not verified."
  resolution: "Verify recipient for testing, or request production access for real sending."

- symptoms: "Daily quota of 200 exceeded"
  diagnosis: "Sandbox daily sending limit reached."
  resolution: "Wait for quota reset or request production access for higher limits."

- symptoms: "Production access request denied"
  diagnosis: "Insufficient use case details or missing bounce/complaint handling."
  resolution: "Resubmit with detailed use case, expected volume, and handling procedures."

## Output Format

```yaml
root_cause: "sandbox — <specific_cause>"
evidence:
  - type: account_status
    content: "<sandbox mode and quota details>"
  - type: verified_identities
    content: "<list of verified identities>"
severity: MEDIUM
mitigation:
  immediate: "Verify test recipients or request production access"
  long_term: "Complete production readiness checklist before requesting access"
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
