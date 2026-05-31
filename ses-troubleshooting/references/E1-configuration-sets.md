---
title: "E1 — Configuration Sets"
description: "Diagnose SES configuration set issues"
status: active
severity: MEDIUM
triggers:
  - "configuration set error"
  - "config set not found"
  - "event tracking not working"
  - "configuration set limit"
owner: devops-agent
objective: "Resolve SES configuration set configuration issues"
context: "Configuration sets enable event tracking, IP pool assignment, and sending identity override. They are optional but recommended for production. Each account can have up to 10,000 configuration sets."
---

## Phase 1 — Triage

MUST:
- List configuration sets: `aws sesv2 list-configuration-sets`
- Check specific configuration set: `aws sesv2 get-configuration-set --configuration-set-name <name>`
- Verify event destinations: `aws sesv2 get-configuration-set-event-destinations --configuration-set-name <name>`
- Check if configuration set name is specified correctly in send calls

SHOULD:
- Verify event destination permissions (SNS topic policy, Kinesis Firehose role)
- Check CloudWatch for configuration set metrics
- Review sending identity override settings

MAY:
- Check IP pool assignment if using dedicated IPs
- Verify reputation metrics per configuration set

## Phase 2 — Remediate

MUST:
- Create configuration set if missing: `aws sesv2 create-configuration-set --configuration-set-name <name>`
- Fix event destination configuration
- Ensure configuration set name matches exactly in send calls (case-sensitive)

SHOULD:
- Add event destinations for bounce, complaint, and delivery events
- Configure reputation tracking on the configuration set

MAY:
- Set up separate configuration sets for different email types (transactional, marketing)
- Configure sending pool for dedicated IP management

## Common Issues

- symptoms: "ConfigurationSetDoesNotExist error"
  diagnosis: "Configuration set name mismatch or set in different region."
  resolution: "Verify exact name (case-sensitive) and region."

- symptoms: "Events not appearing in destination"
  diagnosis: "Event destination misconfigured or missing permissions."
  resolution: "Check event destination configuration and IAM/resource permissions."

- symptoms: "Cannot create more configuration sets"
  diagnosis: "Account limit of 10,000 configuration sets reached."
  resolution: "Delete unused configuration sets or request limit increase."

## Output Format

```yaml
root_cause: "configuration_set — <specific_cause>"
evidence:
  - type: config_set
    content: "<configuration set details>"
  - type: event_destinations
    content: "<event destination configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix configuration set name, permissions, or event destinations"
  long_term: "Standardize configuration set usage across applications"
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
