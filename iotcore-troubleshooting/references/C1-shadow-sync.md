---
title: "C1 — Device Shadow Sync Issues"
description: "Diagnose device shadow synchronization problems"
status: active
severity: MEDIUM
triggers:
  - "shadow not syncing"
  - "shadow update failed"
  - "desired state not applied"
  - "reported state stale"
  - "shadow document error"
owner: devops-agent
objective: "Identify and resolve device shadow synchronization issues"
context: "Shadow sync issues include: device not subscribing to shadow topics, shadow update rejected, document too large (8 KB limit), policy not allowing shadow operations, or delta not being processed."
---

## Phase 1 — Triage

MUST:
- Get current shadow: `aws iot-data get-thing-shadow --thing-name <thing-name> /dev/stdout`
- Check shadow metadata (version, timestamps)
- Verify IoT policy allows shadow topics: `$aws/things/<thing>/shadow/*`
- Check CloudWatch metrics for shadow operations

SHOULD:
- Verify device subscribes to shadow delta topic: `$aws/things/<thing>/shadow/update/delta`
- Check shadow document size (must be under 8 KB)
- Review shadow update/accepted and update/rejected topics

MAY:
- Check if named shadows are being used vs classic shadow
- Review device-side shadow handling code

## Phase 2 — Remediate

MUST:
- Ensure policy allows shadow operations on `$aws/things/<thing>/shadow/*`
- Fix shadow document if over 8 KB limit
- Verify device subscribes to delta and accepted/rejected topics

SHOULD:
- Update shadow: `aws iot-data update-thing-shadow --thing-name <thing-name> --payload '{"state":{"desired":{"key":"value"}}}' /dev/stdout`
- Delete and recreate shadow if corrupted: `aws iot-data delete-thing-shadow --thing-name <thing-name>`

MAY:
- Use named shadows for different state aspects
- Implement shadow version checking for conflict prevention

## Output Format

```yaml
root_cause: "shadow_sync_issue — <specific_cause>"
evidence:
  - type: shadow_state
    content: "<current shadow document>"
  - type: policy_check
    content: "<shadow topic permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix policy or shadow document"
  long_term: "Implement proper shadow handling with version checking"
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
