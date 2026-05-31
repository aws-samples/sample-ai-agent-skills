---
title: "B4 — Skills-Based Routing Issues"
description: "Diagnose issues with skills-based or attribute-based routing"
status: active
severity: MEDIUM
triggers:
  - "Wrong agent getting calls"
  - "Skills routing not working"
  - "Agent proficiency mismatch"
owner: devops-agent
objective: "Fix skills-based routing to match contacts with appropriate agents"
context: "Connect supports routing based on agent proficiency and routing profile configuration. Contacts can be routed to specific queues based on attributes, and agents serve queues based on their routing profile assignments."
---

## Phase 1 — Triage

MUST:
- Check the contact flow routing logic (attribute-based queue selection)
- Verify agent routing profiles match expected skill assignments
- Check queue-to-routing-profile mappings
- Verify contact attributes used for routing decisions are set correctly

SHOULD:
- Check if routing profiles have the correct queue priorities
- Verify agent proficiency levels if using proficiency-based routing
- Check if the routing logic handles all possible attribute values

MAY:
- Trace specific contacts through the flow to see routing decisions
- Compare routing outcomes with expected behavior

## Phase 2 — Remediate

MUST:
- Fix routing profile queue assignments to match skill requirements
- Correct contact flow routing logic
- Verify attribute values used for routing decisions

SHOULD:
- Document the routing strategy and skill matrix
- Test routing with different attribute combinations
- Implement fallback routing for unmatched skills

MAY:
- Implement more granular routing using Lambda-based decisions
- Set up routing analytics to monitor skill matching effectiveness

## Common Issues

- symptoms: "Spanish-speaking callers routed to English-only agents"
  diagnosis: "Language attribute not set correctly or routing logic doesn't branch on language."
  resolution: "Fix the contact flow to set language attribute and route to the correct queue."

- symptoms: "All contacts go to the same queue regardless of selection"
  diagnosis: "Set working queue block uses a static queue instead of dynamic routing."
  resolution: "Use contact attributes and conditional logic to set the queue dynamically."


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
3. If Lambda or Lex integration was changed, restore the original function/bot association in the contact flow
4. If agent routing or security profiles were modified, restore original profile assignments
5. If phone number associations were changed, reassociate the number with the original contact flow
## Output Format

```yaml
root_cause: "skills_routing — <specific issue>"
evidence:
  - type: routing_config
    content: "<routing profile and flow logic>"
severity: MEDIUM
mitigation:
  immediate: "Fix routing logic and profile assignments"
  long_term: "Document routing strategy and implement monitoring"
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
