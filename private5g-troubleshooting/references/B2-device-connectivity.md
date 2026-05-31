---
title: "B2 — Device Connectivity"
description: "Diagnose device connectivity failures on Private 5G"
status: active
severity: HIGH
triggers: ["device cannot connect", "no signal", "device offline", "authentication failed"]
owner: devops-agent
objective: "Resolve device connectivity issues on the Private 5G network"
context: "Device connectivity requires an active radio unit, a provisioned SIM card, and the device within coverage range."
---

## Phase 1 — Triage

MUST:
- List device identifiers: `aws private-networks list-device-identifiers --network-arn <arn>`
- Check SIM status for the device: `aws private-networks get-device-identifier --device-identifier-arn <arn>`
- Verify radio unit is active: `aws private-networks list-network-resources --network-arn <arn> --filters name=type,values=RADIO_UNIT`

SHOULD:
- Verify device is within radio unit coverage range
- Check if SIM card is properly inserted in the device
- Review CloudWatch connectivity metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Private5G --metric-name ConnectedDevices --dimensions Name=NetworkArn,Value=<arn> --start-time <start> --end-time <end> --period 300 --statistics Sum`

MAY:
- Test with a different device using the same SIM
- Check for RF interference in the area

## Phase 2 — Remediate

MUST:
- Ensure SIM is in ACTIVE status
- Confirm radio unit is ACTIVE and broadcasting
- Verify device supports CBRS Band 48 (3.5 GHz)

SHOULD:
- Move device closer to radio unit to rule out coverage issues
- Restart the device to force network re-registration
- Check device APN settings match the Private 5G network configuration

MAY:
- Deactivate and reactivate the SIM if stuck in an error state
- Test connectivity with a known-good device

## Safety Ratings

```
safety_ratings:
  - "list-device-identifiers: GREEN — read-only enumeration"
  - "get-device-identifier: GREEN — read-only device details"
  - "list-network-resources: GREEN — read-only resource listing"
  - "deactivate-device-identifier: YELLOW — disconnects device from network"
```

## Escalation Conditions

- Multiple devices cannot connect despite active radio units and SIMs
- Device authentication failures with no clear SIM or configuration issue
- Connectivity drops intermittently across all devices at a site
- Device connects but cannot reach VPC resources

## Data Sensitivity

- `aws private-networks list-device-identifiers`: MEDIUM — reveals IMSI and device identity information
- `aws private-networks get-device-identifier`: MEDIUM — reveals SIM and subscriber details
- Device connectivity logs: MEDIUM — may contain user activity patterns

## Prohibited Actions

- NEVER deactivate a SIM without confirming with the device owner
- NEVER share IMSI or ICCID values in logs or outputs
- NEVER assume device hardware failure without testing with a known-good SIM
- NEVER modify network configuration to troubleshoot a single device issue

## Phase 3 — Rollback

1. If a SIM was deactivated during troubleshooting, reactivate it: `aws private-networks activate-device-identifier --device-identifier-arn <arn>`
2. If device APN settings were changed, restore original settings
3. Verify the device reconnects successfully after changes
4. Confirm other devices on the network were not affected
5. Document the connectivity issue and resolution

## Common Issues

- symptoms: "Device shows 'No Service' despite being near a radio unit"
  diagnosis: "Device may not support CBRS Band 48 or SIM is not activated."
  resolution: "Verify device CBRS compatibility. Check SIM status is ACTIVE."

- symptoms: "Device connects but has no data connectivity"
  diagnosis: "VPC routing or security group may be blocking traffic."
  resolution: "Check VPC subnet routing table and security group rules for the Private 5G ENI."

## Output Format

```yaml
root_cause: "device_connectivity — <specific_cause>"
evidence:
  - type: connectivity_status
    content: "<status details>"
severity: HIGH
mitigation:
  immediate: "Restore device connectivity"
  long_term: "Document device compatibility requirements"
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
