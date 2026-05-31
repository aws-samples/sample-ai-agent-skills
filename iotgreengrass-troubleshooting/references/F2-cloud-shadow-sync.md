---
title: "F2 — Cloud Shadow Sync Issues"
description: "Diagnose shadow synchronization between local and cloud"
status: active
severity: MEDIUM
triggers:
  - "cloud shadow not syncing"
  - "shadow out of sync"
  - "cloud shadow stale"
owner: devops-agent
objective: "Identify and resolve cloud shadow synchronization issues"
context: "Cloud sync issues occur when: sync not enabled, IoT Core policy missing shadow permissions, network connectivity lost, or shadow conflicts between local and cloud."
---

## Phase 1 — Triage

MUST:
- Check ShadowManager sync configuration for cloud sync enabled
- Verify IoT Core policy allows shadow topics: `aws iot get-policy --policy-name <policy>`
- Check device connectivity to IoT Core
- Review ShadowManager logs for sync errors

SHOULD:
- Verify shadow sync direction (deviceToCloud, cloudToDevice, or both)
- Check for shadow version conflicts between local and cloud
- Verify shadow exists in cloud: `aws iot-data get-thing-shadow --thing-name <name> /dev/stdout`

MAY:
- Compare local and cloud shadow documents for drift
- Check CloudWatch IoT shadow metrics

## Phase 2 — Remediate

MUST:
- Enable cloud sync in ShadowManager configuration
- Add shadow topic permissions to IoT Core policy
- Fix network connectivity

SHOULD:
- Configure sync direction based on use case
- Implement conflict resolution strategy

## Output Format

```yaml
root_cause: "cloud_shadow_sync — <specific_cause>"
evidence:
  - type: sync_config
    content: "<sync configuration>"
severity: MEDIUM
mitigation:
  immediate: "Enable sync and fix permissions"
  long_term: "Implement shadow sync monitoring"
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
- Never remove core device provisioning without confirming all edge workloads are migrated
- Never force-deploy components to production core devices without staging validation
- Never delete component versions that are actively deployed to edge devices
- Never modify MQTT bridge configurations on devices running critical edge processing
- Never cancel OTA updates that are partially applied to production devices

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Trigger a rollback deployment to restore previous component versions on core devices
- Restore previous MQTT bridge and topic mapping configurations if messaging breaks
- Revert nucleus configuration changes if core device connectivity is lost

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
