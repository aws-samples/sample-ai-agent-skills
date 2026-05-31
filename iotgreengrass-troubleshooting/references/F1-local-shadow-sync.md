---
title: "F1 — Local Shadow Sync Issues"
description: "Diagnose local shadow service synchronization issues"
status: active
severity: MEDIUM
triggers:
  - "shadow not updating"
  - "local shadow error"
  - "IPC shadow failed"
  - "shadow manager error"
owner: devops-agent
objective: "Identify and resolve local shadow synchronization issues"
context: "Local shadow issues occur when: ShadowManager not deployed, IPC permissions missing, shadow document too large, or version conflicts."
---

## Phase 1 — Triage

MUST:
- Verify ShadowManager deployed: check for aws.greengrass.ShadowManager in installed components
- Check ShadowManager logs: `cat /greengrass/v2/logs/aws.greengrass.ShadowManager.log`
- Verify component has IPC authorization for shadow operations
- Check shadow document size (8 KB limit)

SHOULD:
- Verify shadow name matches between producer and consumer components
- Check for version conflicts in shadow updates
- Review IPC authorization policy in component recipe

MAY:
- Test shadow operations via Greengrass CLI: `greengrass-cli get-shadow --thing-name <name>`

## Phase 2 — Remediate

MUST:
- Deploy ShadowManager component if missing
- Add IPC authorization for shadow operations in component recipe
- Reduce shadow document size if exceeding 8 KB

SHOULD:
- Use named shadows to separate concerns
- Implement version checking for concurrent updates

## Output Format

```yaml
root_cause: "local_shadow_sync — <specific_cause>"
evidence:
  - type: shadow_manager_log
    content: "<shadow error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix IPC permissions or deploy ShadowManager"
  long_term: "Implement shadow versioning strategy"
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
