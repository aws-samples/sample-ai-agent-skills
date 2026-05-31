---
title: "G1 — OTA Update Failures"
description: "Diagnose over-the-air update and deployment failures"
status: active
severity: HIGH
triggers:
  - "OTA update failed"
  - "component update failed"
  - "deployment timeout"
  - "update stuck"
owner: devops-agent
objective: "Identify and resolve OTA update failures"
context: "OTA failures occur when: device offline, artifact too large, deployment timeout, component lifecycle script fails, or insufficient disk space."
---

## Phase 1 — Triage

MUST:
- Check deployment status: `aws greengrassv2 get-deployment --deployment-id <id>`
- Verify device is online: `aws greengrassv2 get-core-device --core-device-thing-name <name>`
- Check deployment logs on device: `grep -i "deploy\|update" /greengrass/v2/logs/greengrass.log`
- Verify disk space for artifact download

SHOULD:
- Check artifact sizes and download progress
- Verify deployment timeout configuration
- Check if previous deployment is still in progress

MAY:
- Check network bandwidth for large artifact downloads
- Verify S3 artifact accessibility

## Phase 2 — Remediate

MUST:
- Ensure device is online before deploying
- Free disk space if insufficient
- Increase deployment timeout for large artifacts

SHOULD:
- Use delta updates for large components
- Configure deployment rate limiting for fleet updates

## Output Format

```yaml
root_cause: "ota_update_failure — <specific_cause>"
evidence:
  - type: deployment_status
    content: "<deployment error details>"
severity: HIGH
mitigation:
  immediate: "Fix connectivity or disk space"
  long_term: "Implement staged rollout with health checks"
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
