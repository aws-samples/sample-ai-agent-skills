---
title: "D1 — SMS Delivery Failures"
description: "Diagnose SMS delivery failures and filtering issues"
status: active
severity: HIGH
triggers:
  - "SMS not delivered"
  - "text message failed"
  - "SMS filtered"
  - "carrier filtering"
owner: devops-agent
objective: "Identify and resolve SMS delivery failures"
context: "SMS delivery depends on origination number, 10DLC registration (US), spending limits, carrier filtering, and destination number validity. Silent filtering is common."
---

## Phase 1 — Triage

MUST:
- Check SMS channel: `aws pinpoint get-sms-channel --application-id <app-id>`
- List phone numbers: `aws pinpoint-sms-voice-v2 describe-phone-numbers`
- Check spending limit: `aws pinpoint-sms-voice-v2 describe-spend-limits`
- Check SMS delivery metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Pinpoint --metric-name DirectSendMessagePermanentFailure --dimensions Name=ApplicationId,Value=<app-id> Name=Channel,Value=SMS --start-time <start> --end-time <end> --period 3600 --statistics Sum`

SHOULD:
- Check 10DLC registration status (US traffic): `aws pinpoint-sms-voice-v2 describe-registration-attachments`
- Verify origination number matches destination country
- Check opt-out list for destination number

MAY:
- Review CloudWatch Logs for SMS delivery events
- Check carrier-specific delivery status

## Phase 2 — Remediate

MUST:
- Provision origination number for target country
- Register for 10DLC if sending to US numbers
- Increase spending limit through AWS Support

SHOULD:
- Use appropriate message type (Transactional vs Promotional)
- Implement delivery status tracking

MAY:
- Set up SNS notifications for delivery failures

## Common Issues

- symptoms: "SMS shows as sent but recipient never receives"
  diagnosis: "Carrier filtering due to missing 10DLC registration."
  resolution: "Complete 10DLC registration for US traffic. Use registered campaign."

- symptoms: "SMS sending stops mid-campaign"
  diagnosis: "Monthly spending limit reached."
  resolution: "Request spending limit increase via AWS Support."

## Output Format

```yaml
root_cause: "sms_delivery — <specific_cause>"
evidence:
  - type: sms_channel
    content: "<SMS channel configuration>"
  - type: spending_limit
    content: "<current spending limit and usage>"
severity: HIGH
mitigation:
  immediate: "Fix origination number or spending limit"
  long_term: "Complete 10DLC registration and set appropriate spending limits"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — SMS delivery involves customer phone numbers
- Service Disruption Risk: MEDIUM — SMS failures affect customer communication
- Compliance Impact: HIGH — SMS must comply with TCPA, carrier regulations, and opt-in requirements

## Escalation Conditions
- SMS delivery rate drops significantly
- SMS messages not being delivered to specific carriers
- SMS spending limit reached
- 10DLC or toll-free registration issues
- SMS opt-out processing failures

## Data Sensitivity
- Phone numbers — RESTRICTED (PII)
- SMS content — INTERNAL
- Delivery metrics — INTERNAL
- Carrier information — INTERNAL
- Never expose customer phone numbers in logs

## Prohibited Actions
- Do NOT send SMS to customers without proper opt-in consent
- Do NOT bypass SMS opt-out requests
- Do NOT exceed SMS spending limits without approval
- Do NOT send SMS from unregistered numbers
- Do NOT ignore carrier compliance requirements
- Do NOT share SMS delivery reports containing phone numbers

## Phase 3 — Rollback
- Restore previous SMS configurations
- Re-register phone numbers if needed
- Restore spending limit configurations
- Verify SMS delivery after changes
- Process pending opt-out requests
- Document SMS delivery issues and compliance status

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
