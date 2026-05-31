---
title: "B2 — Hours of Operation Issues"
description: "Diagnose routing failures caused by hours of operation configuration"
status: active
severity: MEDIUM
triggers:
  - "Calls going to after-hours"
  - "Hours of operation wrong"
  - "Timezone mismatch"
  - "Queue closed unexpectedly"
owner: devops-agent
objective: "Fix hours of operation configuration to match business requirements"
context: "Hours of operation define when a queue accepts contacts. The 'Check hours of operation' block in contact flows branches based on whether the current time falls within configured hours. Timezone mismatches are the most common issue."
---

## Phase 1 — Triage

MUST:
- Check hours of operation configuration: `aws connect describe-hours-of-operation --instance-id <id> --hours-of-operation-id <id>`
- Verify the timezone setting matches the business location
- Check if the contact flow uses "Check hours of operation" block
- Verify which hours of operation resource is referenced in the flow

SHOULD:
- Check for holiday or special day overrides
- Verify the hours cover all expected business days
- Check if DST (daylight saving time) transitions caused the issue

MAY:
- Compare the configured timezone with the caller's expected timezone
- Check if multiple hours of operation resources exist and the wrong one is referenced

## Phase 2 — Remediate

MUST:
- Fix the timezone to match the business location
- Update hours to cover all expected business periods
- Verify the correct hours of operation resource is referenced in the flow

SHOULD:
- Add holiday handling in the contact flow (check date before hours)
- Test the hours configuration across timezone boundaries
- Document the hours of operation for each queue

MAY:
- Implement dynamic hours using Lambda and contact attributes
- Set up alerts for after-hours call volume spikes

## Common Issues

- symptoms: "Queue shows closed during business hours"
  diagnosis: "Timezone mismatch — hours are configured in wrong timezone."
  resolution: "Update the hours of operation timezone to match the business location."

- symptoms: "Queue open on holidays when it should be closed"
  diagnosis: "No holiday handling in the contact flow."
  resolution: "Add date-checking logic before the hours of operation check."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Diagnostic investigation: GREEN - No changes to infrastructure"
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
3. If hours of operation were modified, revert to the original schedule immediately
4. If Lambda or Lex integration was changed, restore the original function/bot association in the contact flow
5. If agent routing or security profiles were modified, restore original profile assignments
## Output Format

```yaml
root_cause: "hours_of_operation — <specific issue>"
evidence:
  - type: hours_config
    content: "<hours and timezone details>"
severity: MEDIUM
mitigation:
  immediate: "Fix hours and timezone configuration"
  long_term: "Implement holiday handling and timezone documentation"
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
