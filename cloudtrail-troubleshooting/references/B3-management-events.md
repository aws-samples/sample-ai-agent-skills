---
title: "B3 — Management Events"
description: "Troubleshoot management event logging issues"
status: active
severity: MEDIUM
triggers:
  - "management events missing"
  - "control plane not logged"
  - "API calls not recorded"
  - "read-only events"
owner: devops-agent
objective: "Ensure management events are properly logged"
context: "Management events are logged by default (first copy free). Issues arise when event selectors filter read-only or write-only events, trails are stopped, or events are in a different region."
---

## Phase 1 — Triage

MUST:
- Check event selectors for management events: `aws cloudtrail get-event-selectors --trail-name <trail-name>`
- Verify trail is logging: `aws cloudtrail get-trail-status --name <trail-name>`
- Check if read/write filtering is applied
- Search for specific events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<event> --max-results 10`

SHOULD:
- Verify trail is multi-region for cross-region events
- Check if exclude management event sources is configured
- Verify global service events are included

MAY:
- Check Event History for the last 90 days
- Review if multiple trails have different configurations

## Phase 2 — Remediate

MUST:
- Ensure management events are enabled (ReadWriteType: All)
- Start trail logging if stopped
- Enable multi-region for comprehensive coverage

SHOULD:
- Include global service events (IAM, STS, CloudFront)
- Remove unnecessary management event source exclusions

MAY:
- Configure separate trails for read-only vs write-only if needed
- Set up EventBridge rules for critical management events

## Common Issues

- symptoms: "Read-only management events missing"
  diagnosis: "Event selector configured for WriteOnly."
  resolution: "Change ReadWriteType to All to capture both read and write events."

- symptoms: "IAM events not appearing"
  diagnosis: "Global service events not included or trail not multi-region."
  resolution: "Enable IncludeGlobalServiceEvents and use multi-region trail."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Ensure management events enabled (ReadWriteType: All) | GREEN | Restores full management event logging |
| Start trail logging | GREEN | Restores event recording |
| Enable multi-region | GREEN | Extends coverage |
| Include global service events | GREEN | Additive coverage |

## Escalation Conditions

- Management events not being logged — escalate immediately to security team
- Trail stopped in production — escalate to security operations
- Global service events (IAM, STS) missing — escalate to security team
- Event selector changes by unknown principal — escalate to security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail get-event-selectors` | HIGH | Exposes management event configuration |
| `aws cloudtrail lookup-events` | HIGH | Exposes API call history |
| `aws cloudtrail get-trail-status` | HIGH | Exposes trail operational status |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** exclude management event sources without compliance review

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Changed ReadWriteType to All | Revert to previous ReadWriteType setting |
| Enabled multi-region | Revert: `aws cloudtrail update-trail --no-is-multi-region-trail` |
| Included global service events | Revert: `aws cloudtrail update-trail --no-include-global-service-events` |

## Output Format

```yaml
root_cause: "management_events — <specific_cause>"
evidence:
  - type: event_selectors
    content: "<management event configuration>"
  - type: trail_config
    content: "<trail region and global events setting>"
severity: MEDIUM
mitigation:
  immediate: "Fix event selector configuration"
  long_term: "Use multi-region trail with all management events"
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
