---
title: "D2 — Outbound Call Failures"
description: "Diagnose failures when making outbound calls from Connect"
status: active
severity: HIGH
triggers:
  - "Outbound call failed"
  - "Cannot make outbound calls"
  - "Caller ID rejected"
  - "International calling blocked"
owner: devops-agent
objective: "Restore outbound calling capability"
context: "Outbound calls require: a claimed caller ID number, outbound calling enabled on the instance, the destination country allowed, and the agent's routing profile configured with an outbound queue. International calling requires explicit country-level enablement."
---

## Phase 1 — Triage

MUST:
- Verify outbound calling is enabled on the instance
- Check the agent's routing profile has a default outbound queue
- Verify the caller ID number is claimed in the instance
- Check if the destination country is allowed for outbound calling

SHOULD:
- Check the outbound whisper flow for errors
- Verify the agent's security profile allows outbound calls
- Check if the destination number format is correct (E.164)

MAY:
- Check CloudWatch metrics for outbound call attempts
- Review CloudTrail for outbound calling configuration changes

## Phase 2 — Remediate

MUST:
- Enable outbound calling on the instance if disabled
- Set a default outbound queue in the agent's routing profile
- Claim a caller ID number if none exists
- Enable the destination country for international calls

SHOULD:
- Verify the outbound whisper flow works correctly
- Test outbound calls to different destination types
- Ensure caller ID number matches the expected region

MAY:
- Implement outbound calling policies and restrictions
- Set up outbound call monitoring

## Common Issues

- symptoms: "Agent gets error when trying to make outbound call"
  diagnosis: "No default outbound queue in routing profile or outbound calling disabled."
  resolution: "Set default outbound queue in routing profile. Enable outbound calling on instance."

- symptoms: "International calls fail"
  diagnosis: "Destination country not enabled for outbound calling."
  resolution: "Enable the country in the Connect instance telephony settings."

- symptoms: "Outbound call connects but caller ID shows wrong number"
  diagnosis: "Outbound caller ID not configured correctly in queue or flow."
  resolution: "Set the correct caller ID number on the outbound queue."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
```

## Escalation Conditions

- Contact center is live with active calls or chats in progress
- Issue affects agent ability to receive or handle contacts
- Fix requires modifying a contact flow associated with a phone number
- Resolution involves changes to hours of operation during business hours
- Issue impacts call recording, CTR generation, or compliance requirements

## Data Sensitivity

HIGH - Contact Trace Records (CTRs) contain customer phone numbers, call timestamps, and interaction details. Call recordings contain customer voice data. Chat transcripts may contain PII, financial data, or healthcare information. Agent data includes employee performance metrics.

## Prohibited Actions

- NEVER suggest deleting a contact flow that is associated with a phone number - this breaks inbound routing
- NEVER suggest modifying production hours of operation during active business hours
- NEVER recommend removing security profiles from agents while they are logged in
- NEVER suggest disabling call recording on queues that have compliance requirements
- NEVER recommend deleting or modifying a published contact flow without first testing in a non-production flow

## Phase 3 - Rollback

1. If a contact flow was modified, publish the previous saved version of the flow from the Connect console
2. If queue configuration was changed, restore original routing profile assignments and queue settings
3. If agent routing or security profiles were modified, restore original profile assignments
4. If phone number associations were changed, reassociate the number with the original contact flow
## Output Format

```yaml
root_cause: "outbound_call_failure — <specific cause>"
evidence:
  - type: instance_config
    content: "<outbound calling configuration>"
severity: HIGH
mitigation:
  immediate: "Fix outbound calling configuration"
  long_term: "Document outbound calling requirements and implement monitoring"
```

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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
