---
title: "B1 — Queue Routing Failures"
description: "Diagnose failures where contacts are not routed to the correct queue or agents"
status: active
severity: HIGH
triggers:
  - "Calls not routing"
  - "Queue empty but agents available"
  - "Contacts stuck in queue"
  - "Wrong queue"
owner: devops-agent
objective: "Identify why contacts are not routing correctly and fix the routing configuration"
context: "Routing involves: contact flow sets the queue → contact enters queue → routing profile determines which agents serve which queues → agent in Available state receives the contact. Failures at any stage break routing."
---

## Phase 1 — Triage

MUST:
- Check the contact flow for "Set working queue" and "Transfer to queue" blocks
- Verify the queue exists and is active: `aws connect describe-queue --instance-id <id> --queue-id <queue-id>`
- Check agent routing profiles: `aws connect describe-routing-profile --instance-id <id> --routing-profile-id <id>`
- Check real-time metrics: agents available, contacts in queue
- Verify agents are in "Available" state

SHOULD:
- Check hours of operation for the queue
- Verify the routing profile includes the target queue
- Check queue priority and delay settings in the routing profile

MAY:
- Check contact flow logs to trace the routing path
- Compare with a working queue/routing configuration

## Phase 2 — Remediate

MUST:
- Ensure the contact flow sets the correct queue before transfer
- Verify agents' routing profiles include the target queue
- Confirm agents are in "Available" state

SHOULD:
- Check queue priority (lower number = higher priority)
- Verify hours of operation match expected business hours
- Test with a known-good agent and queue

MAY:
- Implement queue overflow to a backup queue
- Add "Check queue status" blocks for dynamic routing

## Common Issues

- symptoms: "Contacts go to queue but no agent picks up"
  diagnosis: "Agents' routing profiles don't include the queue, or agents aren't Available."
  resolution: "Add the queue to agents' routing profiles. Verify agent state."

- symptoms: "Contacts route to wrong queue"
  diagnosis: "Contact flow sets the wrong queue or conditional routing is misconfigured."
  resolution: "Check 'Set working queue' block. Verify conditional logic."

- symptoms: "Queue shows contacts but agents show Available"
  diagnosis: "Routing profile queue priority or delay is misconfigured."
  resolution: "Check routing profile queue associations, priority, and delay settings."


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
3. If hours of operation were modified, revert to the original schedule immediately
4. If agent routing or security profiles were modified, restore original profile assignments
5. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "queue_routing — <specific cause>"
evidence:
  - type: routing_config
    content: "<queue and routing profile details>"
severity: HIGH
mitigation:
  immediate: "Fix routing configuration"
  long_term: "Implement routing validation and monitoring"
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
