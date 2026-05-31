---
title: "C2 — Custom Metric Issues"
description: "Diagnose custom metric creation and reporting issues"
status: active
severity: MEDIUM
triggers: ["custom metric not reporting", "metric not appearing", "device metric error", "MQTT metric publish failed"]
owner: devops-agent
objective: "Resolve custom metric issues including metric creation, device-side reporting, and security profile integration"
context: "Custom metrics extend Device Defender Detect with application-specific metrics. Devices must publish metrics via MQTT to the reserved Defender topic. Custom metrics must be created in the cloud and referenced in security profile behaviors."
---

## Phase 1 — Triage

MUST:
- List custom metrics: `aws iot list-custom-metrics`
- Check metric details: `aws iot describe-custom-metric --metric-name <name>`
- Verify security profile references the custom metric: `aws iot describe-security-profile --security-profile-name <name>`

SHOULD:
- Verify device is publishing to the correct MQTT topic: `$aws/things/{thingName}/defender/metrics/json`
- Check the metric report format matches the expected schema
- Verify the device's IoT policy allows publishing to the Defender topic

MAY:
- Subscribe to the Defender metrics topic to verify device reports
- Check CloudWatch for custom metric data points

## Phase 2 — Remediate

MUST:
- Create custom metric if not exists: `aws iot create-custom-metric --metric-name <name> --metric-type <type> --display-name <display>`
- Fix device-side metric reporting (correct MQTT topic, correct JSON format)
- Ensure device IoT policy allows iot:Publish on the Defender metrics topic

SHOULD:
- Add custom metric to security profile behavior with appropriate thresholds
- Verify metric type matches the data being reported (number, string-list, ip-address-list, number-list)

MAY:
- Set up test device to validate metric reporting pipeline
- Configure CloudWatch dashboards for custom metric visualization

## Safety Ratings

```
safety_ratings:
  - "iot list-custom-metrics: GREEN — read-only listing"
  - "iot describe-custom-metric: GREEN — read-only metric details"
  - "iot create-custom-metric: YELLOW — creates new metric definition"
  - "device IoT policy update: YELLOW — modifies device permissions"
```

## Escalation Conditions

- Custom metrics not appearing despite correct device-side reporting
- Metric type mismatch causing data ingestion failures
- Large fleet needs IoT policy updates for Defender topic access
- Custom metric data not available for ML Detect training

## Data Sensitivity

- `aws iot list-custom-metrics`: LOW — returns metric names
- `aws iot describe-custom-metric`: LOW — returns metric configuration
- `device metric reports`: MEDIUM — contain device-specific operational data
- `MQTT topic subscriptions`: MEDIUM — reveal device communication patterns

## Prohibited Actions

- NEVER delete custom metrics that are referenced by active security profiles
- NEVER modify device IoT policies to grant overly broad MQTT publish permissions
- NEVER publish fake metric data to test detection without clearly marking it as test data
- NEVER expose device metric reports containing IP addresses or network details publicly

## Phase 3 — Rollback

1. If custom metric was created incorrectly, delete and recreate: `aws iot delete-custom-metric --metric-name <name>`
2. If device IoT policy was modified, restore previous version
3. If security profile behavior was changed, restore previous thresholds
4. Verify devices are still reporting metrics correctly
5. Document the custom metric configuration and reporting requirements

## Common Issues

- symptoms: "Custom metric not appearing in security profile violations"
  diagnosis: "Device not publishing to the correct MQTT topic or metric not added to security profile behavior."
  resolution: "Verify MQTT topic, JSON format, and add metric to security profile behavior."

## Output Format

```yaml
root_cause: "custom_metric — <specific_cause>"
evidence:
  - type: metric_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix metric reporting pipeline"
  long_term: "Automate metric validation and monitoring"
```

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
