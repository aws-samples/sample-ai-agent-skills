---
title: "D1 — Archive Configuration Issues"
description: "Diagnose EventBridge archive creation and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "archive configuration"
  - "archive not working"
  - "archive creation"
  - "events not archived"
  - "archive retention"
owner: devops-agent
objective: "Identify and resolve archive configuration issues on EventBridge"
context: "EventBridge archives store events from an event bus for later replay. Archives can filter events using event patterns. Retention can be indefinite or time-limited. Archive issues include configuration errors, event pattern mismatches, and storage concerns."
---

## Phase 1 — Triage

MUST:
- List archives: `aws events list-archives`
- Describe the archive: `aws events describe-archive --archive-name <archive-name>`
- Check archive State (ENABLED, DISABLED, CREATING, CREATE_FAILED)
- Verify the event pattern matches desired events

SHOULD:
- Check archive event count and size
- Verify the source event bus is correct
- Check retention days configuration
- Review CloudTrail for archive configuration changes

MAY:
- Test the event pattern against sample events
- Monitor archive storage growth
- Check for archive quotas

## Phase 2 — Remediate

MUST:
- Fix archive event pattern to match desired events
- Enable the archive if disabled
- Verify the source event bus has events flowing

SHOULD:
- Set appropriate retention period to manage storage costs
- Test archive by replaying a small time window
- Document archive configuration and purpose

MAY:
- Set up monitoring for archive size and event count
- Implement archive lifecycle management

## Common Issues

- symptoms: "Archive created but no events stored"
  diagnosis: "Event pattern does not match any events on the bus, or no events are flowing."
  resolution: "Verify event pattern matches actual events. Check event bus has activity."

- symptoms: "Archive in CREATE_FAILED state"
  diagnosis: "Archive creation failed due to invalid event pattern or permissions."
  resolution: "Check event pattern syntax. Verify IAM permissions. Recreate the archive."

- symptoms: "Archive storage growing unexpectedly"
  diagnosis: "Event pattern is too broad, capturing more events than intended."
  resolution: "Narrow the event pattern. Set retention days to limit storage duration."

## Output Format

```yaml
root_cause: "archive_configuration — <specific_cause>"
evidence:
  - type: archive
    content: "<archive status and configuration>"
  - type: event_pattern
    content: "<archive event pattern>"
  - type: metrics
    content: "<event count and storage size>"
severity: MEDIUM
mitigation:
  immediate: "Fix archive configuration or event pattern"
  long_term: "Implement archive monitoring and lifecycle management"
```


## Safety Ratings
```
safety_ratings:
  - "List and describe archives: GREEN — read-only API call"
  - "Check archive event count: GREEN — read-only API call"
  - "Verify event pattern: GREEN — read-only analysis"
  - "Enable disabled archive: YELLOW — begins storing events, incurs storage cost"
  - "Update archive event pattern: YELLOW — changes which events are archived"
  - "Set retention days: YELLOW — may cause older archived events to be deleted"
```

## Escalation Conditions
- Archive used for production event replay and disaster recovery
- Archive in CREATE_FAILED state blocking replay capability
- Archive storage growing unexpectedly causing cost concerns
- Event pattern changes needed for compliance or audit requirements

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Archived events: historical business event data"
    - "Event patterns: business logic for event selection"
    - "Archive retention settings: data retention policy"
  handling: "Archived events contain historical business data. Apply appropriate access controls and retention policies."
```

## Prohibited Actions
- NEVER delete an archive that serves as the only replay source for disaster recovery
- NEVER set retention days to a value shorter than compliance requirements
- NEVER modify archive event patterns without understanding impact on replay capability

## Phase 3 — Rollback
- If archive was disabled: re-enable the archive (events during the disabled period are lost)
- If event pattern was changed: restore previous event pattern configuration
- If retention days was reduced: increase retention days (already-expired events cannot be recovered)
- If archive was deleted: recreate the archive (historical events are permanently lost)

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
