---
title: "E2 — Stream Storage Issues"
description: "Diagnose stream manager local storage issues"
status: active
severity: MEDIUM
triggers:
  - "storage full"
  - "data lost"
  - "overflow"
  - "disk space stream"
owner: devops-agent
objective: "Identify and resolve stream manager local storage issues"
context: "Storage issues occur when: disk full, storage limit reached, overflow policy misconfigured, or old data not cleaned up."
---

## Phase 1 — Triage

MUST:
- Check disk usage: `df -h /greengrass/`
- Check stream manager storage config in component configuration
- Review stream manager logs for storage warnings
- Check overflow strategy: REJECT_NEW_DATA or OVERWRITE_OLDEST_DATA

SHOULD:
- Calculate data ingestion rate vs export rate
- Check if export failures are causing data accumulation
- Verify stream TTL settings

MAY:
- List active streams and their sizes
- Check for orphaned stream data

## Phase 2 — Remediate

MUST:
- Free disk space or increase storage allocation
- Fix export failures to drain accumulated data
- Configure appropriate overflow strategy

SHOULD:
- Set stream TTL to auto-delete old data
- Monitor disk usage with CloudWatch agent

## Output Format

```yaml
root_cause: "stream_storage — <specific_cause>"
evidence:
  - type: disk_usage
    content: "<storage utilization>"
severity: MEDIUM
mitigation:
  immediate: "Free disk space and fix exports"
  long_term: "Configure storage limits and TTL policies"
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
