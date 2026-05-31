---
title: "E2 — Device Shadow Sync"
description: "Diagnose device shadow synchronization failures and state conflicts"
status: active
severity: HIGH
triggers: ["shadow sync failed", "shadow conflict", "desired state not applied", "reported state stale", "shadow version mismatch"]
owner: devops-agent
objective: "Resolve device shadow synchronization issues between device and cloud"
context: "Device shadows maintain desired and reported state. Shadows use last-writer-wins for conflicts. Named shadows allow multiple state documents per thing. Shadow version enables optimistic locking."
---

## Phase 1 — Triage

MUST:
- Get the device shadow: `aws iot-data get-thing-shadow --thing-name <name> /dev/stdout | python -m json.tool`
- Check shadow metadata for timestamps: review the metadata section for last update times
- Compare desired vs reported state for discrepancies

SHOULD:
- List named shadows: `aws iot-data list-named-shadows-for-thing --thing-name <name>`
- Check device connectivity: `aws iot search-index --query-string "thingName:<name> AND connectivity.connected:true"`
- Verify device is subscribed to shadow delta topic: $aws/things/<name>/shadow/update/delta

MAY:
- Check CloudWatch Logs for shadow update errors: `aws logs filter-log-events --log-group-name AWSIotLogsV2 --filter-pattern "shadow"`
- Review shadow update history via CloudTrail
- Check shadow document size (max 8KB for classic, 64KB for named shadows in some regions)

## Phase 2 — Remediate

MUST:
- If desired state not reaching device: verify device subscribes to shadow delta topic
- If reported state stale: verify device publishes to $aws/things/<name>/shadow/update
- For version conflicts: implement optimistic locking using shadow version in update requests

SHOULD:
- Clear stale desired state if device has already applied it: update shadow to remove fulfilled desired properties
- Ensure shadow document does not exceed size limits
- Verify IoT policy allows shadow operations: iot:GetThingShadow, iot:UpdateThingShadow, iot:DeleteThingShadow

MAY:
- Delete and recreate shadow if corrupted: `aws iot-data delete-thing-shadow --thing-name <name>`
- Implement application-level conflict resolution using shadow version numbers
- Use named shadows to separate different state domains

## Safety Ratings

```
safety_ratings:
  - "get-thing-shadow: GREEN — read-only state retrieval"
  - "list-named-shadows-for-thing: GREEN — read-only enumeration"
  - "update-thing-shadow: YELLOW — modifies device desired/reported state"
  - "delete-thing-shadow: RED — permanently removes shadow state document"
```

## Escalation Conditions

- Shadow sync failures affect device operation or safety-critical state
- Persistent version conflicts indicate race condition in application architecture
- Shadow document corruption cannot be resolved without device-side intervention
- Large fleet experiencing simultaneous shadow sync failures

## Data Sensitivity

- `aws iot-data get-thing-shadow`: HIGH — contains device operational state and configuration
- `aws iot-data update-thing-shadow`: HIGH — modifies device behavior through desired state
- `aws iot-data delete-thing-shadow`: HIGH — removes all device state
- Shadow delta messages: MEDIUM — contain state change details

## Prohibited Actions

- NEVER delete a device shadow in production without understanding the impact on device behavior
- NEVER overwrite reported state from the cloud — only devices should update reported state
- NEVER ignore shadow version conflicts — they indicate concurrent modification issues
- NEVER store sensitive credentials or secrets in device shadows

## Phase 3 — Rollback

1. If shadow was incorrectly updated, restore previous desired state: `aws iot-data update-thing-shadow --thing-name <name> --payload '{"state":{"desired":<previous-state>}}'`
2. If shadow was deleted, device will recreate it on next connection with current reported state
3. If named shadow was corrupted, delete and let application recreate: `aws iot-data delete-thing-shadow --thing-name <name> --shadow-name <shadow-name>`
4. Verify device reconnects and reports current state after shadow restoration
5. Monitor shadow sync for 15 minutes to confirm stability

## Common Issues

- symptoms: "Desired state set but device never applies it"
  diagnosis: "Device is not connected or not subscribed to the shadow delta topic."
  resolution: "Verify device connectivity. Ensure device subscribes to $aws/things/<name>/shadow/update/delta and processes delta messages."

- symptoms: "Shadow update returns 409 Conflict error"
  diagnosis: "Shadow version in the update request does not match current version. Another writer updated the shadow concurrently."
  resolution: "Read current shadow version, merge changes, and retry update with correct version number."

## Output Format

```yaml
root_cause: "shadow_sync — <specific_cause>"
evidence:
  - type: shadow_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix shadow synchronization"
  long_term: "Implement version-based conflict resolution"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
