---
title: "B1 — Deployment Failures"
description: "Diagnose Greengrass component deployment failures"
status: active
severity: HIGH
triggers:
  - "deployment failed"
  - "component not installed"
  - "deployment stuck"
  - "FAILED status"
owner: devops-agent
objective: "Identify and resolve component deployment failures"
context: "Deployment failures occur when: recipe invalid, artifact download fails, lifecycle script errors, dependency resolution fails, or device offline."
---

## Phase 1 — Triage

MUST:
- Check deployment status: `aws greengrassv2 get-deployment --deployment-id <id>`
- List effective deployments: `aws greengrassv2 list-effective-deployments --core-device-thing-name <name>`
- Check installed components: `aws greengrassv2 list-installed-components --core-device-thing-name <name>`
- Review deployment logs on device: `grep -i "deploy" /greengrass/v2/logs/greengrass.log`

SHOULD:
- Verify component recipe: `aws greengrassv2 describe-component --arn <component-arn>`
- Check artifact accessibility in S3
- Verify TES role has s3:GetObject for artifact bucket

MAY:
- Check for deployment merge conflicts with other deployments
- Verify disk space for artifact download

## Phase 2 — Remediate

MUST:
- Fix recipe errors and create new component version
- Grant S3 access to TES role for artifact bucket
- Ensure device is online before deploying

SHOULD:
- Cancel stuck deployments: `aws greengrassv2 cancel-deployment --deployment-id <id>`
- Configure deployment rollback policies

## Output Format

```yaml
root_cause: "deployment_failure — <specific_cause>"
evidence:
  - type: deployment_status
    content: "<deployment details>"
severity: HIGH
mitigation:
  immediate: "Fix recipe or artifact access"
  long_term: "Implement deployment validation and rollback"
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
