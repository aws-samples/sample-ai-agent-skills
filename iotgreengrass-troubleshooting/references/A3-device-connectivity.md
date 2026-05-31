---
title: "A3 — Device Connectivity Issues"
description: "Diagnose core device connectivity to IoT Core"
status: active
severity: HIGH
triggers:
  - "device offline"
  - "connection lost"
  - "MQTT disconnect"
  - "cannot reach cloud"
owner: devops-agent
objective: "Identify and resolve core device connectivity failures"
context: "Connectivity issues occur when: certificate expired, network blocked, IoT endpoint unreachable, or TLS handshake fails."
---

## Phase 1 — Triage

MUST:
- Check core device status: `aws greengrassv2 get-core-device --core-device-thing-name <name>`
- Verify IoT endpoint: `aws iot describe-endpoint --endpoint-type iot:Data-ATS`
- Check certificate status: `aws iot describe-certificate --certificate-id <cert-id>`
- Test network from device: `openssl s_client -connect <endpoint>:8883`

SHOULD:
- Check nucleus connectivity logs: `grep -i "connection" /greengrass/v2/logs/greengrass.log`
- Verify DNS resolution from device
- Check firewall rules for port 8883 outbound

MAY:
- Check IoT Core MQTT metrics: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name Connect.Success --start-time <start> --end-time <end> --period 300 --statistics Sum`

## Phase 2 — Remediate

MUST:
- Renew certificate if expired
- Fix network/firewall rules to allow port 8883 outbound
- Verify IoT endpoint in nucleus configuration

SHOULD:
- Configure connection retry settings in nucleus
- Set up connectivity monitoring

## Output Format

```yaml
root_cause: "connectivity_failure — <specific_cause>"
evidence:
  - type: device_status
    content: "<core device status>"
severity: HIGH
mitigation:
  immediate: "Fix network or certificate issue"
  long_term: "Implement connectivity monitoring and alerting"
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
