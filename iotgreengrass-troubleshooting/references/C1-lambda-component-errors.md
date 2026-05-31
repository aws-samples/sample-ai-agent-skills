---
title: "C1 — Lambda Component Errors"
description: "Diagnose Lambda component execution failures on Greengrass"
status: active
severity: HIGH
triggers:
  - "Lambda component failed"
  - "function error"
  - "Lambda not running"
  - "handler error"
owner: devops-agent
objective: "Identify and resolve Lambda component execution failures"
context: "Lambda component errors occur when: runtime missing, handler misconfigured, memory insufficient, or IPC permissions denied."
---

## Phase 1 — Triage

MUST:
- Check component status: `aws greengrassv2 list-installed-components --core-device-thing-name <name>`
- Review component logs: `cat /greengrass/v2/logs/<component-name>.log`
- Verify Lambda runtime is installed on device (Python, Node.js, Java)
- Check component recipe for handler configuration

SHOULD:
- Verify IPC authorization policy in recipe
- Check memory allocation in component configuration
- Verify environment variables are set correctly

MAY:
- Test Lambda function locally on the device
- Check for missing dependencies or libraries

## Phase 2 — Remediate

MUST:
- Install required runtime on the device
- Fix handler path in component recipe
- Grant IPC permissions for required services

SHOULD:
- Increase memory allocation if OOM errors
- Add error handling and logging to Lambda function

## Output Format

```yaml
root_cause: "lambda_component_error — <specific_cause>"
evidence:
  - type: component_log
    content: "<error from component log>"
severity: HIGH
mitigation:
  immediate: "Fix runtime or handler configuration"
  long_term: "Implement component health monitoring"
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
