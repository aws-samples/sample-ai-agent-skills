---
title: "G1 — Partner Integration Issues"
description: "Diagnose partner event source integration problems"
status: active
severity: MEDIUM
triggers:
  - "partner event source"
  - "partner integration"
  - "partner events not arriving"
  - "SaaS integration"
  - "partner event bus"
owner: devops-agent
objective: "Identify and resolve partner event source integration issues"
context: "Partner event sources (SaaS integrations) deliver events to a dedicated partner event bus. The partner event source must be associated with an event bus. Partner sources have a lifecycle: pending → active → deleted. Issues include association, activation, and event delivery."
---

## Phase 1 — Triage

MUST:
- List partner event sources: `aws events list-partner-event-source-accounts --event-source-name <partner-source>`
- List event buses for partner sources: `aws events list-event-buses --name-prefix aws.partner`
- Check if the partner event source is associated with an event bus
- Verify the partner event source state (pending, active, deleted)

SHOULD:
- Check if events are flowing to the partner event bus
- Verify rules exist on the partner event bus to process events
- Check partner-side configuration (API keys, webhooks, etc.)
- Review CloudTrail for partner event source changes

MAY:
- Check partner documentation for setup requirements
- Verify partner account/subscription is active
- Test with partner's event testing tools

## Phase 2 — Remediate

MUST:
- Associate the partner event source with an event bus if not done
- Create rules on the partner event bus to process events
- Verify partner-side configuration is correct

SHOULD:
- Configure DLQ on rules processing partner events
- Set up monitoring for partner event delivery
- Document partner integration configuration

MAY:
- Implement partner event validation
- Set up alerts for partner event delivery failures

## Common Issues

- symptoms: "Partner event source in pending state"
  diagnosis: "The partner event source has not been associated with an event bus."
  resolution: "Create a partner event bus: aws events create-partner-event-bus --event-source-name <partner-source>."

- symptoms: "Partner events not arriving despite active source"
  diagnosis: "Partner-side configuration is incorrect or partner is not sending events."
  resolution: "Verify partner-side setup (API keys, webhook URLs, event types). Contact partner support."

- symptoms: "Partner event bus exists but no rules processing events"
  diagnosis: "No rules have been created on the partner event bus."
  resolution: "Create rules on the partner event bus with appropriate event patterns and targets."

## Output Format

```yaml
root_cause: "partner_integration — <specific_cause>"
evidence:
  - type: partner_source
    content: "<partner event source status>"
  - type: event_bus
    content: "<partner event bus configuration>"
  - type: rules
    content: "<rules on the partner event bus>"
severity: MEDIUM
mitigation:
  immediate: "Fix partner source association or configuration"
  long_term: "Implement partner event monitoring and documentation"
```


## Safety Ratings
```
safety_ratings:
  - "List partner event sources: GREEN — read-only API call"
  - "List partner event buses: GREEN — read-only API call"
  - "Check partner source state: GREEN — read-only inspection"
  - "Create partner event bus: YELLOW — creates new event bus resource"
  - "Create rules on partner bus: YELLOW — begins processing partner events"
  - "Delete partner event bus: RED — permanently removes partner integration"
```

## Escalation Conditions
- Partner integration serves production SaaS event processing
- Partner event source in deleted state requiring re-creation from partner side
- Partner-side configuration changes requiring vendor coordination
- Partner events not arriving despite correct AWS-side configuration

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Partner event source names: SaaS integration identifiers"
    - "Partner event payloads: third-party business data"
    - "Partner API keys and webhook URLs: authentication credentials"
  handling: "Partner integration details may reveal vendor relationships. Do not expose partner credentials."
```

## Prohibited Actions
- NEVER delete a partner event bus that is actively receiving production events
- NEVER share partner API keys or webhook URLs in logs or outputs
- NEVER modify partner-side configuration without coordinating with the partner
- NEVER create rules on a partner bus without understanding the event volume and format

## Phase 3 — Rollback
- If partner event bus was created: delete the bus if not needed (partner source returns to pending)
- If rules were added to partner bus: delete the rules to stop processing partner events
- If partner event bus was deleted: re-associate the partner event source (requires partner-side action if source was deleted)
- If DLQ was configured on partner bus rules: remove DLQ configuration if not needed

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
