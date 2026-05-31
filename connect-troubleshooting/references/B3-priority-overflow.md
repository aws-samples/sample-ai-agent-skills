---
title: "B3 — Queue Priority and Overflow"
description: "Diagnose issues with queue priority, delay settings, and overflow handling"
status: active
severity: MEDIUM
triggers:
  - "Queue overflow"
  - "Priority routing not working"
  - "Contacts waiting too long"
  - "Queue full"
owner: devops-agent
objective: "Fix queue priority and overflow configuration"
context: "Routing profiles assign queues with priority (1=highest) and delay (seconds before routing). When queues overflow, contacts need a fallback path. Without overflow handling, contacts wait indefinitely or disconnect."
---

## Phase 1 — Triage

MUST:
- Check routing profile queue priorities and delays
- Check real-time queue metrics: contacts in queue, oldest contact age, agents available
- Verify the contact flow has overflow handling (Check queue status block)
- Check if queue maximum contacts limit is configured

SHOULD:
- Check if agents are handling contacts from higher-priority queues first
- Verify the delay settings aren't causing unexpected wait times
- Check if the queue has a customer queue flow with wait handling

MAY:
- Analyze historical metrics for queue wait times
- Check if staffing levels match contact volume

## Phase 2 — Remediate

MUST:
- Configure appropriate queue priorities in routing profiles
- Add overflow handling in contact flows (Check queue status → transfer to backup)
- Set up customer queue flows with periodic messages and callback offers

SHOULD:
- Implement callback queues for overflow scenarios
- Set queue maximum contacts limits
- Configure queue-level metrics alarms

MAY:
- Implement dynamic routing based on queue status
- Add estimated wait time announcements

## Common Issues

- symptoms: "Contacts wait in queue while agents handle lower-priority queue contacts"
  diagnosis: "Queue priority misconfigured in routing profile."
  resolution: "Set the correct priority (lower number = higher priority) in the routing profile."

- symptoms: "Contacts disconnect after long wait"
  diagnosis: "No customer queue flow or overflow handling."
  resolution: "Create a customer queue flow with periodic messages and callback option."


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
## Output Format

```yaml
root_cause: "queue_priority_overflow — <specific issue>"
evidence:
  - type: routing_profile
    content: "<priority and delay settings>"
  - type: queue_metrics
    content: "<queue depth and wait times>"
severity: MEDIUM
mitigation:
  immediate: "Fix priority settings and add overflow handling"
  long_term: "Implement dynamic routing and callback queues"
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
