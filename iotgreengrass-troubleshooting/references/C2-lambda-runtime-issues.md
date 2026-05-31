---
title: "C2 — Lambda Runtime Issues"
description: "Diagnose Lambda runtime and environment issues on Greengrass"
status: active
severity: MEDIUM
triggers:
  - "runtime not found"
  - "Python version mismatch"
  - "Node.js missing"
  - "containerized Lambda"
owner: devops-agent
objective: "Identify and resolve Lambda runtime environment issues"
context: "Runtime issues occur when: runtime version mismatch, containerization config wrong, environment variables missing, or library dependencies unavailable."
---

## Phase 1 — Triage

MUST:
- Check installed runtimes: `python3 --version`, `node --version`, `java -version`
- Verify component recipe runtime specification
- Check if containerization is enabled and Docker is available
- Review component logs for import/module errors

SHOULD:
- Verify Lambda SDK version compatibility
- Check if aws.greengrass.LambdaLauncher component is deployed
- Verify aws.greengrass.LambdaRuntimes component version

MAY:
- Check Docker status if using containerized mode: `docker info`
- Verify cgroup configuration for resource limits

## Phase 2 — Remediate

MUST:
- Install correct runtime version on device
- Deploy LambdaLauncher and LambdaRuntimes components
- Fix containerization settings in recipe

SHOULD:
- Pin runtime versions in component recipe
- Include all dependencies in Lambda deployment package

## Output Format

```yaml
root_cause: "lambda_runtime — <specific_cause>"
evidence:
  - type: runtime_check
    content: "<runtime version or error>"
severity: MEDIUM
mitigation:
  immediate: "Install correct runtime"
  long_term: "Use containerized mode for consistent environments"
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
