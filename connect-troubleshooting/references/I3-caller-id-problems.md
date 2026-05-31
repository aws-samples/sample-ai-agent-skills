---
title: "I3 — Caller ID Problems"
description: "Diagnose issues with outbound caller ID display"
status: active
severity: MEDIUM
triggers:
  - "Wrong caller ID"
  - "Caller ID not showing"
  - "CNAM issues"
  - "Outbound number display"
owner: devops-agent
objective: "Fix caller ID configuration for outbound calls"
context: "Outbound caller ID must be a number claimed in the Connect instance. The displayed caller ID depends on: queue outbound caller ID config, contact flow 'Set callback number' block, and the receiving carrier's CNAM lookup. You cannot spoof arbitrary caller IDs."
---

## Phase 1 — Triage

MUST:
- Check the outbound caller ID configuration on the queue
- Verify the caller ID number is claimed in the instance
- Check if the contact flow sets a callback number
- Verify the caller ID number is a DID or toll-free (not all types support outbound)

SHOULD:
- Test outbound calls to different carriers to check caller ID display
- Check if CNAM registration is configured for the number
- Verify the caller ID matches what the recipient sees

MAY:
- Check with the receiving carrier about CNAM lookup behavior
- Review outbound whisper flow for caller ID settings

## Phase 2 — Remediate

MUST:
- Set the correct outbound caller ID on the queue
- Ensure the caller ID number is claimed and active
- Use "Set callback number" in the contact flow if dynamic caller ID is needed

SHOULD:
- Register CNAM for the outbound numbers
- Test caller ID display from the recipient's perspective
- Document caller ID configuration per queue

MAY:
- Implement caller ID management process
- Set up monitoring for outbound call success rates

## Common Issues

- symptoms: "Outbound calls show 'Unknown' or wrong number"
  diagnosis: "Queue outbound caller ID not configured or set to wrong number."
  resolution: "Set the correct caller ID number on the outbound queue."

- symptoms: "Caller ID shows number but not company name"
  diagnosis: "CNAM not registered for the number."
  resolution: "Register CNAM with the carrier. Note: CNAM display depends on the receiving carrier."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
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
root_cause: "caller_id — <specific issue>"
evidence:
  - type: queue_config
    content: "<outbound caller ID setting>"
severity: MEDIUM
mitigation:
  immediate: "Fix caller ID configuration"
  long_term: "Register CNAM and document caller ID strategy"
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
