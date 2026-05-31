---
title: "C2 — Schema Discovery Issues"
description: "Diagnose schema discovery configuration and operation problems"
status: active
severity: LOW
triggers:
  - "schema discovery"
  - "discovery not working"
  - "schemas not discovered"
  - "auto-discover"
owner: devops-agent
objective: "Identify and resolve schema discovery issues on EventBridge event buses"
context: "Schema discovery automatically creates schemas from events flowing through an event bus. It must be explicitly enabled per bus. Discovery has cost implications ($0.10 per million events). Discovered schemas appear in the discovered-schemas registry."
---

## Phase 1 — Triage

MUST:
- Check if discovery is enabled on the event bus: `aws schemas list-discoverers`
- Describe the discoverer: `aws schemas describe-discoverer --discoverer-id <discoverer-id>`
- Verify events are flowing through the event bus
- Check the discovered-schemas registry: `aws schemas list-schemas --registry-name discovered-schemas`

SHOULD:
- Verify the discoverer State is STARTED (not STOPPED)
- Check if the event bus has recent events
- Review discovered schema content for accuracy
- Check CloudTrail for discoverer configuration changes

MAY:
- Monitor discovery costs
- Check for cross-account discovery limitations
- Review discoverer tags

## Phase 2 — Remediate

MUST:
- Enable discovery on the event bus if not enabled
- Start the discoverer if stopped: `aws schemas start-discoverer --discoverer-id <discoverer-id>`
- Ensure events are flowing through the bus

SHOULD:
- Disable discovery when not actively developing (cost savings)
- Review discovered schemas for accuracy
- Use discovered schemas for code generation

MAY:
- Set up cost monitoring for schema discovery
- Implement discovery lifecycle management

## Common Issues

- symptoms: "No schemas being discovered"
  diagnosis: "Discovery is not enabled or the discoverer is stopped."
  resolution: "Create a discoverer for the event bus and start it."

- symptoms: "Discovery enabled but specific event types not discovered"
  diagnosis: "Events must flow through the bus after discovery is enabled. Historical events are not discovered."
  resolution: "Send new events through the bus. Discovery only captures events after enablement."

- symptoms: "High costs from schema discovery"
  diagnosis: "Discovery charges per event analyzed on high-volume buses."
  resolution: "Disable discovery when not actively developing. Enable only during development phases."

## Output Format

```yaml
root_cause: "schema_discovery — <specific_cause>"
evidence:
  - type: discoverer
    content: "<discoverer status and configuration>"
  - type: schemas
    content: "<discovered schemas>"
  - type: event_bus
    content: "<event bus activity>"
severity: LOW
mitigation:
  immediate: "Enable and start discovery"
  long_term: "Manage discovery lifecycle and costs"
```


## Safety Ratings
```
safety_ratings:
  - "List discoverers: GREEN — read-only API call"
  - "Describe discoverer: GREEN — read-only API call"
  - "List discovered schemas: GREEN — read-only API call"
  - "Start discoverer: YELLOW — begins analyzing events, incurs cost ($0.10/million events)"
  - "Stop discoverer: GREEN — stops analysis, no data loss"
  - "Create discoverer: YELLOW — enables event analysis with cost implications"
```

## Escalation Conditions
- Schema discovery costs exceeding budget on high-volume event buses
- Discovered schemas not matching expected event structures
- Discovery needed for production event bus analysis
- Cross-account discovery limitations blocking development

## Data Sensitivity
```
data_sensitivity:
  classification: LOW
  sensitive_fields:
    - "Discovered schemas: event structure revealing data model"
    - "Event bus activity: event volume and patterns"
  handling: "Discovered schemas may reveal data model details. Manage access appropriately."
```

## Prohibited Actions
- NEVER leave discovery enabled on high-volume production buses without cost monitoring
- NEVER delete discovered schemas that are actively used for code generation
- NEVER enable discovery on event buses carrying sensitive data without reviewing schema exposure

## Phase 3 — Rollback
- If discoverer was started: stop the discoverer to halt cost accrual
- If discoverer was created: delete the discoverer if not needed
- If discovery was stopped: restart the discoverer if schemas are still needed
- If discovered schemas were deleted: re-enable discovery and send events to regenerate schemas

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
