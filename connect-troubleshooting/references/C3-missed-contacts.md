---
title: "C3 — Missed Contacts"
description: "Diagnose why agents are missing contacts or contacts are being routed away"
status: active
severity: MEDIUM
triggers:
  - "Missed contact"
  - "Agent didn't receive call"
  - "Contact routed to wrong agent"
  - "Agent timeout"
owner: devops-agent
objective: "Reduce missed contacts and improve routing reliability"
context: "Missed contacts occur when an agent doesn't accept within the agent timeout period. The contact is then routed to the next available agent. High missed contact rates indicate CCP issues, agent behavior, or timeout configuration problems."
---

## Phase 1 — Triage

MUST:
- Check missed contact metrics per agent and queue
- Verify agent timeout setting in the queue configuration
- Check if agents are using softphone or desk phone (desk phone adds ring delay)
- Verify CCP is functioning correctly for affected agents

SHOULD:
- Check if missed contacts correlate with specific times or agents
- Verify network quality for agents with high miss rates
- Check if agents have CCP audio notifications enabled

MAY:
- Review CTRs for missed contacts to see the routing path
- Check if auto-accept is configured for the queue

## Phase 2 — Remediate

MUST:
- Adjust agent timeout if too short (default is 20 seconds)
- Verify CCP connectivity for agents with high miss rates
- Ensure agents have audio notifications enabled

SHOULD:
- Consider enabling auto-accept for specific queues
- Implement missed contact alerts for supervisors
- Train agents on CCP best practices

MAY:
- Implement agent performance dashboards
- Set up callback for contacts that experience multiple misses

## Common Issues

- symptoms: "High missed contact rate for desk phone agents"
  diagnosis: "Desk phone ring delay plus agent timeout is too short."
  resolution: "Increase agent timeout or switch to softphone."

- symptoms: "Agent reports not hearing the ring"
  diagnosis: "CCP audio notifications disabled or browser audio blocked."
  resolution: "Enable CCP notifications. Check browser audio permissions."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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
3. If phone number associations were changed, reassociate the number with the original contact flow
4. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "missed_contacts — <specific cause>"
evidence:
  - type: agent_metrics
    content: "<missed contact rate and details>"
severity: MEDIUM
mitigation:
  immediate: "Fix timeout settings and CCP configuration"
  long_term: "Implement monitoring and agent training"
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
