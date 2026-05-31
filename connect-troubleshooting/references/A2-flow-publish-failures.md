---
title: "A2 — Contact Flow Publish Failures"
description: "Diagnose failures when saving or publishing contact flows"
status: active
severity: HIGH
triggers:
  - "Cannot publish flow"
  - "Flow save error"
  - "Invalid contact flow"
owner: devops-agent
objective: "Identify why the contact flow cannot be published and fix it"
context: "Contact flows fail to publish due to: size limit exceeded (32 KB), invalid block configuration, missing required fields, orphaned blocks, or circular references. The error message is often generic."
---

## Phase 1 — Triage

MUST:
- Check the error message in the Connect console when publishing
- Verify the flow definition size is under 32 KB
- Check for orphaned blocks (blocks not connected to the flow)
- Verify all blocks have required fields configured (prompts, queues, Lambda ARNs)

SHOULD:
- Check for circular references (flow loops without exit conditions)
- Verify referenced resources exist (queues, Lambda functions, Lex bots, prompts)
- Check if the flow was recently imported or modified via API

MAY:
- Export the flow JSON and validate it manually
- Compare with a known-good version of the flow

## Phase 2 — Remediate

MUST:
- If size limit: split the flow into sub-flows using "Transfer to flow" blocks
- If orphaned blocks: delete or connect them
- If missing resources: create the resources or update the references

SHOULD:
- Validate the flow in the visual editor before publishing
- Test the flow after publishing with a test call

MAY:
- Implement version control for contact flow definitions
- Use the Connect API to validate flows programmatically

## Common Issues

- symptoms: "Generic error when publishing, no specific message"
  diagnosis: "Flow definition exceeds 32 KB size limit."
  resolution: "Split into sub-flows. Remove unused blocks. Simplify branching."

- symptoms: "Error referencing a specific block"
  diagnosis: "Block has invalid configuration (missing prompt, invalid ARN)."
  resolution: "Fix the block configuration. Verify all referenced resources exist."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
3. If Lambda or Lex integration was changed, restore the original function/bot association in the contact flow
4. If agent routing or security profiles were modified, restore original profile assignments
5. If phone number associations were changed, reassociate the number with the original contact flow
## Output Format

```yaml
root_cause: "flow_publish_failure — <specific cause>"
evidence:
  - type: flow_definition
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix the flow and publish"
  long_term: "Implement flow size monitoring and version control"
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
