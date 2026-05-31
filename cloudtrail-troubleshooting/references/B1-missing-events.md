---
title: "B1 — Missing Events"
description: "Diagnose why expected events are missing from CloudTrail"
status: active
severity: HIGH
triggers:
  - "event not found"
  - "missing events"
  - "API call not logged"
  - "no record of action"
owner: devops-agent
objective: "Identify why expected events are not appearing in CloudTrail"
context: "Events can appear missing due to delivery delay (5-15 min), event type not enabled (data events), region mismatch, event selectors filtering, or the API not being supported by CloudTrail."
---

## Phase 1 — Triage

MUST:
- Search Event History: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<event-name> --start-time <time> --end-time <time>`
- Check event selectors: `aws cloudtrail get-event-selectors --trail-name <trail-name>`
- Verify trail is logging: `aws cloudtrail get-trail-status --name <trail-name>`
- Check if event is a data event (requires explicit enablement)

SHOULD:
- Verify the time range accounts for delivery delay (add 15 min buffer)
- Check if the event occurred in a different region
- Verify the trail is multi-region if searching cross-region events
- Check if advanced event selectors are filtering the event

MAY:
- Check if the API is supported by CloudTrail
- Search S3 logs directly for the event
- Check CloudTrail Lake if configured

## Phase 2 — Remediate

MUST:
- Enable data events if the missing event is a data plane operation
- Ensure trail covers the correct region
- Verify event selectors include the event type

SHOULD:
- Enable multi-region trail for comprehensive coverage
- Use advanced event selectors for granular control
- Verify with a test API call after fixing

MAY:
- Configure CloudTrail Lake for SQL-based event search
- Set up EventBridge rules for critical event monitoring

## Common Issues

- symptoms: "S3 GetObject not in CloudTrail"
  diagnosis: "S3 data events not enabled on the trail."
  resolution: "Enable S3 data events in event selectors for the specific bucket."

- symptoms: "Event from another region not found"
  diagnosis: "Trail is single-region, event occurred in different region."
  resolution: "Enable multi-region trail or create trail in the event's region."

- symptoms: "Event not found but happened minutes ago"
  diagnosis: "CloudTrail delivery delay (5-15 minutes)."
  resolution: "Wait 15 minutes and search again. Use EventBridge for near-real-time."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable data events for specific resources | YELLOW | Increases logging volume and cost |
| Enable multi-region trail | GREEN | Extends coverage without affecting existing logging |
| Verify with test API call | GREEN | Read-only verification |
| Configure CloudTrail Lake | GREEN | New resource creation |

## Escalation Conditions

- Critical security events missing from CloudTrail — escalate immediately to security team
- Event selectors filtering out required compliance events — escalate to compliance team
- Data events needed for forensic investigation — escalate to security operations
- Multi-region trail changes in organization — escalate to management account administrator

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail lookup-events` | HIGH | Exposes API call history with principals and parameters |
| `aws cloudtrail get-event-selectors` | HIGH | Exposes what events are being logged |
| `aws cloudtrail get-trail-status` | HIGH | Exposes trail operational status |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** narrow event selectors without understanding the compliance impact

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled data events | Remove data event selectors: `aws cloudtrail put-event-selectors` with original config |
| Enabled multi-region trail | Revert to single-region: `aws cloudtrail update-trail --no-is-multi-region-trail` |
| Modified event selectors | Restore previous event selector configuration |

## Output Format

```yaml
root_cause: "missing_events — <specific_cause>"
evidence:
  - type: event_selectors
    content: "<current event selector configuration>"
  - type: trail_config
    content: "<trail region and multi-region setting>"
severity: HIGH
mitigation:
  immediate: "Enable the missing event type or fix trail configuration"
  long_term: "Use multi-region trail with appropriate event selectors"
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
