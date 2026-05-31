---
title: "F2 — Event Ingestion Issues"
description: "Diagnose event data ingestion and format problems"
status: active
severity: MEDIUM
triggers: ["event format error", "event properties wrong", "event type missing"]
owner: devops-agent
objective: "Identify and resolve event ingestion issues"
context: "PutEvents accepts events with type, properties, and timestamp. Event properties must match interactions schema. Events are appended to interactions dataset."
---

## Phase 1 — Triage

MUST:
- Verify event format matches interactions schema
- Check event properties JSON structure
- Verify sentAt timestamp format (epoch seconds)
- Test PutEvents with minimal event

SHOULD:
- Check event type matches expected values
- Verify properties match schema field names
- Check for encoding issues

MAY:
- Monitor event ingestion rate
- Check for dropped events

## Phase 2 — Remediate

MUST:
- Match event properties to interactions schema
- Use epoch seconds for sentAt timestamp
- Include required event fields

SHOULD:
- Validate events before sending
- Use consistent event types

MAY:
- Implement event validation pipeline

## Common Issues

- symptoms: "Events accepted but don't affect recommendations"
  diagnosis: "Event properties don't match schema or event type not used in training."
  resolution: "Align event properties with schema. Use event types from training data."

- symptoms: "PutEvents returns validation error"
  diagnosis: "Event format incorrect."
  resolution: "Check JSON format. Verify required fields present."

## Output Format

```yaml
root_cause: "event_ingestion — <specific_cause>"
evidence:
  - type: event_format
    content: "<event data format>"
severity: MEDIUM
mitigation:
  immediate: "Fix event format"
  long_term: "Implement event validation"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — event ingestion processes user behavioral data
- Service Disruption Risk: LOW — ingestion issues affect data freshness
- Compliance Impact: HIGH — data ingestion must comply with privacy regulations

## Escalation Conditions
- Event ingestion pipeline fails completely
- Ingestion latency exceeds acceptable thresholds
- Ingested events contain invalid or corrupted data
- Ingestion rate exceeds service limits
- Events lost during ingestion

## Data Sensitivity
- Ingested event data — RESTRICTED
- Ingestion pipeline configurations — INTERNAL
- Event validation rules — INTERNAL
- Ingestion metrics — INTERNAL
- Never expose user event data during troubleshooting

## Prohibited Actions
- Do NOT ingest events without data validation
- Do NOT ignore ingestion failures
- Do NOT ingest data from untrusted sources
- Do NOT bypass event format validation
- Do NOT retain ingestion logs containing user data
- Do NOT share ingestion pipeline details externally

## Phase 3 — Rollback
- Restore previous ingestion configurations
- Re-process failed events after fixing issues
- Revert ingestion pipeline changes
- Verify event delivery after recovery
- Document ingestion issues and data quality findings
- Update ingestion monitoring and alerting

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
