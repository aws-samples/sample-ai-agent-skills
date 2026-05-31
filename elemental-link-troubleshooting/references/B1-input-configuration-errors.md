---
title: "B1 — Input Configuration Errors"
description: "Diagnose Elemental Link input configuration errors"
status: active
severity: MEDIUM
triggers: ["input config error", "cannot configure input", "input settings wrong", "resolution mismatch"]
owner: devops-agent
objective: "Resolve input configuration issues on Elemental Link devices"
context: "Elemental Link input settings control encoding resolution, bitrate, and codec parameters. Settings sync between cloud configuration and the physical device."
---

## Phase 1 — Triage

MUST:
- Check current device configuration: `aws medialive describe-input-device --input-device-id <device-id>`
- Verify settings sync state: `aws medialive describe-input-device --input-device-id <device-id> --query 'DeviceSettingsSyncState'`
- Check the configured resolution and codec settings match the source video

SHOULD:
- Verify the input source is providing a supported resolution and frame rate
- Check if the device model (HD vs UHD) supports the desired configuration

MAY:
- Review CloudTrail for recent UpdateInputDevice calls: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateInputDevice --max-results 10`

## Phase 2 — Remediate

MUST:
- Update device settings to match source video: `aws medialive update-input-device --input-device-id <device-id> --hd-device-settings '{"MaxBitrate":<bitrate>,"ConfiguredInput":"AUTO"}'`
- Wait for DeviceSettingsSyncState to show SYNCED after changes
- Verify the source video format is within device capabilities

SHOULD:
- Use AUTO for ConfiguredInput unless a specific input is required
- Set MaxBitrate appropriate for the network upload bandwidth

MAY:
- Test with a known-good video source to isolate configuration vs source issues

## Safety Ratings

```
safety_ratings:
  - "describe-input-device: GREEN — read-only device status check"
  - "update-input-device: YELLOW — modifies device encoding settings, may affect active stream"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Device settings fail to sync after multiple update attempts
- Device reports unsupported input format for a format that should be supported
- Configuration changes cause device to disconnect
- Settings sync state stuck in SYNCING for more than 10 minutes

## Data Sensitivity

- `aws medialive describe-input-device`: LOW — returns device configuration metadata
- `aws medialive update-input-device`: MEDIUM — modifies device encoding parameters
- `aws cloudtrail lookup-events`: MEDIUM — reveals who changed device settings and when

## Prohibited Actions

- NEVER change device settings during a live production stream without coordination
- NEVER set bitrate higher than the available upload bandwidth
- NEVER suggest manual firmware changes to fix configuration issues
- NEVER assume UHD capabilities on an HD model device

## Phase 3 — Rollback

1. Revert device settings to previous configuration: `aws medialive update-input-device --input-device-id <device-id> --hd-device-settings '{"MaxBitrate":<previous-bitrate>,"ConfiguredInput":"<previous-input>"}'`
2. Wait for DeviceSettingsSyncState to return to SYNCED
3. Verify the active stream (if any) is functioning correctly after rollback
4. Document the configuration change attempt and reason for rollback
5. Notify the production team if stream quality was affected

## Common Issues

- symptoms: "Settings show SYNCING but never reach SYNCED"
  diagnosis: "Device connectivity is intermittent, preventing settings delivery."
  resolution: "Resolve connectivity issues first (see A2), then retry configuration."

## Output Format

```yaml
root_cause: "input_configuration — <specific_cause>"
evidence:
  - type: config_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Correct input configuration"
  long_term: "Document standard configurations for each use case"
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
