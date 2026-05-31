---
title: "D1 — MQTT Bridge Configuration"
description: "Diagnose MQTT bridge component configuration issues"
status: active
severity: HIGH
triggers:
  - "bridge not working"
  - "messages not forwarded"
  - "MQTT bridge error"
  - "local to cloud failed"
owner: devops-agent
objective: "Identify and resolve MQTT bridge configuration issues"
context: "Bridge config issues occur when: component not deployed, topic mappings missing, client device auth not configured, or bridge component version incompatible."
---

## Phase 1 — Triage

MUST:
- Verify bridge component deployed: `aws greengrassv2 list-installed-components --core-device-thing-name <name>` (look for aws.greengrass.clientdevices.mqtt.Bridge)
- Check bridge component logs: `cat /greengrass/v2/logs/aws.greengrass.clientdevices.mqtt.Bridge.log`
- Review bridge configuration in deployment for topic mappings
- Verify client device auth component is deployed

SHOULD:
- Check if aws.greengrass.clientdevices.mqtt.Moquette (local MQTT broker) is deployed
- Verify client device certificates are authorized
- Test local MQTT connectivity from client device

MAY:
- Check IoT Core MQTT metrics for messages arriving from bridge
- Verify QoS settings in topic mappings

## Phase 2 — Remediate

MUST:
- Deploy MQTT bridge component if missing
- Add topic mappings in bridge configuration
- Deploy client device auth component

SHOULD:
- Configure both directions (LocalMqtt↔IotCore) as needed
- Set appropriate QoS for each topic mapping

## Output Format

```yaml
root_cause: "mqtt_bridge_config — <specific_cause>"
evidence:
  - type: bridge_config
    content: "<topic mapping configuration>"
severity: HIGH
mitigation:
  immediate: "Fix bridge configuration and topic mappings"
  long_term: "Document all required topic mappings"
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
