---
title: "F1 — Event Tracking Issues"
description: "Diagnose event tracker and real-time event problems"
status: active
severity: MEDIUM
triggers: ["events not tracked", "PutEvents error", "event tracker error"]
owner: devops-agent
objective: "Identify and resolve event tracking issues"
context: "Event trackers enable real-time event ingestion via PutEvents. Events influence recommendations within 2-15 minutes. Tracker must be created for the dataset group."
---

## Phase 1 — Triage

MUST:
- List event trackers: `aws personalize list-event-trackers --dataset-group-arn <arn>`
- Verify event tracker exists and is ACTIVE
- Check PutEvents call format (trackingId, userId, sessionId, eventList)
- Verify trackingId matches event tracker

SHOULD:
- Check event schema matches expected format
- Verify events are being sent (CloudWatch metrics)
- Test with sample PutEvents call

MAY:
- Check for PutEvents throttling
- Review event data quality

## Phase 2 — Remediate

MUST:
- Create event tracker if not exists
- Use correct trackingId in PutEvents
- Include required fields (userId, sessionId, eventList)

SHOULD:
- Include event type and properties
- Send events in real-time (not batched)

MAY:
- Implement event tracking monitoring

## Common Issues

- symptoms: "PutEvents returns error"
  diagnosis: "Invalid trackingId or missing required fields."
  resolution: "Use trackingId from event tracker. Include userId, sessionId, eventList."

- symptoms: "Events sent but recommendations don't change"
  diagnosis: "Events take 2-15 minutes to influence. New items need retrain."
  resolution: "Wait 15 minutes. Retrain for new items/users."

## Output Format

```yaml
root_cause: "event_tracking — <specific_cause>"
evidence:
  - type: event_tracker
    content: "<event tracker details>"
severity: MEDIUM
mitigation:
  immediate: "Fix event tracker or PutEvents format"
  long_term: "Implement event tracking monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — event tracking captures user behavioral data
- Service Disruption Risk: LOW — tracking issues affect model freshness, not availability
- Compliance Impact: HIGH — event tracking must comply with consent and privacy requirements

## Escalation Conditions
- Event tracking completely stops
- Events contain PII that should not be tracked
- Event volume exceeds expected thresholds
- Event tracker permissions revoked
- Events not being incorporated into model updates

## Data Sensitivity
- User interaction events — RESTRICTED
- Event tracker configurations — INTERNAL
- User session data — RESTRICTED
- Event metadata — INTERNAL
- Never log individual user events in troubleshooting

## Prohibited Actions
- Do NOT track user events without proper consent
- Do NOT log individual user events during troubleshooting
- Do NOT track events containing PII
- Do NOT share event data with unauthorized parties
- Do NOT disable event tracking without understanding model impact
- Do NOT retain event data beyond approved retention periods

## Phase 3 — Rollback
- Restore event tracker configurations
- Re-enable event tracking if disabled
- Verify event delivery after changes
- Restore event tracker permissions
- Document event tracking issues and consent compliance
- Update event tracking procedures

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
