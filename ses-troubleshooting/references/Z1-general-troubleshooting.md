---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for SES issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "SES problem"
  - "email issue"
  - "something wrong with SES"
  - "SES not working"
owner: devops-agent
objective: "Systematically diagnose unspecified SES issues"
context: "This catch-all runbook provides a systematic approach when the specific issue category is unclear. It covers initial data collection, common checks, and routing to specific runbooks."
---

## Phase 1 — Triage

MUST:
- Check account status: `aws sesv2 get-account`
- Verify sending is enabled and check sandbox status
- Check reputation metrics (bounce and complaint rates)
- List verified identities: `aws sesv2 list-email-identities`

SHOULD:
- Check recent sending statistics: `aws ses get-send-statistics`
- Review CloudWatch SES metrics for anomalies
- Check suppression list for affected addresses

MAY:
- Review CloudTrail for SES API errors
- Check AWS Health Dashboard for SES service events

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings
- If sending issue → A1-A3
- If deliverability issue → B1-B3
- If receiving issue → C1-C2
- If template issue → D1-D2
- If configuration issue → E1-E2
- If sandbox issue → F1-F2
- If security issue → G1-G2
- If quota issue → H1-H2

SHOULD:
- Document findings and route to specific runbook
- Collect all relevant evidence before escalating

MAY:
- Check SES event logs for detailed error information
- Contact AWS Support for account-specific issues

## Common Issues

- symptoms: "SES not working (unspecified)"
  diagnosis: "Multiple possible causes. Systematic triage required."
  resolution: "Follow Phase 1 triage to identify category, then route to specific runbook."

- symptoms: "Emails not being received by recipients"
  diagnosis: "Could be sending failure, deliverability issue, or recipient-side filtering."
  resolution: "Check send status first, then deliverability, then recipient configuration."

- symptoms: "SES costs unexpectedly high"
  diagnosis: "High sending volume, dedicated IPs, or deliverability dashboard charges."
  resolution: "Review sending volume and SES pricing components."

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: account_status
    content: "<account details and reputation>"
  - type: triage_findings
    content: "<initial diagnostic results>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook based on triage"
  long_term: "Implement comprehensive SES monitoring and alerting"
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
