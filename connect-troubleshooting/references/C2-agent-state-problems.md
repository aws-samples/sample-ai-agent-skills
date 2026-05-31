---
title: "C2 — Agent State Problems"
description: "Diagnose issues where agents are stuck in wrong states or states don't transition correctly"
status: active
severity: HIGH
triggers:
  - "Agent stuck in state"
  - "Agent not receiving calls"
  - "State not changing"
  - "After call work stuck"
owner: devops-agent
objective: "Resolve agent state issues and restore normal contact routing"
context: "Agent states control contact routing. Agents must be in 'Available' to receive contacts. States can get stuck due to CCP issues, network disconnects, or configuration problems. ACW (After Call Work) auto-timeout can be configured."
---

## Phase 1 — Triage

MUST:
- Check current agent state via real-time metrics or API
- Verify the agent's CCP is connected and responsive
- Check if the agent has multiple CCP sessions open (causes conflicts)
- Check if ACW timeout is configured on the queue

SHOULD:
- Check if the agent's state matches what they see in CCP
- Verify the routing profile's default outbound queue
- Check for recent network disconnections

MAY:
- Check CloudWatch metrics for agent state durations
- Review CCP logs in the browser console

## Phase 2 — Remediate

MUST:
- If stuck in ACW: configure ACW timeout on the queue, or have agent manually change state
- If stuck in Error: have agent refresh CCP and set to Available
- If multiple sessions: close all but one CCP tab/window

SHOULD:
- Configure ACW auto-timeout to prevent indefinite ACW
- Implement agent state monitoring and alerts
- Train agents on proper CCP usage (single session)

MAY:
- Implement supervisor state change capabilities
- Set up dashboards for agent state monitoring

## Common Issues

- symptoms: "Agent stuck in After Call Work indefinitely"
  diagnosis: "ACW timeout not configured on the queue."
  resolution: "Set ACW timeout on the queue (e.g., 60 seconds). Agent will auto-transition to Available."

- symptoms: "Agent shows Available but doesn't receive contacts"
  diagnosis: "CCP session disconnected but state not updated, or routing profile issue."
  resolution: "Refresh CCP. Verify routing profile includes the active queues."

- symptoms: "Agent stuck in Error state"
  diagnosis: "CCP encountered a WebRTC or network error."
  resolution: "Refresh CCP, check network connectivity, close other CCP sessions."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
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

1. If queue configuration was changed, restore original routing profile assignments and queue settings
2. If agent routing or security profiles were modified, restore original profile assignments
## Output Format

```yaml
root_cause: "agent_state — <specific issue>"
evidence:
  - type: agent_metrics
    content: "<agent state and duration>"
severity: HIGH
mitigation:
  immediate: "Reset agent state via CCP refresh or supervisor action"
  long_term: "Configure ACW timeout and implement state monitoring"
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
