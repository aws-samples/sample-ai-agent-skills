---
title: "B1 — SMS Sending Failures"
description: "Diagnose SMS message delivery failures"
status: active
severity: HIGH
triggers:
  - "SMS failed"
  - "SMS not delivered"
  - "text message failed"
  - "SMS error"
owner: devops-agent
objective: "Identify and resolve SMS sending failures"
context: "SMS failures can be caused by sandbox restrictions, opt-out, spending limits, invalid numbers, carrier issues, or origination number problems. SMS delivery status logging provides specific failure reasons."
---

## Phase 1 — Triage

MUST:
- Check SMS attributes: `aws sns get-sms-attributes`
- Check if in sandbox: look for SandboxPhoneNumbers in attributes
- Check opt-out status: `aws sns check-if-phone-number-is-opted-out --phone-number <number>`
- Check spending limit and monthly spend: `aws sns get-sms-attributes --attributes MonthlySpendLimit`
- Check SMS delivery logs if enabled: `aws logs filter-log-events --log-group-name sns/<region>/<account-id>/DirectPublishToPhoneNumber --filter-pattern "FAILURE"`

SHOULD:
- Verify phone number format (E.164: +1234567890)
- Check for SMS-specific CloudWatch metrics
- Verify origination number or sender ID configuration

MAY:
- Check carrier-specific delivery issues
- Review SMS message content for filtering triggers

## Phase 2 — Remediate

MUST:
- If sandbox: request production access or add number to sandbox verified list
- If opted-out: cannot send — respect opt-out. User must opt back in
- If spending limit: request limit increase
- If invalid number: use E.164 format with country code

SHOULD:
- Enable SMS delivery status logging for visibility
- Set appropriate SMS type (Transactional vs Promotional)
- Configure origination numbers for better deliverability

MAY:
- Use Amazon Pinpoint for advanced SMS features
- Implement SMS delivery tracking

## Common Issues

- symptoms: "SMS publish returns success but message not received"
  diagnosis: "Number opted out (silent drop) or sandbox restriction."
  resolution: "Check opt-out status. Verify sandbox mode and verified numbers."

- symptoms: "SMS fails with spending limit error"
  diagnosis: "Monthly SMS spending limit reached ($1.00 default)."
  resolution: "Request spending limit increase through AWS Support."

## Output Format

```yaml
root_cause: "sms_failure — <specific_cause>"
evidence:
  - type: sms_attributes
    content: "<SMS configuration>"
  - type: opt_out_status
    content: "<opt-out check result>"
severity: HIGH
mitigation:
  immediate: "Fix sandbox, opt-out, or spending limit issue"
  long_term: "Enable SMS logging and configure proper origination"
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

- Message payloads may contain sensitive application data
- Subscription endpoints reveal integration architecture
- SMS messages may contain customer PII or authentication codes
- Classification: CONFIDENTIAL - message content and subscriber endpoint data

## Prohibited Actions

- NEVER delete SNS topics with active subscriptions in production
- NEVER modify filter policies on production subscriptions without testing
- NEVER change access policies that could disrupt cross-account message delivery
- NEVER purge dead-letter queues without analyzing failed messages

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
