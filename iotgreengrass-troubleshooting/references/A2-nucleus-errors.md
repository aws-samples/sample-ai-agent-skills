---
title: "A2 — Nucleus Errors"
description: "Diagnose Greengrass nucleus startup and runtime errors"
status: active
severity: CRITICAL
triggers:
  - "nucleus not starting"
  - "greengrass service failed"
  - "nucleus crash"
  - "runtime error"
owner: devops-agent
objective: "Identify and resolve nucleus startup and runtime failures"
context: "Nucleus errors occur when: configuration is invalid, Java runtime missing, disk full, permissions incorrect, or nucleus version incompatible."
---

## Phase 1 — Triage

MUST:
- Check nucleus logs: `sudo tail -100 /greengrass/v2/logs/greengrass.log`
- Check systemd service status: `sudo systemctl status greengrass`
- Verify Java version: `java -version` (requires Java 8+)
- Check disk space: `df -h /greengrass/`

SHOULD:
- Check nucleus config: `cat /greengrass/v2/config/effectiveConfig.yaml`
- Verify file permissions on /greengrass/v2/ directory
- Check if port 8883 is accessible for MQTT

MAY:
- Check kernel logs for OOM kills: `dmesg | grep -i oom`
- Verify system clock is synchronized

## Phase 2 — Remediate

MUST:
- Fix Java installation if missing or wrong version
- Correct file permissions: `sudo chown -R ggc_user:ggc_group /greengrass/v2/`
- Free disk space if full

SHOULD:
- Reset nucleus config if corrupted: re-run installer
- Restart service: `sudo systemctl restart greengrass`

## Output Format

```yaml
root_cause: "nucleus_error — <specific_cause>"
evidence:
  - type: nucleus_log
    content: "<error from greengrass.log>"
severity: CRITICAL
mitigation:
  immediate: "Fix runtime environment or configuration"
  long_term: "Monitor nucleus health with CloudWatch"
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
