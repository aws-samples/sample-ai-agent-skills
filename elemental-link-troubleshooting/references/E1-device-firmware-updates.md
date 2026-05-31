---
title: "E1 — Device Firmware Updates"
description: "Diagnose Elemental Link firmware update issues"
status: active
severity: MEDIUM
triggers: ["firmware update failed", "firmware pending", "device outdated", "update stuck"]
owner: devops-agent
objective: "Resolve firmware update issues on Elemental Link devices"
context: "Firmware updates are managed by AWS and pushed to devices automatically. Operators can accept or defer updates. Updates require device connectivity and may cause brief interruptions."
---

## Phase 1 — Triage

MUST:
- Check device firmware and update status: `aws medialive describe-input-device --input-device-id <device-id> --query '{DeviceUpdateStatus:DeviceUpdateStatus,MacAddress:MacAddress,SerialNumber:SerialNumber}'`
- Verify device is CONNECTED: `aws medialive describe-input-device --input-device-id <device-id> --query 'ConnectionState'`
- Check if there are pending updates across the fleet: `aws medialive list-input-devices --query 'InputDevices[?DeviceUpdateStatus!=`NOT_UPDATE_AVAILABLE`].{Id:Id,Name:Name,UpdateStatus:DeviceUpdateStatus}'`

SHOULD:
- Verify the device has been connected long enough for update delivery
- Check if the device is actively streaming (updates may be deferred during active streams)

MAY:
- Review CloudTrail for UpdateInputDevice events related to firmware acceptance
- Check AWS service announcements for known firmware issues

## Phase 2 — Remediate

MUST:
- Accept pending firmware update: `aws medialive update-input-device --input-device-id <device-id> --device-update-status ACCEPT`
- Ensure the device remains connected during the update process
- Schedule updates during maintenance windows to avoid stream interruption

SHOULD:
- Update devices in a staged rollout (test devices first, then production)
- Monitor device status during and after the update

MAY:
- Defer updates for production devices until tested: `aws medialive update-input-device --input-device-id <device-id> --device-update-status DEFER`
- Set up notifications for new firmware availability

## Safety Ratings

```
safety_ratings:
  - "describe-input-device: GREEN — read-only device status check"
  - "list-input-devices: GREEN — read-only enumeration"
  - "update-input-device ACCEPT: YELLOW — triggers firmware update, may cause brief interruption"
  - "update-input-device DEFER: GREEN — postpones update, no immediate impact"
```

## Escalation Conditions

- Firmware update fails repeatedly on the same device
- Device becomes unresponsive after a firmware update
- Update status stuck in a transitional state for more than 1 hour
- Critical security firmware update cannot be applied due to production constraints

## Data Sensitivity

- `aws medialive describe-input-device`: LOW — returns device metadata and firmware status
- `aws medialive list-input-devices`: LOW — returns fleet-wide device information
- Device serial number and MAC address: MEDIUM — uniquely identifies physical hardware

## Prohibited Actions

- NEVER suggest downloading or manually installing firmware — it is AWS-managed
- NEVER accept firmware updates on production devices during live streams without approval
- NEVER suggest opening the device to access internal storage or firmware chips
- NEVER defer critical security updates indefinitely

## Phase 3 — Rollback

1. Firmware updates cannot be rolled back by the operator — contact AWS Support if a firmware update causes issues
2. If a device becomes unresponsive after update, power cycle the device
3. If the update was accepted in error during production, the update may not apply until the next maintenance window
4. Document the firmware version before and after the update for AWS Support reference
5. If multiple devices are affected, halt the rollout and contact AWS Support

## Common Issues

- symptoms: "DeviceUpdateStatus shows UPDATE_AVAILABLE but update never applies"
  diagnosis: "Update has not been accepted — it requires explicit acceptance."
  resolution: "Accept the update: aws medialive update-input-device --input-device-id <id> --device-update-status ACCEPT."

## Output Format

```yaml
root_cause: "firmware_update — <specific_cause>"
evidence:
  - type: firmware_status
    content: "<status details>"
severity: MEDIUM
mitigation:
  immediate: "Resolve firmware update issue"
  long_term: "Implement staged firmware update process with testing"
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
