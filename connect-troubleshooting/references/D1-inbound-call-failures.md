---
title: "D1 — Inbound Call Failures"
description: "Diagnose failures where inbound calls don't reach the Connect instance"
status: active
severity: CRITICAL
triggers:
  - "Calls not coming in"
  - "Number not working"
  - "Caller hears busy or error"
  - "Inbound calls failing"
owner: devops-agent
objective: "Restore inbound call connectivity"
context: "Inbound calls flow: PSTN → phone number → contact flow → queue → agent. Failures can occur at any stage: number not claimed, number not associated with a flow, flow errors, or telephony issues."
---

## Phase 1 — Triage

MUST:
- Verify the phone number is claimed and active: `aws connect describe-phone-number --phone-number-id <id>`
- Check the phone number is associated with a contact flow
- Verify the contact flow is published and has no errors
- Check CloudWatch metric `MisconfiguredPhoneNumbers`

SHOULD:
- Test calling the number from an external phone
- Check if the issue affects all numbers or specific ones
- Verify the number hasn't been released or ported away
- Check for telephony service issues in the region

MAY:
- Check CloudTrail for phone number configuration changes
- Review CloudWatch `CallsPerInterval` metric for the number

## Phase 2 — Remediate

MUST:
- Associate the phone number with the correct contact flow
- Fix any contact flow errors
- Verify the number is in CLAIMED status

SHOULD:
- Test the full call path after fix
- Check other phone numbers for similar issues
- Verify the contact flow handles all entry points correctly

MAY:
- Set up monitoring for phone number health
- Implement redundant phone numbers for critical lines

## Common Issues

- symptoms: "Caller hears 'number not in service'"
  diagnosis: "Phone number not claimed or was released."
  resolution: "Claim the number or verify it's still in the instance."

- symptoms: "Caller hears ringing but no one answers"
  diagnosis: "Contact flow routes to queue but no agents available."
  resolution: "Check agent availability and queue configuration."

- symptoms: "Caller hears nothing and call drops"
  diagnosis: "Phone number not associated with a contact flow."
  resolution: "Associate the phone number with the correct inbound contact flow."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
5. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "inbound_call_failure — <specific cause>"
evidence:
  - type: phone_number_config
    content: "<number status and flow association>"
severity: CRITICAL
mitigation:
  immediate: "Fix phone number configuration"
  long_term: "Implement phone number monitoring and redundancy"
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
