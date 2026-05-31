---
title: "E2 — Event Destinations"
description: "Diagnose SES event destination delivery failures"
status: active
severity: MEDIUM
triggers:
  - "event destination not working"
  - "events not delivered"
  - "SNS notification missing"
  - "Firehose delivery failed"
owner: devops-agent
objective: "Resolve SES event destination configuration and delivery issues"
context: "Event destinations publish SES events (send, delivery, bounce, complaint, open, click) to CloudWatch, SNS, Kinesis Data Firehose, or Pinpoint. Each destination type has specific permission requirements."
---

## Phase 1 — Triage

MUST:
- Check event destination configuration: `aws sesv2 get-configuration-set-event-destinations --configuration-set-name <name>`
- Verify matching event types are configured for the destination
- Check destination resource permissions (SNS topic policy, Firehose IAM role)
- Confirm the configuration set is specified in send calls

SHOULD:
- Check CloudWatch for SES event metrics
- Verify SNS topic exists and is in the same region
- Check Kinesis Firehose delivery stream status

MAY:
- Review CloudTrail for event destination API errors
- Check if event destination was recently modified

## Phase 2 — Remediate

MUST:
- Configure correct event types on the destination
- Fix resource permissions for the destination type
- Ensure destination resources exist in the same region

SHOULD:
- Test with a simple SNS destination before complex setups
- Add multiple event types to a single destination for comprehensive tracking

MAY:
- Set up CloudWatch destination for metrics and SNS for real-time notifications
- Implement Kinesis Firehose for high-volume event archival

## Common Issues

- symptoms: "Bounce events not appearing in SNS"
  diagnosis: "Event destination not configured for Bounce event type, or SNS topic policy missing."
  resolution: "Add BOUNCE to matched event types. Add SES service principal to SNS topic policy."

- symptoms: "Open/click tracking not working"
  diagnosis: "Open/click tracking requires HTML email and configuration set with tracking enabled."
  resolution: "Send HTML email. Enable open/click tracking on configuration set."

- symptoms: "Firehose delivery failing"
  diagnosis: "IAM role for Firehose missing SES permissions or Firehose stream inactive."
  resolution: "Verify Firehose IAM role and stream status."

## Output Format

```yaml
root_cause: "event_destination — <specific_cause>"
evidence:
  - type: destination_config
    content: "<event destination settings and event types>"
  - type: permissions
    content: "<destination resource permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix event types, permissions, or destination configuration"
  long_term: "Implement comprehensive event tracking across all email types"
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
