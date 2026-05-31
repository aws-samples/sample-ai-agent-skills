---
title: "A1 — Device Registration Failures"
description: "Diagnose Elemental Link device registration failures"
status: active
severity: HIGH
triggers: ["device not appearing", "registration failed", "device not found in console"]
owner: devops-agent
objective: "Resolve Elemental Link device registration issues so the device appears in the AWS console"
context: "Elemental Link devices register automatically when connected to the internet. Registration associates the device with the purchasing AWS account and region."
---

## Phase 1 — Triage

MUST:
- Check if device appears: `aws medialive list-input-devices`
- Verify the device has power (check LED indicators on the physical device)
- Verify the device has internet connectivity (check network cable and link lights)

SHOULD:
- Confirm the AWS account matches the account used to purchase the device
- Verify you are checking the correct AWS region
- Check CloudTrail for device-related events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=medialive.amazonaws.com --max-results 10`

MAY:
- Check for pending device transfers: `aws medialive list-input-device-transfers --transfer-type INCOMING`
- Verify DNS resolution from the device network to AWS endpoints

## Phase 2 — Remediate

MUST:
- Ensure the device has outbound HTTPS (port 443) access to AWS endpoints
- Verify the device is connected to a network with DHCP or properly configured static IP
- Power cycle the device if it has been connected for more than 30 minutes without registering

SHOULD:
- Check firewall rules for AWS IP range access (refer to AWS IP ranges JSON)
- Verify no proxy server is intercepting HTTPS traffic from the device

MAY:
- Contact AWS Support with the device serial number if registration fails after network verification
- Request a device transfer if the device was purchased under a different account

## Safety Ratings

```
safety_ratings:
  - "list-input-devices: GREEN — read-only enumeration of registered devices"
  - "list-input-device-transfers: GREEN — read-only check for pending transfers"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "power cycle device: YELLOW — interrupts any active stream from the device"
```

## Escalation Conditions

- Device does not appear after 1 hour with confirmed internet connectivity
- Device serial number does not match any AWS account records
- Multiple devices from the same shipment fail to register
- Network team confirms all firewall rules are correct but device still fails

## Data Sensitivity

- `aws medialive list-input-devices`: LOW — returns device metadata and connection state
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- Device serial number: MEDIUM — uniquely identifies physical hardware asset

## Prohibited Actions

- NEVER suggest a manual registration API — it does not exist
- NEVER suggest flashing firmware to fix registration — firmware is AWS-managed
- NEVER suggest opening SSH or console access to the device
- NEVER transfer a device to another account without explicit owner approval

## Phase 3 — Rollback

1. If a device transfer was initiated in error, cancel it: `aws medialive cancel-input-device-transfer --input-device-id <device-id>`
2. If network changes were made to accommodate the device, revert firewall rules if registration is not proceeding
3. If the device was power cycled during an active stream, coordinate with production to restart the MediaLive channel
4. Document the device serial number and registration attempt timeline for AWS Support
5. Verify no other devices were affected by network changes

## Common Issues

- symptoms: "Device has power and network but does not appear in console"
  diagnosis: "Firewall blocking outbound HTTPS to AWS MediaLive endpoints."
  resolution: "Allow outbound 443 to AWS IP ranges for the medialive service in the target region."

- symptoms: "Device appears in a different region than expected"
  diagnosis: "Device registered to the region associated with the purchase order."
  resolution: "Transfer the device to the desired region: aws medialive transfer-input-device."

## Output Format

```yaml
root_cause: "device_registration — <specific_cause>"
evidence:
  - type: registration_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Resolve registration blocker"
  long_term: "Document network requirements for future device deployments"
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
