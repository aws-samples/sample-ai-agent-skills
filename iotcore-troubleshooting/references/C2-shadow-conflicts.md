---
title: "C2 — Shadow Conflicts"
description: "Diagnose device shadow version conflicts and concurrent update issues"
status: active
severity: MEDIUM
triggers:
  - "shadow conflict"
  - "version mismatch"
  - "shadow update rejected"
  - "concurrent update"
  - "optimistic locking"
owner: devops-agent
objective: "Identify and resolve shadow version conflicts"
context: "Shadow conflicts occur when multiple sources update the shadow simultaneously, version numbers don't match for optimistic locking, or desired/reported states diverge unexpectedly."
---

## Phase 1 — Triage

MUST:
- Get shadow with version: `aws iot-data get-thing-shadow --thing-name <thing-name> /dev/stdout`
- Check shadow version number and metadata timestamps
- Identify all sources updating the shadow (device, cloud app, rules)
- Check for rejected updates in IoT logs

SHOULD:
- Monitor shadow update frequency
- Check if version is included in update requests (optimistic locking)
- Review delta between desired and reported states

MAY:
- Check for race conditions in multi-source updates
- Review shadow update patterns over time

## Phase 2 — Remediate

MUST:
- Implement version-based updates to prevent conflicts
- Ensure device processes delta messages to reconcile desired/reported
- Handle update/rejected responses in device code

SHOULD:
- Use optimistic locking: include version in update payload
- Implement retry logic for version conflicts
- Separate concerns using named shadows

MAY:
- Implement conflict resolution strategy (last-write-wins vs merge)
- Use named shadows to reduce contention

## Output Format

```yaml
root_cause: "shadow_conflict — <specific_cause>"
evidence:
  - type: shadow_version
    content: "<current version and metadata>"
  - type: update_sources
    content: "<sources updating the shadow>"
severity: MEDIUM
mitigation:
  immediate: "Implement version-based updates"
  long_term: "Design shadow update strategy with conflict resolution"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never revoke or delete device certificates without confirming device decommission
- Never modify IoT policies on production devices without testing in a staging environment
- Never delete thing groups that contain active production devices
- Never alter MQTT topic rules that route critical telemetry data
- Never disable custom authorizers serving active device connections

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Reattach previous certificates if device connectivity is lost after changes
- Restore previous IoT policy versions if devices lose permissions
- Revert rule engine action configurations if message routing breaks

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
