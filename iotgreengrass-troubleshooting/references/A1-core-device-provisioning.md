---
title: "A1 — Core Device Provisioning Failures"
description: "Diagnose Greengrass V2 core device provisioning and registration failures"
status: active
severity: CRITICAL
triggers:
  - "core device not registering"
  - "provisioning failed"
  - "installer error"
  - "thing not created"
owner: devops-agent
objective: "Identify and resolve core device provisioning failures"
context: "Provisioning failures occur when: IAM permissions insufficient, certificate creation fails, thing already exists, or installer configuration is incorrect."
---

## Phase 1 — Triage

MUST:
- Check if thing exists: `aws iot describe-thing --thing-name <core-device-name>`
- Check core device status: `aws greengrassv2 get-core-device --core-device-thing-name <name>`
- Verify IAM role for installer: `aws iam get-role --role-name GreengrassV2TokenExchangeRole`
- Check installer logs on device at `/greengrass/v2/logs/greengrass.log`

SHOULD:
- Verify IoT policy attached to certificate: `aws iot list-attached-policies --target <cert-arn>`
- Check role alias: `aws iot describe-role-alias --role-alias GreengrassV2TokenExchangeRoleAlias`
- Verify Java 8+ is installed on the device

MAY:
- Check fleet provisioning template if using fleet provisioning
- Verify network connectivity to IoT Core endpoint

## Phase 2 — Remediate

MUST:
- Ensure IAM role has greengrass, iot, s3, and logs permissions
- Create role alias if missing: `aws iot create-role-alias --role-alias GreengrassV2TokenExchangeRoleAlias --role-arn <role-arn>`
- Verify IoT policy allows greengrass:* and iot:* operations

SHOULD:
- Re-run installer with --provision true flag
- Check system requirements (Java, Python, disk space)

## Output Format

```yaml
root_cause: "provisioning_failure — <specific_cause>"
evidence:
  - type: installer_log
    content: "<error from greengrass.log>"
severity: CRITICAL
mitigation:
  immediate: "Fix IAM role or certificate configuration"
  long_term: "Automate provisioning with fleet provisioning templates"
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
