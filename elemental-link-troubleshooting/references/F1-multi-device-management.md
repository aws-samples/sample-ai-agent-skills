---
title: "F1 — Multi-Device Management"
description: "Diagnose Elemental Link fleet and multi-device management issues"
status: active
severity: MEDIUM
triggers: ["manage multiple devices", "fleet management", "device transfer", "bulk operations", "device inventory"]
owner: devops-agent
objective: "Resolve multi-device management, transfer, and fleet operations issues"
context: "Organizations with multiple Elemental Link devices need fleet management for transfers, monitoring, and configuration. Devices can be transferred between accounts and regions."
---

## Phase 1 — Triage

MUST:
- List all devices and their status: `aws medialive list-input-devices --query 'InputDevices[*].{Id:Id,Name:Name,Type:Type,ConnectionState:ConnectionState,DeviceSettingsSyncState:DeviceSettingsSyncState}'`
- Check for pending transfers: `aws medialive list-input-device-transfers --transfer-type INCOMING`
- Check outgoing transfers: `aws medialive list-input-device-transfers --transfer-type OUTGOING`

SHOULD:
- Verify device distribution across regions
- Check for devices with outdated firmware: `aws medialive list-input-devices --query 'InputDevices[?DeviceUpdateStatus!=`NOT_UPDATE_AVAILABLE`]'`

MAY:
- Audit device naming conventions for consistency
- Review device utilization (connected vs disconnected devices)

## Phase 2 — Remediate

MUST:
- Accept incoming device transfers: `aws medialive accept-input-device-transfer --input-device-transfer-message-id <transfer-id>`
- Initiate device transfers to target accounts: `aws medialive transfer-input-device --input-device-id <device-id> --target-customer-id <account-id>`
- Ensure transferred devices are reconfigured for the new account's MediaLive channels

SHOULD:
- Implement consistent naming conventions across the fleet
- Maintain an inventory of device serial numbers, locations, and account assignments
- Stage firmware updates across the fleet (test devices first)

MAY:
- Automate fleet monitoring with CloudWatch dashboards
- Use AWS Organizations for centralized device management across accounts
- Implement tagging for device categorization

## Safety Ratings

```
safety_ratings:
  - "list-input-devices: GREEN — read-only fleet enumeration"
  - "list-input-device-transfers: GREEN — read-only transfer status"
  - "accept-input-device-transfer: YELLOW — accepts device ownership"
  - "transfer-input-device: RED — transfers device ownership to another account"
  - "reject-input-device-transfer: YELLOW — rejects incoming transfer"
```

## Escalation Conditions

- Device transfer stuck in pending state for more than 24 hours
- Transferred device does not appear in the target account
- Need to recover a device transferred to the wrong account
- Fleet-wide firmware issue affecting multiple devices simultaneously

## Data Sensitivity

- `aws medialive list-input-devices`: LOW — returns device metadata
- `aws medialive list-input-device-transfers`: MEDIUM — reveals cross-account relationships
- `aws medialive transfer-input-device`: HIGH — changes device ownership between accounts
- Target account ID: MEDIUM — reveals AWS account identifiers

## Prohibited Actions

- NEVER transfer a device that is actively streaming without stopping the channel first
- NEVER transfer devices to unknown or unverified account IDs
- NEVER accept transfers from unknown sources without verification
- NEVER bulk-transfer production devices without a rollback plan

## Phase 3 — Rollback

1. Cancel pending outgoing transfers: `aws medialive cancel-input-device-transfer --input-device-id <device-id>`
2. If a device was transferred to the wrong account, request the recipient to transfer it back
3. Reject unwanted incoming transfers: `aws medialive reject-input-device-transfer --input-device-transfer-message-id <transfer-id>`
4. Restore device configurations after transfer (settings are reset on transfer)
5. Update inventory documentation to reflect current device assignments

## Common Issues

- symptoms: "Transferred device shows no inputs in the new account"
  diagnosis: "Device transfer resets input associations — new inputs must be created in the target account."
  resolution: "Create new INPUT_DEVICE type inputs in the target account for the transferred device."

## Output Format

```yaml
root_cause: "multi_device_management — <specific_cause>"
evidence:
  - type: fleet_issue
    content: "<details>"
severity: MEDIUM
mitigation:
  immediate: "Resolve fleet management issue"
  long_term: "Implement fleet management procedures and inventory tracking"
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
