---
title: "D2 — Phone Number Provisioning Issues"
description: "Diagnose phone number provisioning and management problems"
status: active
severity: HIGH
triggers:
  - "phone number provisioning failed"
  - "cannot request phone number"
  - "10DLC registration"
  - "toll-free verification"
owner: devops-agent
objective: "Identify and resolve phone number provisioning issues"
context: "Phone numbers must be provisioned through Pinpoint SMS and Voice V2. Different number types have different requirements: long codes need 10DLC, toll-free needs verification, short codes need approval."
---

## Phase 1 — Triage

MUST:
- List phone numbers: `aws pinpoint-sms-voice-v2 describe-phone-numbers`
- Check phone number status: `aws pinpoint-sms-voice-v2 describe-phone-numbers --phone-number-ids <phone-number-id>`
- List registrations: `aws pinpoint-sms-voice-v2 describe-registrations`
- Check account attributes: `aws pinpoint-sms-voice-v2 describe-account-attributes`

SHOULD:
- Verify 10DLC brand registration status
- Check toll-free verification status
- Review phone number capabilities (SMS, Voice, MMS)

MAY:
- Check phone number pool configuration
- Review opt-out list for the number

## Phase 2 — Remediate

MUST:
- Complete required registrations (10DLC brand + campaign, toll-free verification)
- Ensure phone number is associated with Pinpoint application
- Verify number capabilities match intended use

SHOULD:
- Use phone number pools for load distribution
- Set up sender ID where supported (non-US)

MAY:
- Request short code for high-volume use cases

## Common Issues

- symptoms: "Phone number request pending for days"
  diagnosis: "10DLC or toll-free verification pending review."
  resolution: "Check registration status. Ensure all required information submitted."

- symptoms: "Phone number cannot send to certain countries"
  diagnosis: "Number type not supported for destination country."
  resolution: "Provision a number in the destination country or use supported number type."

## Output Format

```yaml
root_cause: "phone_number — <specific_cause>"
evidence:
  - type: phone_numbers
    content: "<phone number list and status>"
  - type: registrations
    content: "<registration status>"
severity: HIGH
mitigation:
  immediate: "Complete required registrations"
  long_term: "Plan number provisioning for all target countries"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — phone numbers are PII
- Service Disruption Risk: MEDIUM — phone number issues affect SMS delivery
- Compliance Impact: HIGH — phone number handling must comply with telecom regulations

## Escalation Conditions
- Phone numbers deregistered or released unexpectedly
- Phone number provisioning fails
- Toll-free number verification fails
- 10DLC registration issues
- Phone number porting failures

## Data Sensitivity
- Phone numbers — RESTRICTED (PII)
- Registration details — CONFIDENTIAL
- Carrier information — INTERNAL
- Provisioning configurations — INTERNAL
- Never expose customer phone numbers in logs

## Prohibited Actions
- Do NOT release phone numbers without verifying they are not in active use
- Do NOT bypass phone number registration requirements
- Do NOT use phone numbers for unauthorized purposes
- Do NOT share phone number provisioning details externally
- Do NOT ignore phone number compliance requirements
- Do NOT send messages from unregistered phone numbers

## Phase 3 — Rollback
- Re-provision released phone numbers if possible
- Restore phone number configurations
- Re-register phone numbers if registration was lost
- Verify SMS delivery after phone number changes
- Document phone number issues and registration status
- Update phone number management procedures

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
