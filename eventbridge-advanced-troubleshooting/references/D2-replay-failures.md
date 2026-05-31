---
title: "D2 — Replay Failures"
description: "Diagnose EventBridge event replay failures and issues"
status: active
severity: MEDIUM
triggers:
  - "replay failed"
  - "replay error"
  - "replay not working"
  - "replay events"
  - "replay stuck"
owner: devops-agent
objective: "Identify and resolve event replay failures on EventBridge"
context: "Event replay delivers archived events back to the same event bus. Replays can fail due to archive issues, time window problems, or rule processing errors. Replayed events include a replay-name header and are processed by all rules on the bus."
---

## Phase 1 — Triage

MUST:
- Describe the replay: `aws events describe-replay --replay-name <replay-name>`
- Check replay State (STARTING, RUNNING, COMPLETED, FAILED, CANCELLING)
- Verify the source archive exists and has events: `aws events describe-archive --archive-name <archive-name>`
- Check the replay time window (EventStartTime, EventEndTime)

SHOULD:
- Verify the time window contains archived events
- Check rules on the target event bus for processing errors
- Review replay progress (EventsReplayedCount)
- Check CloudTrail for replay errors

MAY:
- Monitor target processing during replay
- Check for duplicate event processing
- Verify replay-name header filtering in rules

## Phase 2 — Remediate

MUST:
- Fix the identified issue (archive, time window, permissions)
- Ensure the archive has events in the specified time window
- Verify rules on the bus can handle replayed events

SHOULD:
- Use narrow time windows to limit replay scope
- Filter replayed events in rules using the replay-name header
- Monitor target services during replay for capacity

MAY:
- Implement idempotent event processing for replay safety
- Document replay procedures

## Common Issues

- symptoms: "Replay completes but no events delivered"
  diagnosis: "The time window does not contain any archived events."
  resolution: "Verify the archive has events in the specified time range. Adjust EventStartTime/EventEndTime."

- symptoms: "Replay causes duplicate processing"
  diagnosis: "All rules on the bus process replayed events, including rules that processed them originally."
  resolution: "Add replay-name header filter to rules that should not process replayed events."

- symptoms: "Replay fails with FAILED state"
  diagnosis: "Archive is in an invalid state or permissions are insufficient."
  resolution: "Verify archive is ENABLED. Check IAM permissions for events:StartReplay."

## Output Format

```yaml
root_cause: "replay_failure — <specific_cause>"
evidence:
  - type: replay
    content: "<replay status and configuration>"
  - type: archive
    content: "<archive status and event count>"
  - type: time_window
    content: "<replay time window and event availability>"
severity: MEDIUM
mitigation:
  immediate: "Fix archive or time window and retry replay"
  long_term: "Implement replay procedures and idempotent processing"
```


## Safety Ratings
```
safety_ratings:
  - "Describe replay status: GREEN — read-only API call"
  - "Describe source archive: GREEN — read-only API call"
  - "Check replay time window: GREEN — read-only analysis"
  - "Start replay: YELLOW — delivers archived events to the bus, all rules will process them"
  - "Cancel replay: GREEN — stops event delivery, no data loss"
```

## Escalation Conditions
- Replay needed for production data recovery
- Replay causing duplicate event processing in downstream systems
- Replay failed and events need to be reprocessed urgently
- Large replay window causing target service capacity concerns

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Replayed events: historical business event data"
    - "Replay time window: incident or recovery time range"
    - "Events replayed count: volume of reprocessed data"
  handling: "Replayed events contain historical business data. Ensure targets can handle replayed events securely."
```

## Prohibited Actions
- NEVER start a replay without confirming downstream targets can handle duplicate events
- NEVER replay a large time window without first testing with a narrow window
- NEVER start a replay during peak business hours without capacity planning
- NEVER ignore the replay-name header — use it to filter replayed events in rules

## Phase 3 — Rollback
- If replay was started: cancel the replay if it's causing issues (already-delivered events cannot be recalled)
- If replay caused duplicate processing: implement deduplication in downstream targets
- If replay time window was wrong: cancel and start a new replay with the correct time window
- If rules processed replayed events incorrectly: add replay-name header filter to prevent future issues

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
