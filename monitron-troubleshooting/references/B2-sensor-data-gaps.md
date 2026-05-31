---
title: "B2 — Sensor Data Gaps"
description: "Diagnose missing sensor data, intermittent readings, and data collection failures"
status: active
severity: HIGH
triggers: ["data gap", "missing data", "no readings", "sensor offline", "intermittent data"]
owner: devops-agent
objective: "Resolve sensor data gaps and restore continuous data collection"
context: "Sensors transmit vibration and temperature data to gateways via BLE. Data gaps can result from BLE range issues, gateway connectivity loss, sensor battery depletion, or sensor malfunction."
---

## Phase 1 — Triage

MUST:
- Check sensor data in CloudWatch: `aws cloudwatch get-metric-statistics --namespace AWS/Monitron --metric-name SensorDataPoints --dimensions Name=ProjectId,Value=<project-id> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Verify gateway is online (check LED status — solid green = connected)
- Check sensor battery status in the Monitron mobile app

SHOULD:
- Determine gap duration and pattern (intermittent vs. continuous)
- Check if multiple sensors on the same gateway have gaps (gateway issue vs. sensor issue)

MAY:
- Review CloudTrail for gateway disconnect events
- Check if data gaps correlate with facility events (power outages, maintenance)

## Phase 2 — Remediate

MUST:
- If gateway offline: restore WiFi connectivity and verify solid green LED
- If sensor battery depleted: replace the sensor unit (batteries are not replaceable)
- If BLE range issue: move sensor closer to gateway or add additional gateway

SHOULD:
- Check for BLE interference from other industrial equipment
- Verify sensor mounting is secure (loose sensors may lose BLE connection)

MAY:
- Set up CloudWatch alarms for data gap detection
- Plan gateway placement to ensure overlapping BLE coverage

## Common Issues

- symptoms: "Single sensor has data gaps while others on same gateway are fine"
  diagnosis: "Sensor BLE range issue or sensor battery depleting."
  resolution: "Check sensor battery status in app. Move sensor closer to gateway. If battery low, replace sensor."

- symptoms: "All sensors on one gateway have simultaneous data gaps"
  diagnosis: "Gateway lost WiFi connectivity."
  resolution: "Check gateway LED. Verify WiFi network is operational. Power cycle gateway if LED shows error state."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metric retrieval"
  - "Sensor replacement: YELLOW — new sensor requires 14-day learning period"
  - "Gateway power cycle: YELLOW — temporarily disrupts all connected sensors"
  - "Gateway relocation: YELLOW — may affect all connected sensor ranges"
```

## Escalation Conditions

- Data gaps persist after verifying gateway connectivity and sensor battery
- Multiple gateways show simultaneous data gaps (network infrastructure issue)
- Sensor replacement does not resolve data gaps (possible environmental interference)
- Data gaps affect safety-critical equipment monitoring

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — returns aggregated metrics
- Sensor data readings: MEDIUM — reveals equipment operational patterns
- Gateway connectivity logs: LOW — operational metadata
- Facility layout information: MEDIUM — reveals equipment placement

## Prohibited Actions

- NEVER ignore data gaps on safety-critical equipment — escalate immediately
- NEVER assume data gaps are normal without investigation
- NEVER modify sensor measurement frequency without understanding battery impact
- NEVER relocate gateways without assessing impact on all connected sensors

## Phase 3 — Rollback

1. If gateway was relocated and caused more gaps, return to original position
2. If sensor was replaced unnecessarily, the old sensor cannot be restored (battery activated)
3. If measurement frequency was changed, restore original frequency setting
4. Document the data gap timeline for maintenance records
5. Verify data collection has resumed and is continuous after remediation

## Output Format

```yaml
root_cause: "sensor_data_gap — <specific_cause>"
evidence:
  - type: data_gap
    content: "<gap duration, affected sensors, CloudWatch metrics>"
severity: HIGH
mitigation:
  immediate: "Restore connectivity or replace sensor"
  long_term: "Implement data gap monitoring and alerting"
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
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
