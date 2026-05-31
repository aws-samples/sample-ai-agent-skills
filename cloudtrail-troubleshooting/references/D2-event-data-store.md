---
title: "D2 — Event Data Store"
description: "Diagnose CloudTrail Lake event data store issues"
status: active
severity: MEDIUM
triggers:
  - "event data store error"
  - "data store empty"
  - "data store creation failed"
  - "retention configuration"
owner: devops-agent
objective: "Configure and troubleshoot CloudTrail Lake event data stores"
context: "Event data stores are the storage backend for CloudTrail Lake. They have configurable retention, event types, and pricing based on data ingested. Issues include creation failures, empty stores, and retention misconfiguration."
---

## Phase 1 — Triage

MUST:
- List data stores: `aws cloudtrail list-event-data-stores`
- Get data store details: `aws cloudtrail get-event-data-store --event-data-store <eds-arn>`
- Check data store status and retention period
- Verify event types configured (management, data, Insights)

SHOULD:
- Check if data store is receiving events
- Verify IAM permissions for data store management
- Review pricing implications of retention period

MAY:
- Check CloudTrail for data store creation events
- Compare data store configuration with requirements

## Phase 2 — Remediate

MUST:
- Create data store if needed: `aws cloudtrail create-event-data-store --name <name> --retention-period <days> --advanced-event-selectors '<selectors>'`
- Fix event selectors to include required event types
- Verify data store is ENABLED

SHOULD:
- Set appropriate retention period (default 2555 days / 7 years)
- Configure advanced event selectors for specific event types
- Monitor data ingestion volume

MAY:
- Enable organization-wide data store
- Configure data store for Insights events

## Common Issues

- symptoms: "Data store created but no events"
  diagnosis: "Event selectors don't match any events or store just created."
  resolution: "Check advanced event selectors. Wait for events to be ingested."

- symptoms: "Data store creation failed"
  diagnosis: "IAM permissions or quota limit reached."
  resolution: "Check IAM permissions. Check event data store quota."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create event data store | YELLOW | Incurs ongoing ingestion and storage costs |
| Fix event selectors | YELLOW | Changes what events are ingested |
| Set retention period | YELLOW | Affects data availability and cost |

## Escalation Conditions

- Event data store creation for organization-wide events — escalate to management account administrator
- Retention period changes affecting compliance — escalate to compliance team
- Data store costs exceeding budget — escalate to cost management team
- Event data store empty when events are expected — escalate to trail configuration owner

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail list-event-data-stores` | HIGH | Exposes data store configuration |
| `aws cloudtrail get-event-data-store` | HIGH | Exposes retention, event types, and advanced selectors |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** reduce retention period below compliance requirements
- **NEVER** delete event data stores without confirming data is backed up or no longer needed

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created event data store | Delete the data store: `aws cloudtrail delete-event-data-store` (data is lost) |
| Changed event selectors | Restore previous advanced event selectors |
| Changed retention period | Update retention period back to previous value |

## Output Format

```yaml
root_cause: "event_data_store — <specific_cause>"
evidence:
  - type: data_store_config
    content: "<data store configuration>"
  - type: event_selectors
    content: "<advanced event selectors>"
severity: MEDIUM
mitigation:
  immediate: "Fix data store configuration"
  long_term: "Monitor data ingestion and optimize retention"
```

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
  - command: "lookup-events"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-trail-status"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-trails"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling CloudTrail logging"
  - "NEVER suggest deleting trail logs"
  - "NEVER reduce logging scope in production"
