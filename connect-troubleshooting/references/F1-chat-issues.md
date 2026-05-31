---
title: "F1 — Chat Channel Issues"
description: "Diagnose issues with the Amazon Connect chat channel"
status: active
severity: HIGH
triggers:
  - "Chat not working"
  - "Chat messages not delivered"
  - "Chat disconnects"
  - "Chat widget error"
owner: devops-agent
objective: "Restore chat channel functionality"
context: "Connect chat uses a different path than voice: chat widget/API → Connect chat endpoint → contact flow → queue → agent CCP. Chat has its own contact flows (chat-specific), participant connections, and message delivery mechanisms."
---

## Phase 1 — Triage

MUST:
- Verify chat is enabled on the instance
- Check the chat contact flow is published and configured
- Verify the chat widget or API integration is correct
- Check contact flow logs for chat-specific errors

SHOULD:
- Verify the routing profile includes chat as a channel
- Check agent CCP for chat handling capability
- Verify the chat participant connection is established
- Check if the issue is with the widget, API, or Connect-side

MAY:
- Check CloudWatch metrics for chat contacts
- Review the chat widget JavaScript console for errors
- Test the chat API directly

## Phase 2 — Remediate

MUST:
- Fix the chat contact flow if it has errors
- Ensure routing profiles include the chat channel
- Verify the chat widget configuration matches the instance

SHOULD:
- Test chat end-to-end after fix
- Verify agents can handle chat contacts in CCP
- Check chat-specific queue configuration

MAY:
- Implement chat monitoring and alerting
- Set up chat transcript logging

## Common Issues

- symptoms: "Chat widget loads but messages don't send"
  diagnosis: "Chat API endpoint or instance configuration incorrect."
  resolution: "Verify the chat widget's instance ID and contact flow ID."

- symptoms: "Chat connects but no agent picks up"
  diagnosis: "Routing profile doesn't include chat channel or no agents available for chat."
  resolution: "Add chat channel to routing profiles. Verify agent availability for chat."

- symptoms: "Chat disconnects after a few minutes"
  diagnosis: "Chat idle timeout or participant connection lost."
  resolution: "Check chat timeout settings. Verify network connectivity."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Network configuration changes: YELLOW - May affect connectivity"
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
## Output Format

```yaml
root_cause: "chat — <specific issue>"
evidence:
  - type: contact_flow_logs
    content: "<chat flow errors>"
severity: HIGH
mitigation:
  immediate: "Fix chat configuration"
  long_term: "Implement chat monitoring and testing"
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
