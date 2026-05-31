---
title: "E1 — Stream Export Failures"
description: "Diagnose stream manager export failures to cloud services"
status: active
severity: HIGH
triggers:
  - "export failed"
  - "S3 upload failed"
  - "Kinesis put failed"
  - "stream manager error"
owner: devops-agent
objective: "Identify and resolve stream manager export failures"
context: "Export failures occur when: TES role lacks permissions, destination doesn't exist, network connectivity lost, or payload format invalid."
---

## Phase 1 — Triage

MUST:
- Check stream manager logs: `cat /greengrass/v2/logs/aws.greengrass.StreamManager.log`
- Verify TES role has permissions for export destination (s3:PutObject, kinesis:PutRecord, etc.)
- Check export destination exists and is accessible
- Verify network connectivity from device

SHOULD:
- Check stream manager configuration for export settings
- Verify export retry policy and backoff settings
- Check local storage usage for data accumulation

MAY:
- Test export destination access manually from device using AWS CLI
- Check CloudWatch metrics on destination service for errors

## Phase 2 — Remediate

MUST:
- Add required permissions to TES IAM role
- Create export destination if missing
- Fix network connectivity

SHOULD:
- Configure appropriate retry policies
- Set up export failure alerting

## Output Format

```yaml
root_cause: "stream_export_failure — <specific_cause>"
evidence:
  - type: stream_manager_log
    content: "<export error details>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or destination access"
  long_term: "Implement export monitoring and retry policies"
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
