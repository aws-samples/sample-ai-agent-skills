---
title: "B1 — Sensor Pairing Errors"
description: "Diagnose sensor pairing and registration failures"
status: active
severity: HIGH
triggers: ["sensor pairing failed", "cannot pair sensor", "sensor not found", "BLE error"]
owner: devops-agent
objective: "Resolve sensor pairing issues and establish data collection"
context: "Sensors pair with gateways via Bluetooth Low Energy (BLE). Pairing requires the Monitron mobile app, physical proximity, and a commissioned gateway."
---

## Phase 1 — Triage

MUST:
- Verify the gateway is commissioned and online (check LED status — solid green = connected)
- Confirm the user has the Monitron mobile app with project access
- Check physical proximity — sensor must be within 1-2 meters of gateway during pairing

SHOULD:
- Verify sensor is new or has been factory reset (previously paired sensors must be reset)
- Check if the gateway has reached its sensor limit
- Confirm Bluetooth is enabled on the mobile device

MAY:
- Check CloudTrail for sensor registration events
- Verify the sensor model is compatible with the gateway model

## Phase 2 — Remediate

MUST:
- Bring sensor within 1-2 meters of gateway for initial pairing
- Use the Monitron mobile app to initiate pairing (CLI cannot pair sensors)
- Ensure gateway LED shows solid green (connected) before pairing

SHOULD:
- Remove sensor from packaging just before pairing (battery activates on removal)
- Pair sensors one at a time to avoid BLE interference

MAY:
- Power cycle the gateway if BLE pairing consistently fails
- Try pairing from a different mobile device if app issues persist

## Common Issues

- symptoms: "App shows 'Sensor not found' during pairing"
  diagnosis: "Sensor too far from gateway, BLE interference, or sensor already paired."
  resolution: "Move sensor within 1-2 meters of gateway. Ensure no other BLE devices interfere. If previously paired, factory reset the sensor."

- symptoms: "Pairing starts but fails to complete"
  diagnosis: "Gateway lost internet connectivity during pairing process."
  resolution: "Verify gateway has stable internet connection (solid green LED). Retry pairing after confirming connectivity."

## Safety Ratings

```
safety_ratings:
  - "Sensor pairing via app: YELLOW — registers sensor to gateway, starts data collection"
  - "Sensor factory reset: YELLOW — clears pairing, requires re-pair and new learning period"
  - "Gateway power cycle: YELLOW — temporarily disrupts all connected sensors"
  - "monitron get-project: GREEN — read-only project details"
```

## Escalation Conditions

- Multiple sensors fail to pair with the same gateway (possible gateway BLE hardware issue)
- Sensor appears defective (no response during pairing attempt)
- Gateway has reached maximum sensor capacity
- Industrial environment causes persistent BLE interference

## Data Sensitivity

- Sensor serial numbers: LOW — device identifiers
- Gateway connection status: LOW — operational metadata
- Pairing logs in mobile app: LOW — device registration events
- `aws cloudtrail lookup-events`: MEDIUM — may reveal user identities

## Prohibited Actions

- NEVER attempt to open or modify sensor hardware
- NEVER pair sensors to a gateway in a different project without authorization
- NEVER force-pair by bypassing the mobile app workflow
- NEVER discard sensors that fail to pair without checking warranty status

## Phase 3 — Rollback

1. If sensor was paired to wrong gateway or asset, unpair through the mobile app
2. If sensor needs to be moved to a different gateway, unpair first then re-pair
3. After unpairing, the sensor restarts its 14-day learning period on re-pair
4. Document the pairing issue for the maintenance team
5. Track sensor serial numbers for warranty and replacement purposes

## Output Format

```yaml
root_cause: "sensor_pairing — <specific_cause>"
evidence:
  - type: pairing_error
    content: "<app error message, LED status, or BLE details>"
severity: HIGH
mitigation:
  immediate: "Fix proximity/connectivity and retry pairing"
  long_term: "Document sensor placement guidelines for the facility"
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
