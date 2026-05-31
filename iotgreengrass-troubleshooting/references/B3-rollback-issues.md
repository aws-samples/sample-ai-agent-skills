---
title: "B3 — Rollback Issues"
description: "Diagnose deployment rollback failures"
status: active
severity: HIGH
triggers:
  - "rollback failed"
  - "stuck after rollback"
  - "previous version not restored"
owner: devops-agent
objective: "Identify and resolve deployment rollback failures"
context: "Rollback issues occur when: rollback not configured, previous artifacts deleted, or nucleus state corrupted."
---

## Phase 1 — Triage

MUST:
- Check deployment rollback config: `aws greengrassv2 get-deployment --deployment-id <id>`
- Check effective deployments: `aws greengrassv2 list-effective-deployments --core-device-thing-name <name>`
- Review rollback logs: `grep -i "rollback" /greengrass/v2/logs/greengrass.log`

SHOULD:
- Verify previous component versions still exist in S3
- Check if rollback timeout was exceeded

MAY:
- Manually redeploy previous known-good configuration

## Phase 2 — Remediate

MUST:
- Configure rollback in deployment: set failureHandlingPolicy to ROLLBACK
- Ensure previous artifacts remain in S3
- Create new deployment with known-good component versions if rollback fails

SHOULD:
- Set appropriate rollback timeout values
- Keep artifact versions in S3 for rollback capability

## Output Format

```yaml
root_cause: "rollback_failure — <specific_cause>"
evidence:
  - type: deployment_config
    content: "<rollback configuration>"
severity: HIGH
mitigation:
  immediate: "Manually deploy known-good configuration"
  long_term: "Always configure rollback policies"
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
