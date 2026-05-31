---
title: "D4 — DTMF Issues"
description: "Diagnose issues with DTMF tone detection in IVR menus"
status: active
severity: MEDIUM
triggers:
  - "DTMF not working"
  - "Key presses not detected"
  - "IVR menu not responding"
  - "Wrong option selected"
owner: devops-agent
objective: "Fix DTMF detection in contact flows"
context: "DTMF is only captured by 'Get customer input' blocks, NOT 'Play prompt' blocks. DTMF detection can also be affected by: timeout settings, Lex bot configuration (if using Lex for input), or telephony quality issues."
---

## Phase 1 — Triage

MUST:
- Verify the contact flow uses "Get customer input" blocks (NOT "Play prompt") for menus
- Check the DTMF timeout setting in the block
- Verify the DTMF option mapping matches expected inputs
- Check contact flow logs for input-related entries

SHOULD:
- Test DTMF from different phone types (mobile, landline, VoIP)
- Check if the issue is with specific digits or all DTMF
- Verify the "Get customer input" block has correct branches for each option

MAY:
- Check if Lex is configured on the block (Lex can interfere with DTMF)
- Test with different callers to rule out caller-side issues

## Phase 2 — Remediate

MUST:
- Replace "Play prompt" blocks with "Get customer input" blocks for menus
- Configure correct DTMF timeout (default 5 seconds between digits)
- Map all expected DTMF options to branches

SHOULD:
- Add a "no input" branch for timeout handling
- Add an "invalid input" branch for unrecognized options
- Test all DTMF options after changes

MAY:
- Implement DTMF retry logic with a counter
- Add audio confirmation of the selected option

## Common Issues

- symptoms: "Customer presses keys but nothing happens"
  diagnosis: "Using 'Play prompt' block instead of 'Get customer input'."
  resolution: "Replace with 'Get customer input' block."

- symptoms: "DTMF works for some digits but not others"
  diagnosis: "Missing branch mappings for specific digits."
  resolution: "Add branches for all expected DTMF options."

- symptoms: "DTMF detected but wrong option selected"
  diagnosis: "DTMF option mapping is incorrect."
  resolution: "Verify the digit-to-branch mapping in the 'Get customer input' block."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
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
2. If Lambda or Lex integration was changed, restore the original function/bot association in the contact flow
3. If agent routing or security profiles were modified, restore original profile assignments
## Output Format

```yaml
root_cause: "dtmf — <specific issue>"
evidence:
  - type: contact_flow
    content: "<block configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix the contact flow block configuration"
  long_term: "Implement DTMF testing as part of flow deployment"
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
