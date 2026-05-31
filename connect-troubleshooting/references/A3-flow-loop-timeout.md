---
title: "A3 — Contact Flow Loop and Timeout"
description: "Diagnose contact flows that loop indefinitely or cause contacts to time out"
status: active
severity: MEDIUM
triggers:
  - "Contact stuck in flow"
  - "Infinite loop"
  - "Flow timeout"
  - "Customer hears repeated prompts"
owner: devops-agent
objective: "Identify and fix flow loops or timeout conditions"
context: "Contact flows can loop when Loop blocks have no exit condition, when error branches loop back to the same block, or when 'Get customer input' blocks loop on no-input/no-match without a maximum retry limit."
---

## Phase 1 — Triage

MUST:
- Check contact flow logs for repeated block executions
- Identify the looping section in the flow definition
- Check Loop blocks for iteration limits
- Check "Get customer input" blocks for timeout and no-match handling

SHOULD:
- Check if the loop is caused by error branches looping back
- Verify Lambda integrations aren't causing retry loops
- Check the contact duration for affected contacts

MAY:
- Trace a specific contact through the flow logs
- Test the flow with specific inputs that trigger the loop

## Phase 2 — Remediate

MUST:
- Add iteration limits to Loop blocks
- Add maximum retry logic to "Get customer input" blocks
- Ensure error branches don't loop back to the same block indefinitely

SHOULD:
- Add a fallback path after max retries (transfer to agent or disconnect with message)
- Add logging to identify where contacts spend the most time
- Test all loop exit conditions

MAY:
- Implement a contact attribute counter for custom loop limits
- Add CloudWatch alarms for abnormally long contact durations

## Common Issues

- symptoms: "Customer hears the same menu repeatedly"
  diagnosis: "'Get customer input' loops on no-match without a retry limit."
  resolution: "Add a counter attribute and check it to limit retries. Transfer to agent after max retries."

- symptoms: "Contact stays connected for hours with no agent"
  diagnosis: "Flow has an infinite loop between blocks."
  resolution: "Add exit conditions to all loops. Ensure error branches lead to resolution."


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
2. If Lambda or Lex integration was changed, restore the original function/bot association in the contact flow
3. If agent routing or security profiles were modified, restore original profile assignments
## Output Format

```yaml
root_cause: "flow_loop — <looping blocks>"
evidence:
  - type: contact_flow_logs
    content: "<repeated block executions>"
severity: MEDIUM
mitigation:
  immediate: "Add loop limits and fallback paths"
  long_term: "Implement flow design standards with mandatory exit conditions"
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
