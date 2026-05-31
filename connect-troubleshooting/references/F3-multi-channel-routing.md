---
title: "F3 — Multi-Channel Routing Issues"
description: "Diagnose issues with routing across voice, chat, and task channels"
status: active
severity: MEDIUM
triggers:
  - "Channel concurrency issues"
  - "Agent handling wrong channel"
  - "Cross-channel routing"
owner: devops-agent
objective: "Fix multi-channel routing configuration"
context: "Routing profiles define which channels an agent handles and the concurrency per channel (e.g., 1 voice + 3 chats). Misconfigured concurrency or channel settings cause agents to receive unexpected contact types or miss contacts."
---

## Phase 1 — Triage

MUST:
- Check routing profile channel configuration and concurrency limits
- Verify which channels are enabled on the instance
- Check if agents are receiving contacts from unexpected channels
- Verify queue channel configuration

SHOULD:
- Check if cross-channel concurrency is configured correctly
- Verify agents understand multi-channel handling in CCP
- Check if channel-specific contact flows are configured

MAY:
- Review historical metrics per channel
- Check if channel concurrency limits are causing contact delays

## Phase 2 — Remediate

MUST:
- Set appropriate concurrency per channel in routing profiles
- Ensure channel-specific contact flows are assigned
- Verify queue supports the required channels

SHOULD:
- Train agents on multi-channel handling
- Test each channel independently and together
- Monitor per-channel metrics

MAY:
- Implement channel-specific routing strategies
- Set up channel utilization dashboards

## Common Issues

- symptoms: "Agent receives chat while on a voice call"
  diagnosis: "Routing profile allows concurrent voice and chat."
  resolution: "Adjust concurrency settings. Set voice to 1 and chat to 0 during voice, or accept the multi-channel design."

- symptoms: "Chat contacts not routing even though agents are available"
  diagnosis: "Routing profile has chat concurrency set to 0 or chat channel not included."
  resolution: "Add chat channel to routing profile with appropriate concurrency."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
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
root_cause: "multi_channel — <specific issue>"
evidence:
  - type: routing_profile
    content: "<channel and concurrency configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix routing profile channel configuration"
  long_term: "Document multi-channel strategy and implement monitoring"
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
