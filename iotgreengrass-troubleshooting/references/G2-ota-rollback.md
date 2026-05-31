---
title: "G2 — OTA Rollback Issues"
description: "Diagnose OTA update rollback and recovery issues"
status: active
severity: HIGH
triggers:
  - "rollback not working"
  - "device bricked"
  - "cannot recover"
  - "rollback timeout"
owner: devops-agent
objective: "Identify and resolve OTA rollback and recovery issues"
context: "Rollback issues occur when: rollback not configured, previous version artifacts deleted, nucleus corrupted, or rollback criteria not met."
---

## Phase 1 — Triage

MUST:
- Check deployment rollback configuration
- Verify rollback criteria (component health check)
- Check if previous component versions are available
- Review nucleus logs for rollback attempts

SHOULD:
- Check if deployment has iotJobConfiguration with rollback settings
- Verify component health check scripts exist and work
- Check if device can still communicate with cloud

MAY:
- SSH to device and check component states manually
- Review /greengrass/v2/deployments/ directory for deployment history

## Phase 2 — Remediate

MUST:
- If device accessible: manually deploy known-good configuration
- If device not accessible: use physical access to reset nucleus
- Create new deployment with previous working component versions

SHOULD:
- Configure automatic rollback with health checks for future deployments
- Maintain artifact versions in S3 for rollback capability

## Output Format

```yaml
root_cause: "ota_rollback — <specific_cause>"
evidence:
  - type: rollback_status
    content: "<rollback error details>"
severity: HIGH
mitigation:
  immediate: "Manual recovery or redeploy"
  long_term: "Always configure rollback with health checks"
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
