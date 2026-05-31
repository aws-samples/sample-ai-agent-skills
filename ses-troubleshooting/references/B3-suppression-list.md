---
title: "B3 — Suppression List"
description: "Diagnose suppression list issues affecting email delivery"
status: active
severity: MEDIUM
triggers:
  - "email suppressed"
  - "suppression list"
  - "address blocked"
  - "cannot send to address"
owner: devops-agent
objective: "Manage SES account-level suppression list issues"
context: "SES maintains an account-level suppression list that automatically adds addresses generating hard bounces or complaints. Sending to suppressed addresses silently fails. Addresses can be manually removed but may be re-added if the underlying issue persists."
---

## Phase 1 — Triage

MUST:
- Check if address is suppressed: `aws sesv2 get-suppressed-destination --email-address <address>`
- List recent suppressions: `aws sesv2 list-suppressed-destinations --start-date <iso> --end-date <iso>`
- Check suppression list configuration: `aws sesv2 get-suppression-list-attributes`
- Verify the reason for suppression (BOUNCE or COMPLAINT)

SHOULD:
- Check if the address is valid and the mailbox exists
- Review bounce notification details for the suppressed address
- Verify suppression list is enabled for the account

MAY:
- Export full suppression list for analysis
- Check if the address was manually added or auto-added

## Phase 2 — Remediate

MUST:
- Investigate why the address was suppressed before removing
- Remove from suppression list only if the underlying issue is resolved: `aws sesv2 delete-suppressed-destination --email-address <address>`
- Verify the address is valid before re-sending

SHOULD:
- Implement automated suppression list management
- Process bounce/complaint notifications to maintain clean lists
- Set up monitoring for suppression list growth

MAY:
- Bulk import addresses to suppression list for known-bad addresses: `aws sesv2 put-suppressed-destination --email-address <address> --reason BOUNCE`
- Implement suppression list sync with application mailing list

## Common Issues

- symptoms: "Email silently not delivered to specific address"
  diagnosis: "Address is on the account-level suppression list."
  resolution: "Check suppression list. Remove if address is now valid."

- symptoms: "Address keeps getting re-added after removal"
  diagnosis: "Underlying deliverability issue not resolved. Address still bouncing."
  resolution: "Verify address validity. Contact recipient to confirm mailbox exists."

- symptoms: "Many addresses suddenly suppressed"
  diagnosis: "Bulk send to stale list caused mass hard bounces."
  resolution: "Clean mailing list. Implement email validation before sending."

## Output Format

```yaml
root_cause: "suppression_list — <specific_cause>"
evidence:
  - type: suppressed_address
    content: "<suppression reason and date>"
  - type: suppression_config
    content: "<account suppression list settings>"
severity: MEDIUM
mitigation:
  immediate: "Remove valid addresses from suppression list"
  long_term: "Implement automated list hygiene and validation"
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
